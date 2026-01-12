# Areas Where OpenCost Should Fail Fast

## Executive Summary

This document identifies critical areas in the OpenCost codebase where errors are currently logged but should instead cause the application to fail fast with clear error messages. Silent failures in these areas prevent valid cost data collection and make troubleshooting difficult.

**Key Principle**: If an error prevents OpenCost from collecting accurate cost data, we should fail fast at startup rather than run in a degraded state that produces incomplete or invalid results.

---

## 1. Critical Infrastructure Failures

### 1.1 Kubernetes Client Initialization

**Current State**: ✅ **GOOD** - Already fails fast
- **Location**: [`pkg/costmodel/router.go:404-407`](pkg/costmodel/router.go:404-407)
- **Behavior**: `log.Fatalf` on failure to build Kubernetes client
- **Impact**: Cannot access any Kubernetes resources
- **Verdict**: Correct - this is a critical failure

```go
kubeClientset, err := kubeconfig.LoadKubeClient("")
if err != nil {
    log.Fatalf("Failed to build Kubernetes client: %s", err.Error())
}
```

### 1.2 Cluster UID Determination

**Current State**: ✅ **GOOD** - Already fails fast
- **Location**: [`pkg/costmodel/router.go:409-412`](pkg/costmodel/router.go:409-412)
- **Behavior**: `log.Fatalf` on failure to get cluster UID
- **Impact**: Cannot identify cluster for cost attribution
- **Verdict**: Correct - cluster UID is fundamental

### 1.3 Prometheus Data Source Creation

**Current State**: ✅ **GOOD** - Already fails fast
- **Location**: [`pkg/costmodel/router.go:484-487`](pkg/costmodel/router.go:484-487), [`pkg/cmd/agent/agent.go:127-130`](pkg/cmd/agent/agent.go:127-130)
- **Behavior**: `log.Fatalf` + `panic` on fatal Prometheus connection errors
- **Impact**: Cannot query metrics for cost calculation
- **Verdict**: Correct - metrics are essential

### 1.4 KubeModel Initialization

**Current State**: ✅ **GOOD** - Already fails fast
- **Location**: [`pkg/costmodel/costmodel.go:70-74`](pkg/costmodel/costmodel.go:70-74)
- **Behavior**: `log.Fatalf` on KubeModel initialization failure
- **Impact**: Cannot build cost model
- **Verdict**: Correct - KubeModel is required

---

## 2. Cloud Provider Configuration

### 2.1 Cloud Provider Initialization

**Current State**: ✅ **GOOD** - Already fails fast
- **Location**: [`pkg/costmodel/router.go:422-425`](pkg/costmodel/router.go:422-425), [`pkg/cmd/agent/agent.go:87-90`](pkg/cmd/agent/agent.go:87-90)
- **Behavior**: `panic` on provider creation failure
- **Impact**: Cannot get cloud pricing data
- **Verdict**: Correct - cloud provider is essential for accurate costs

### 2.2 Pricing Data Download Failures

**Current State**: ⚠️ **SHOULD FAIL FAST** (with caveats)
- **Locations**:
  - [`pkg/cmd/agent/agent.go:154-157`](pkg/cmd/agent/agent.go:154-157) - Logs error, continues
  - [`pkg/costmodel/router.go:519-521`](pkg/costmodel/router.go:519-521) - Logs info, continues
  - [`pkg/cloud/aws/provider.go:862-864`](pkg/cloud/aws/provider.go:862-864) - Logs error, continues
  - [`pkg/cloud/gcp/provider.go:1039-1041`](pkg/cloud/gcp/provider.go:1039-1041) - Logs error, continues
  - [`pkg/cloud/azure/provider.go:921-923`](pkg/cloud/azure/provider.go:921-923) - Logs error, continues

**Problem**: Application starts successfully but uses stale/default pricing, leading to inaccurate costs.

**Impact**: 
- **High** - Incorrect cost calculations
- Users don't realize pricing data is outdated
- Costs may be significantly wrong

**Recommendation**: 
```go
// Option 1: Fail fast on first pricing download
err = cloudProvider.DownloadPricingData()
if err != nil {
    log.Fatalf("FATAL: Failed to download pricing data on startup: %s\n"+
        "OpenCost cannot provide accurate costs without current pricing data.\n"+
        "Please check cloud provider credentials and network connectivity.", err)
}

// Option 2: Allow startup with warning if cached pricing exists
err = cloudProvider.DownloadPricingData()
if err != nil {
    if !cloudProvider.HasCachedPricing() {
        log.Fatalf("FATAL: Failed to download pricing data and no cached pricing available: %s", err)
    }
    log.Warnf("WARNING: Failed to download current pricing data, using cached pricing: %s", err)
}
```

---

## 3. Data Source & Storage Failures

### 3.1 Prometheus Query Failures During Initialization

**Current State**: ⚠️ **MIXED** - Some queries fail silently
- **Location**: [`pkg/costmodel/costmodel.go:192-194`](pkg/costmodel/costmodel.go:192-194)
- **Behavior**: Logs warning, continues with partial data
- **Impact**: Missing critical metrics for cost calculation

**Problem**: If Prometheus is unreachable or queries fail during startup, we continue with no data.

**Recommendation**:
```go
// During initialization, validate Prometheus connectivity
func validatePrometheusConnectivity(dataSource source.OpenCostDataSource) error {
    // Try a simple query to verify Prometheus is accessible
    ctx, cancel := context.WithTimeout(context.Background(), 30*time.Second)
    defer cancel()
    
    _, err := dataSource.Query(ctx, "up", time.Now(), time.Now())
    if err != nil {
        return fmt.Errorf("Prometheus connectivity check failed: %w", err)
    }
    return nil
}

// In initialization code:
if err := validatePrometheusConnectivity(dataSource); err != nil {
    log.Fatalf("FATAL: Cannot connect to Prometheus: %s\n"+
        "OpenCost requires Prometheus for metrics collection.\n"+
        "Please verify Prometheus is running and accessible.", err)
}
```

### 3.2 Storage Backend Failures

**Current State**: ⚠️ **SHOULD FAIL FAST**
- **Locations**:
  - [`pkg/config/configfile.go:267-269`](pkg/config/configfile.go:267-269) - Logs error, continues
  - [`pkg/config/configfile.go:297-299`](pkg/config/configfile.go:297-299) - Logs warning, continues

**Problem**: If storage backend (S3, Azure Blob, etc.) is misconfigured, writes fail silently.

**Impact**:
- **High** - Data loss
- Cost data not persisted
- No historical data available

**Recommendation**:
```go
// Validate storage backend on startup
func validateStorageBackend(storage Storage) error {
    // Try a write/read/delete test
    testKey := fmt.Sprintf("opencost-health-check-%d", time.Now().Unix())
    testData := []byte("health-check")
    
    if err := storage.Write(testKey, testData); err != nil {
        return fmt.Errorf("storage write test failed: %w", err)
    }
    
    if _, err := storage.Read(testKey); err != nil {
        return fmt.Errorf("storage read test failed: %w", err)
    }
    
    storage.Delete(testKey) // Best effort cleanup
    return nil
}

// In initialization:
if err := validateStorageBackend(storageBackend); err != nil {
    log.Fatalf("FATAL: Storage backend validation failed: %s\n"+
        "OpenCost cannot persist cost data.\n"+
        "Please verify storage configuration and credentials.", err)
}
```

---

## 4. Configuration & Secrets

### 4.1 Cloud Provider Credentials

**Current State**: ⚠️ **SHOULD FAIL FAST**
- **Locations**:
  - [`pkg/cloud/gcp/provider.go:197-211`](pkg/cloud/gcp/provider.go:197-211) - Logs warning, continues
  - [`pkg/cloud/config/watcher.go:38-47`](pkg/cloud/config/watcher.go:38-47) - Logs error, returns nil

**Problem**: Missing or invalid cloud credentials cause pricing lookups to fail, but application continues.

**Impact**:
- **High** - All cloud costs will be wrong
- Falls back to default/estimated pricing
- No indication to user that credentials are missing

**Recommendation**:
```go
// Validate cloud credentials on startup
func validateCloudCredentials(provider Provider) error {
    // Attempt a simple API call to verify credentials
    if err := provider.TestCredentials(); err != nil {
        return fmt.Errorf("cloud provider credential validation failed: %w", err)
    }
    return nil
}

// In initialization:
if err := validateCloudCredentials(cloudProvider); err != nil {
    if env.IsCloudProviderRequired() {
        log.Fatalf("FATAL: Cloud provider credentials invalid: %s\n"+
            "OpenCost requires valid cloud credentials for accurate pricing.\n"+
            "Please verify credentials are correctly configured.", err)
    }
    log.Warnf("WARNING: Cloud provider credentials invalid, using default pricing: %s", err)
}
```

### 4.2 Custom Pricing Configuration

**Current State**: ⚠️ **SHOULD VALIDATE**
- **Location**: [`pkg/cloud/provider/providerconfig.go:88-90`](pkg/cloud/provider/providerconfig.go:88-90)
- **Behavior**: Logs info, returns default pricing
- **Impact**: User thinks custom pricing is applied but it's not

**Recommendation**:
```go
// If custom pricing file is specified but fails to load, fail fast
if customPricingPath := env.GetCustomPricingPath(); customPricingPath != "" {
    pricing, err := loadCustomPricing(customPricingPath)
    if err != nil {
        log.Fatalf("FATAL: Custom pricing file specified but failed to load: %s\n"+
            "Path: %s\n"+
            "Please verify the file exists and is valid JSON.", err, customPricingPath)
    }
}
```

---

## 5. Resource Watcher Failures

### 5.1 Cluster Cache Initialization (CRITICAL)

**Current State**: ⚠️ **VULNERABLE** - No validation
- **Location**: [`pkg/clustercache/clustercache.go:52-108`](pkg/clustercache/clustercache.go:52-108)
- **Behavior**: Creates watchers, waits for sync, but doesn't validate RBAC permissions
- **Impact**: **CRITICAL** - No cost data if watchers can't access resources

**Problem**: Same as the ConfigMap issue - watchers fail silently if RBAC permissions are missing.

**Recommendation**: See [`docs/RBAC_VALIDATION_ANALYSIS.md`](docs/RBAC_VALIDATION_ANALYSIS.md) for detailed solution.

```go
// Add RBAC validation before creating watchers
func NewKubernetesClusterCacheV1(client kubernetes.Interface) cc.ClusterCache {
    // Validate RBAC permissions first
    if err := validateClusterCachePermissions(client); err != nil {
        panic(fmt.Sprintf("FATAL: Missing required RBAC permissions for cluster cache: %s", err))
    }
    
    // Continue with existing initialization...
}
```

### 5.2 ConfigMap Watcher Failures

**Current State**: ✅ **FIXED** - Now validates and fails fast
- **Location**: [`pkg/util/watcher/configwatchers.go:36-52`](pkg/util/watcher/configwatchers.go:36-52)
- **Behavior**: Validates RBAC permissions, panics with clear error if missing
- **Verdict**: Correct implementation

---

## 6. Data Quality Issues

### 6.1 Node Pricing Lookup Failures

**Current State**: ⚠️ **SHOULD WARN LOUDLY**
- **Locations**:
  - [`pkg/costmodel/costmodel.go:199-201`](pkg/costmodel/costmodel.go:199-201) - Logs warning, returns error
  - [`pkg/costmodel/cluster_helpers.go:46-48`](pkg/costmodel/cluster_helpers.go:46-48) - Logs warning, continues

**Problem**: If node pricing can't be determined, costs will be wrong but application continues.

**Impact**:
- **High** - Incorrect costs for all workloads on that node
- Silent data quality issue

**Recommendation**:
```go
// Track nodes with missing pricing
type NodePricingStatus struct {
    NodesWithPricing    int
    NodesWithoutPricing int
    MissingNodes        []string
}

// Expose metric and log prominently
if status.NodesWithoutPricing > 0 {
    log.Errorf("CRITICAL: %d/%d nodes have no pricing data: %v\n"+
        "Cost calculations will be inaccurate.\n"+
        "Please verify cloud provider configuration and node labels.",
        status.NodesWithoutPricing,
        status.NodesWithPricing+status.NodesWithoutPricing,
        status.MissingNodes)
    
    // Expose metric
    metrics.NodesMissingPricing.Set(float64(status.NodesWithoutPricing))
}
```

### 6.2 PV Pricing Lookup Failures

**Current State**: ⚠️ **SHOULD WARN LOUDLY**
- **Location**: [`pkg/costmodel/costmodel.go:207-209`](pkg/costmodel/costmodel.go:207-209)
- **Behavior**: Logs warning, continues with empty map
- **Impact**: Storage costs will be missing or wrong

**Recommendation**: Similar to node pricing - track and expose metrics for missing PV pricing.

---

## 7. Integration Failures

### 7.1 Cloud Cost Integration Failures

**Current State**: ⚠️ **SHOULD FAIL FAST** (if configured)
- **Locations**:
  - [`pkg/cloudcost/ingestionmanager.go:47-49`](pkg/cloudcost/ingestionmanager.go:47-49) - Logs error, continues
  - [`pkg/cloudcost/ingestionmanager.go:70-72`](pkg/cloudcost/ingestionmanager.go:70-72) - Logs error, continues

**Problem**: If cloud cost integration (Athena, BigQuery, etc.) is configured but fails, no cloud costs are ingested.

**Recommendation**:
```go
// If cloud cost integration is explicitly configured, validate it works
if cloudCostConfig := env.GetCloudCostIntegrationConfig(); cloudCostConfig != nil {
    if err := validateCloudCostIntegration(cloudCostConfig); err != nil {
        log.Fatalf("FATAL: Cloud cost integration configured but validation failed: %s\n"+
            "Please verify integration configuration and credentials.", err)
    }
}
```

### 7.2 Custom Cost Plugin Failures

**Current State**: ⚠️ **SHOULD VALIDATE**
- **Locations**:
  - [`pkg/customcost/pipelineservice.go:38-40`](pkg/customcost/pipelineservice.go:38-40) - Logs error, returns error
  - [`pkg/customcost/pipelineservice.go:72-74`](pkg/customcost/pipelineservice.go:72-74) - Logs error, returns error

**Problem**: If custom cost plugins are configured but fail to load, custom costs are missing.

**Recommendation**:
```go
// If custom cost plugins are configured, validate they load successfully
if pluginDir := env.GetCustomCostPluginDir(); pluginDir != "" {
    plugins, err := loadCustomCostPlugins(pluginDir)
    if err != nil {
        log.Fatalf("FATAL: Custom cost plugin directory specified but failed to load plugins: %s\n"+
            "Directory: %s\n"+
            "Please verify plugins are correctly installed.", err, pluginDir)
    }
    if len(plugins) == 0 {
        log.Warnf("WARNING: Custom cost plugin directory specified but no plugins found: %s", pluginDir)
    }
}
```

---

## 8. Certificate & TLS Failures

### 8.1 Node Client Certificate Loading

**Current State**: ✅ **GOOD** - Already fails fast
- **Location**: [`pkg/costmodel/nodeclientconfig.go:45-59`](pkg/costmodel/nodeclientconfig.go:45-59)
- **Behavior**: `log.Fatalf` on certificate loading failure
- **Verdict**: Correct - TLS is critical for secure communication

---

## 9. Startup Health Check Recommendations

### Comprehensive Startup Validation

Create a startup health check that validates all critical components:

```go
// pkg/health/startup.go
package health

type StartupValidator struct {
    k8sClient     kubernetes.Interface
    cloudProvider Provider
    dataSource    source.OpenCostDataSource
    storage       Storage
}

type ValidationResult struct {
    Component string
    Status    string // "OK", "WARNING", "CRITICAL"
    Message   string
    Error     error
}

func (v *StartupValidator) ValidateAll() []ValidationResult {
    results := []ValidationResult{}
    
    // 1. Kubernetes connectivity
    results = append(results, v.validateKubernetes())
    
    // 2. RBAC permissions
    results = append(results, v.validateRBACPermissions())
    
    // 3. Prometheus connectivity
    results = append(results, v.validatePrometheus())
    
    // 4. Cloud provider credentials
    results = append(results, v.validateCloudProvider())
    
    // 5. Pricing data availability
    results = append(results, v.validatePricingData())
    
    // 6. Storage backend
    results = append(results, v.validateStorage())
    
    return results
}

func (v *StartupValidator) FailOnCritical(results []ValidationResult) {
    criticalFailures := []ValidationResult{}
    warnings := []ValidationResult{}
    
    for _, result := range results {
        switch result.Status {
        case "CRITICAL":
            criticalFailures = append(criticalFailures, result)
        case "WARNING":
            warnings = append(warnings, result)
        }
    }
    
    // Log all warnings
    for _, warning := range warnings {
        log.Warnf("STARTUP WARNING [%s]: %s", warning.Component, warning.Message)
    }
    
    // Fail fast on any critical failures
    if len(criticalFailures) > 0 {
        log.Errorf("╔════════════════════════════════════════════════════════════════╗")
        log.Errorf("║           OPENCOST STARTUP VALIDATION FAILED                   ║")
        log.Errorf("╚════════════════════════════════════════════════════════════════╝")
        log.Errorf("")
        log.Errorf("The following critical issues prevent OpenCost from starting:")
        log.Errorf("")
        
        for i, failure := range criticalFailures {
            log.Errorf("%d. [%s] %s", i+1, failure.Component, failure.Message)
            if failure.Error != nil {
                log.Errorf("   Error: %v", failure.Error)
            }
            log.Errorf("")
        }
        
        log.Fatalf("OpenCost cannot start with %d critical failure(s). Please resolve the issues above.", len(criticalFailures))
    }
    
    log.Infof("✓ Startup validation passed with %d warning(s)", len(warnings))
}
```

---

## 10. Summary & Priority Matrix

### Critical (Must Fail Fast)
1. ✅ **Kubernetes client initialization** - Already handled
2. ✅ **Cluster UID determination** - Already handled
3. ✅ **Prometheus data source** - Already handled
4. ✅ **KubeModel initialization** - Already handled
5. ⚠️ **Cluster cache RBAC permissions** - Needs implementation
6. ⚠️ **Initial pricing data download** - Should fail if no cached data
7. ⚠️ **Storage backend validation** - Should validate on startup

### High Priority (Should Fail Fast or Warn Loudly)
8. ⚠️ **Cloud provider credentials** - Validate if required
9. ⚠️ **Custom pricing configuration** - Fail if specified but invalid
10. ⚠️ **Node pricing availability** - Track and expose metrics
11. ⚠️ **Cloud cost integration** - Validate if configured

### Medium Priority (Should Warn)
12. ⚠️ **PV pricing availability** - Track and expose metrics
13. ⚠️ **Custom cost plugins** - Validate if configured
14. ⚠️ **Prometheus query failures** - Distinguish transient vs permanent

---

## Implementation Roadmap

### Phase 1: Critical Failures (Week 1-2)
- [ ] Implement cluster cache RBAC validation
- [ ] Add pricing data validation on startup
- [ ] Create storage backend health check
- [ ] Implement comprehensive startup validator

### Phase 2: High Priority (Week 3-4)
- [ ] Add cloud provider credential validation
- [ ] Validate custom pricing configuration
- [ ] Implement node pricing tracking
- [ ] Add cloud cost integration validation

### Phase 3: Observability (Week 5-6)
- [ ] Add metrics for all validation checks
- [ ] Create startup health dashboard
- [ ] Implement alerting for degraded states
- [ ] Add troubleshooting documentation

---

## Testing Strategy

### Unit Tests
```go
func TestStartupValidator_FailsOnMissingRBAC(t *testing.T) {
    // Mock k8s client that returns Forbidden
    // Verify startup fails with clear error
}

func TestStartupValidator_FailsOnInvalidPricing(t *testing.T) {
    // Mock pricing download failure
    // Verify startup fails if no cached pricing
}
```

### Integration Tests
```go
func TestStartup_WithInvalidCredentials(t *testing.T) {
    // Deploy with invalid cloud credentials
    // Verify startup fails with actionable error
}
```

### E2E Tests
- Deploy OpenCost with various misconfigurations
- Verify appropriate fail-fast behavior
- Verify error messages are actionable

---

## Conclusion

The current codebase has good fail-fast behavior for core infrastructure (Kubernetes, Prometheus, KubeModel) but allows silent failures in areas that directly impact data quality:

1. **RBAC permissions** - Watchers fail silently
2. **Pricing data** - Continues with stale/default pricing
3. **Cloud credentials** - Falls back to estimates
4. **Storage** - Writes fail silently

By implementing the recommendations in this document, OpenCost will:
- **Fail fast** when critical components are misconfigured
- **Provide clear, actionable error messages** for troubleshooting
- **Expose metrics** for monitoring data quality
- **Prevent silent data quality issues** that lead to incorrect costs

The key principle: **If we can't collect accurate cost data, we should fail fast rather than produce incorrect results.**