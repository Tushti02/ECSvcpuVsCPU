# Storage Builder - Production Deployment Guide
## High CPU Usage Fix (9113% CPU Issue)

### ?? CRITICAL ISSUE SUMMARY

**Current Production Resources:**
- CPU: 4 units (0.004 vCPU / 0.39% of 1 core)
- Memory: 256 MB reserved
- Status: **SEVERELY UNDER-RESOURCED**

**Observed Symptoms:**
- CPU utilization: 9113% (attempting to use 91x allocated CPU)
- Service timeouts and throttling
- Extremely slow UpdateAll operations (hours to complete)
- Potential OutOfMemoryException risks

**Root Causes Identified:**
1. ? **FIXED**: Item-by-item serialization (millions of async operations)
2. ? **CRITICAL**: Insufficient ECS task resources
3. ?? **RECOMMENDED**: No incremental update support (always full sync)
4. ?? **RECOMMENDED**: Sequential processing (MaxParallelism=1)

---

## ?? FIXES IMPLEMENTED IN THIS BRANCH

### Fix 1: Memory-Aware Batch Serialization
**File:** `../Shared/SnapshotStorage/Data/SerializeHelper.cs`

**Changes:**
- Replaced item-by-item serialization with intelligent batching
- Dynamic batch size based on available memory:
  - 256 MB containers: 500 items/batch (ultra-conservative)
  - 512 MB containers: 2,000 items/batch
  - 1 GB containers: 5,000 items/batch
  - 2 GB+ containers: 10,000+ items/batch
- Added memory pressure monitoring
- Automatic Gen0 GC suggestions when memory >75%

**Expected Impact:**
- 90-95% reduction in async state machine allocations
- 80-90% reduction in CPU context switching
- More predictable memory usage
- Still limited by ECS resource constraints

---

## ?? DEPLOYMENT OPTIONS

### ? OPTION A: Increase ECS Resources (STRONGLY RECOMMENDED)

This is the **only sustainable solution** for production workloads.

#### Updated Task Definition

**Current (Inadequate):**
```json
"cpu": 4,              // 0.004 vCPU
"memoryReservation": 256  // 256 MB
```

**Recommended (Production-Ready):**
```json
"cpu": 2048,           // 2 vCPU
"memoryReservation": 2048,  // 2 GB soft limit
"memory": 4096         // 4 GB hard limit
```

**Alternative (Minimum Viable):**
```json
"cpu": 1024,           // 1 vCPU
"memoryReservation": 1024,  // 1 GB soft limit
"memory": 2048         // 2 GB hard limit
```

#### Deployment Steps

1. **Create new task definition revision:**
   ```bash
   # Use the provided task-definition-recommended.json
   aws ecs register-task-definition \
     --cli-input-json file://StorageBuilder/task-definition-recommended.json
   ```

2. **Update service:**
   ```bash
   aws ecs update-service \
     --cluster <your-cluster> \
     --service storage-bldr \
     --task-definition PRODUCTION-storage-bldr:45  # New revision number
   ```

3. **Monitor deployment:**
   ```bash
   aws ecs describe-services \
     --cluster <your-cluster> \
     --services storage-bldr
   ```

4. **Verify metrics after 1 hour:**
   - CPU utilization should be <200% (under 2 vCPU)
   - Memory should be <70% of reservation
   - UpdateAll operations should complete in reasonable time

#### Expected Outcomes with 2 vCPU / 2 GB:
- ? CPU: 30-50% utilization (healthy)
- ? Memory: 500-800 MB (comfortable headroom)
- ? UpdateAll duration: 30-90 minutes (vs. hours)
- ? No CPU throttling
- ? No OOM risks

#### Cost Impact:
- **Current:** ~$2-5/month (essentially free-tier)
- **2 vCPU/2GB:** ~$30-50/month (reasonable for production)
- **Justification:** Service is currently non-functional; this makes it operational

---

### ?? OPTION B: Deploy Code Fix Only (NOT RECOMMENDED)

Deploy the batching fix without increasing resources. This will help but **won't fully resolve the issue**.

#### Steps:

1. **Build and push new image:**
   ```bash
   cd StorageBuilder
   docker build -t 722382204906.dkr.ecr.us-east-1.amazonaws.com/storage-builder-service:1.0.XXXX .
   docker push 722382204906.dkr.ecr.us-east-1.amazonaws.com/storage-builder-service:1.0.XXXX
   ```

2. **Update task definition with new image:**
   ```bash
   aws ecs register-task-definition \
     --family PRODUCTION-storage-bldr \
     --container-definitions '[{
       "name": "storage-bldr",
       "image": "722382204906.dkr.ecr.us-east-1.amazonaws.com/storage-builder-service:1.0.XXXX",
       "cpu": 4,
       "memoryReservation": 256,
       ... (rest of existing config)
     }]'
   ```

3. **Update service:**
   ```bash
   aws ecs update-service \
     --cluster <your-cluster> \
     --service storage-bldr \
     --task-definition PRODUCTION-storage-bldr:45
   ```

#### Expected Outcomes with 0.004 vCPU / 256 MB:
- ?? CPU: Still 500-2000% (significantly throttled but better)
- ?? Memory: 180-230 MB (very tight, possible OOM)
- ?? UpdateAll duration: Still 3-10+ hours
- ?? Still experiencing CPU throttling
- ?? Risk of OOM during large client updates

**This is a band-aid, not a cure.**

---

## ?? RECOMMENDED DEPLOYMENT STRATEGY

### Phase 1: IMMEDIATE (This Week)
1. ? Deploy code fix (already in this branch)
2. ? Increase ECS resources to 2 vCPU / 2 GB (see Option A)
3. Monitor for 48 hours

### Phase 2: SHORT-TERM (Next Sprint)
Implement **Option 4: Version Tracking** (see detailed guide below)
- Store last sync version per storage
- Enable incremental updates (delta only)
- Expected improvement: 90-99% reduction in data transferred

### Phase 3: MEDIUM-TERM (Following Sprint)
Implement **Option 2: Increase Parallelism** (see detailed guide below)
- Bump MaxParallelism from 1 to 4
- Process multiple targets simultaneously
- Expected improvement: 4x faster completion time

### Phase 4: LONG-TERM (Optional)
Implement **Option 5: Background Queue System** (see detailed guide below)
- Better monitoring and observability
- Automatic retries
- Progress tracking

---

## ?? MONITORING & VALIDATION

### Key Metrics to Track

**Before Fix:**
```
CPU: 9113% (91x over allocation)
Memory: ~200 MB (78% of 256 MB)
UpdateAll Duration: 8-20+ hours
Errors: Frequent timeouts, possible OOM
```

**After Fix (with code + 2vCPU/2GB):**
```
CPU: 30-60% (healthy utilization)
Memory: 600-1200 MB (comfortable)
UpdateAll Duration: 30-90 minutes
Errors: Minimal/none
```

### CloudWatch Alarms to Configure

```bash
# CPU Utilization - Alert if >80%
aws cloudwatch put-metric-alarm \
  --alarm-name storage-bldr-high-cpu \
  --metric-name CPUUtilization \
  --namespace AWS/ECS \
  --statistic Average \
  --period 300 \
  --threshold 80 \
  --comparison-operator GreaterThanThreshold \
  --evaluation-periods 2

# Memory Utilization - Alert if >85%
aws cloudwatch put-metric-alarm \
  --alarm-name storage-bldr-high-memory \
  --metric-name MemoryUtilization \
  --namespace AWS/ECS \
  --statistic Average \
  --period 300 \
  --threshold 85 \
  --comparison-operator GreaterThanThreshold \
  --evaluation-periods 2
```

### Application Logs to Monitor

```bash
# Check batch size being used
aws logs tail storage-bldr-PRODUCTION --follow | grep "SerializeHelper.*batch size"

# Monitor update operations
aws logs tail storage-bldr-PRODUCTION --follow | grep "updated to version"

# Watch for memory pressure
aws logs tail storage-bldr-PRODUCTION --follow | grep "memory"
```

---

## ?? ROLLBACK PLAN

If issues occur after deployment:

1. **Immediate rollback to previous task definition:**
   ```bash
   aws ecs update-service \
     --cluster <your-cluster> \
     --service storage-bldr \
     --task-definition PRODUCTION-storage-bldr:44  # Previous revision
   ```

2. **Monitor for stabilization (5 minutes)**

3. **Investigate logs:**
   ```bash
   aws logs tail storage-bldr-PRODUCTION --since 30m
   ```

4. **Common issues and fixes:**
   - **OOM errors:** Increase memory reservation
   - **High CPU (>90%):** Increase CPU allocation
   - **Slow performance:** Check MaxParallelism setting
   - **Connection errors:** Check database/S3 connectivity

---

## ?? TESTING CHECKLIST

Before deploying to production:

### Pre-Deployment
- [ ] Code builds successfully
- [ ] Unit tests pass
- [ ] Docker image builds
- [ ] Task definition JSON validated
- [ ] ECS cluster has capacity for new resources

### Post-Deployment
- [ ] Task starts successfully (check ECS console)
- [ ] Health check passes
- [ ] Service registers in Consul
- [ ] Logs show expected batch sizes
- [ ] Test `/health/storage-bldr` endpoint
- [ ] Test `/storage-builder/update?clientId=X` for single client
- [ ] Monitor for 1 hour before testing full sync

### Full Sync Testing
- [ ] Trigger `/storage-builder/update-all`
- [ ] Monitor CPU/Memory metrics
- [ ] Check CloudWatch logs for errors
- [ ] Verify S3 uploads complete
- [ ] Confirm task completes (check `/storage-builder/is-completed?taskId=X`)
- [ ] Validate data integrity in S3

---

## ?? SECURITY CONSIDERATIONS

The code changes do not modify:
- Authentication/Authorization
- API surface area
- Network configuration
- Secrets management
- IAM permissions

No security review required for this deployment.

---

## ?? SUPPORT & ESCALATION

**If deployment fails:**
1. Immediately rollback (see Rollback Plan)
2. Collect logs: `aws logs tail storage-bldr-PRODUCTION --since 1h > failure-logs.txt`
3. Check ECS events: `aws ecs describe-services --cluster <cluster> --services storage-bldr`
4. Contact: [Your DevOps team contact info]

**Expected deployment time:**
- Option A (Code + Resources): 20-30 minutes
- Option B (Code Only): 10-15 minutes

**Recommended deployment window:**
- Low-traffic period
- Have rollback person standing by
- Monitor for 2 hours post-deployment

---

## ?? SUCCESS CRITERIA

Deployment is considered successful when:
1. ? CPU utilization <80% during UpdateAll
2. ? Memory utilization <85% at peak
3. ? No OOM errors in logs
4. ? UpdateAll completes in <2 hours
5. ? No timeout errors reported
6. ? All 6 storage types upload successfully per client
7. ? S3 data validated and accessible

---

## ?? NEXT STEPS AFTER DEPLOYMENT

Once stable, consider implementing these enhancements:

1. **Version Tracking (High Priority)**
   - Reduces data transfer by 90-99%
   - See: Option 4 in previous documentation
   - Estimated effort: 2-3 days

2. **Parallel Processing (Medium Priority)**
   - 4x faster completion time
   - See: Option 2 in previous documentation
   - Estimated effort: 1-2 days

3. **Background Queue System (Low Priority)**
   - Better observability and retry logic
   - See: Option 5 in previous documentation
   - Estimated effort: 3-5 days

4. **Monitoring Improvements**
   - Custom CloudWatch metrics for batch processing
   - S3 upload success/failure tracking
   - Per-client processing time metrics

---

## ? APPROVAL CHECKLIST

Before proceeding to production:
- [ ] Technical Lead approval
- [ ] DevOps approval for resource increase
- [ ] Cost center approval (if increasing resources)
- [ ] Tested in staging environment
- [ ] Rollback plan reviewed
- [ ] On-call engineer notified
- [ ] Deployment window scheduled
- [ ] Monitoring dashboards prepared

---

**Document Version:** 1.0
**Last Updated:** [Current Date]
**Branch:** `fb/cnc/PBI1280574-fix-highcpu-usage`
**Author:** [Your Name]
