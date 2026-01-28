# Storage Builder - Quick Fix Reference Card

## ?? PROBLEM
**9113% CPU utilization** = Container trying to use 91x more CPU than allocated

## ? SOLUTION (2 Parts Required)

### Part 1: Code Fix ? DONE
**Branch:** `fb/cnc/PBI1280574-fix-highcpu-usage`
**File Changed:** `../Shared/SnapshotStorage/Data/SerializeHelper.cs`
**Action:** Merge and deploy this branch

### Part 2: Increase Resources ?? ACTION REQUIRED
**Current:** 4 CPU units (0.004 vCPU), 256 MB memory
**Needed:** 2048 CPU units (2 vCPU), 2048 MB memory

## ?? QUICK DEPLOYMENT STEPS

### Step 1: Deploy Code (15 min)
```bash
# Build and push image
docker build -t 722382204906.dkr.ecr.us-east-1.amazonaws.com/storage-builder-service:1.0.XXXX .
docker push 722382204906.dkr.ecr.us-east-1.amazonaws.com/storage-builder-service:1.0.XXXX
```

### Step 2: Update ECS Resources (10 min)
```bash
# Register new task definition with 2 vCPU / 2 GB
aws ecs register-task-definition \
  --cli-input-json file://StorageBuilder/task-definition-recommended.json

# Update service
aws ecs update-service \
  --cluster <your-cluster> \
  --service storage-bldr \
  --task-definition PRODUCTION-storage-bldr:45
```

### Step 3: Monitor (2 hours)
```bash
# Watch CloudWatch metrics
# - CPU should be <80%
# - Memory should be <85%
# - No timeout errors
```

## ?? EXPECTED RESULTS

| Metric | Before | After |
|--------|--------|-------|
| CPU | 9113% ? | 30-60% ? |
| Memory | 200 MB ?? | 600-1200 MB ? |
| UpdateAll Time | 8-20 hours ? | 30-90 min ? |

## ?? COST
**Increase:** ~$30-50/month (~$0.04/hour)
**Justification:** Service currently non-functional without this

## ?? ROLLBACK (If Needed)
```bash
aws ecs update-service \
  --cluster <your-cluster> \
  --service storage-bldr \
  --task-definition PRODUCTION-storage-bldr:44  # Previous revision
```

## ?? FULL DOCUMENTATION
- **Executive Summary:** `StorageBuilder/EXECUTIVE-SUMMARY.md`
- **Deployment Guide:** `StorageBuilder/PRODUCTION-DEPLOYMENT-GUIDE.md`
- **Task Definition:** `StorageBuilder/task-definition-recommended.json`

## ? PRE-FLIGHT CHECKLIST
- [ ] Code builds successfully (`dotnet build`)
- [ ] Branch merged to main/master
- [ ] Docker image pushed to ECR
- [ ] Task definition JSON validated
- [ ] Cost approval obtained
- [ ] On-call engineer notified
- [ ] Deployment window scheduled
- [ ] Rollback plan reviewed

## ?? EMERGENCY CONTACTS
- **Technical Issues:** [Your team contact]
- **Cost Approvals:** [Finance contact]
- **Infrastructure:** [DevOps contact]

---
**?? IMPORTANT:** Both Part 1 (code) AND Part 2 (resources) are required for full fix!
