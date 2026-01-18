# Comprehensive Lab: Hooks, Waves, Health, and Sync Options

## Objective
Create a production-like application that demonstrates all concepts together

## Scenario
You're deploying an e-commerce application with:
- Database requiring migrations
- Multiple microservices with dependencies
- Health checks and smoke tests
- Proper cleanup on failure

## Task 1: Build Complete Application

### Requirements
1. Use sync waves for proper ordering
2. Implement PreSync hook for database migration
3. Implement PostSync hook for smoke tests
4. Add custom health checks
5. Configure appropriate sync options
6. Add SyncFail hook for cleanup

### Architecture
```
PreSync Hooks:
├── Check external dependencies
└── Run database migrations

Wave -2: ConfigMaps, Secrets
Wave -1: PVCs
Wave  0: Database
Wave  1: Redis, RabbitMQ
Wave  2: Auth Service
Wave  3: Core Services (parallel)
Wave  4: API Gateway
Wave  5: Frontend

PostSync Hooks:
├── Smoke tests
├── Performance tests
└── Send notification

If Failure:
└── SyncFail: Rollback and alert
```

### Implementation
```bash
# Create the application structure
mkdir -p ../argocd-demo-apps/production-demo/manifests/{infra,services,tests}

# Implement each component with appropriate:
# - sync-wave annotations
# - hook annotations  
# - health checks
# - resource limits
# - sync options
```

## Task 2: Test Failure Scenarios

### Test 2.1: PreSync Hook Failure
1. Make migration hook fail
2. Observe: Sync stops, main app not deployed
3. SyncFail hook executes
4. Fix and retry

### Test 2.2: Database Unhealthy
1. Use invalid database configuration
2. Observe: Wave 0 never becomes healthy
3. Wave 1+ resources never created
4. Fix wave 0, watch cascade deployment

### Test 2.3: PostSync Smoke Test Failure
1. App deploys successfully
2. Smoke test fails
3. App marked as degraded
4. Decision: Rollback or fix forward?

## Task 3: Optimize Sync Performance

### Current State
- Sequential deployment
- Each wave waits for previous
- Total time: 10+ minutes

### Optimization
1. Identify resources that can deploy in parallel
2. Group into same wave
3. Reduce total number of waves
4. Measure improvement

### Target
- Total sync time: < 5 minutes
- No compromise on safety
- Maintain dependency ordering

## Task 4: Document Production Deployment

Create runbook with:
1. Pre-deployment checklist
2. Expected sync timeline
3. Verification steps
4. Rollback procedure
5. Troubleshooting guide

## Success Criteria

✅ Application deploys in correct order
✅ All hooks execute successfully
✅ Health checks pass
✅ Failed sync triggers cleanup
✅ Documentation is complete
✅ Sync time is optimized

## Bonus Challenges

### Challenge 1: Blue-Green with Waves
Implement blue-green deployment using waves:
- Wave 0: Deploy green (new version)
- Wave 1: Run tests against green
- Wave 2: Switch traffic to green
- Wave 3: Shutdown blue (old version)

### Challenge 2: Multi-Region Deployment
Use waves to deploy across regions:
- Wave 0: Region 1 (canary)
- Wave 1: Region 2
- Wave 2: Region 3
- Wave 3: Remaining regions

### Challenge 3: Dependency Graph
Create visualization of:
- All resources
- Their wave numbers
- Dependencies between them
- Total deployment flow

## Submission

Create PR with:
1. All manifests
2. ArgoCD Application definition
3. Documentation
4. Test results
5. Performance metrics
