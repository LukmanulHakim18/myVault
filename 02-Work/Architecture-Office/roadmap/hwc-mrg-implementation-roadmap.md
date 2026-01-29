---
type: roadmap
project: MRG
status: planning
owner: Adam
reviewer: Lukman
timeline: W2-W4 January
priority:
  - P0
  - P1
  - P2
tags:
  - hwc
  - infrastructure
  - mrg
  - assessment
  - implementation
---
# 🚀 HWC Assessment Implementation Roadmap - MRG
## mybluebird-prod-bb-8445

**Project Lead:** Adam  
**Reviewer:** Lukman  
**DB/Cache Specialist:** Imam  
**Status:** Planning Phase  
**Document Date:** 26 January 2026  
**Timeline:** W2 Jan - W4 Jan (4 weeks)

---

## 📊 Executive Summary

- **Total Items:** 12 (6 High Risk, 6 Low Risk)
- **Already Mitigated:** 3 items (verification only)
- **Action Required:** 9 items
- **Critical Urgency (P0):** 2 items (DCS Redis)
- **High Importance (P1):** 3 items (ECS SPOF, RDS, Health Probes)
- **Medium Priority (P2):** 4 items (Cost optimization, Best practices)

**Estimated Effort:**
- **P0 Items:** 2-3 weeks (code optimization + testing)
- **P1 Items:** 3-4 weeks (migration + configuration)
- **P2 Items:** 2-3 weeks (implementation + validation)

**Overall Timeline:** 4 weeks (parallel execution recommended)

---

## 🎯 Priority Breakdown

### 🔴 **PRIORITY P0 - CRITICAL (IMMEDIATE)**

**Items:** #22, #23  
**Service:** DCS (Redis)  
**Impact:** Production performance degradation, service latency  
**Owner:** Adam + Imam  
**Timeline:** W2 - W3 January

---

### 🟠 **PRIORITY P1 - HIGH (IMPORTANT)**

**Items:** #13, #3, #10  
**Services:** ECS (SPOF), RDS (Durability), CCE (Health)  
**Impact:** Infrastructure stability, disaster recovery  
**Owner:** Adam  
**Timeline:** W2 - W4 January

---

### 🟡 **PRIORITY P2 - MEDIUM (CAN WAIT)**

**Items:** #2, #5, #7, #9, #17, #18, #25  
**Services:** Multiple  
**Impact:** Cost optimization, best practices, verification  
**Owner:** Adam  
**Timeline:** W2 - W4 January

---

## 📋 Detailed Action Plan

---

# P0: CRITICAL ITEMS (Belum Mitigated)

## Item #22 - DCS: Dangerous Redis Commands

**Service:** DCS (Redis)  
**Risk Level:** 🔴 HIGH  
**Status:** ❌ NOT FIXED  
**Impact Score:** 9/10  
**Owner:** Adam + Imam  
**Timeline:** TBI W3 January  

### Problem Description

Dangerous commands detected in Redis operation logs:
- SCAN, KEYS, HGETALL, HSCAN, SMEMBERS, ZREVRANGE, LRANGE

Risk: These commands have O(n) complexity and can block Redis server, affecting ALL connected clients and causing service-wide latency spikes.

Condition: Log records ≥ 1 (meaning at least one dangerous command executed)

### Impact Analysis

**Severity:** 🔴 CRITICAL  
**Affected Services:** All services using MRG Redis cache  
**User Impact:** 
- Latency spikes (100ms - 1s+)
- Request timeouts
- Service degradation under load

**Monitoring Evidence:**
- Query metric: DCS operation logs (last 24h)
- Aggregate: Commands that appear in logs
- Baseline: Zero dangerous commands should appear

### Root Cause Analysis

1. **Code Pattern Issue:**
   - Service code using inefficient Redis query patterns
   - Likely: Scanning entire keyspace or large hash/set operations
   - Example: `KEYS user:*` to find all user keys (instead of maintaining index)

2. **Specific Commands to Investigate:**
   - `KEYS pattern` - Full keyspace scan
   - `SCAN cursor` - Iterative scan (less bad but still risky)
   - `HGETALL key` - Get entire hash (expensive on large hashes)
   - `SMEMBERS key` - Get entire set (expensive on large sets)
   - `ZREVRANGE key 0 -1` - Get entire sorted set

### Action Items

#### Week 2 (Immediate)

**Task 1: Extract & Analyze Logs (Effort: 4h)**
- Extract dangerous command logs from last 7 days
- Identify which services/endpoints are calling them
- Group by command type and frequency
- Create summary report

**Deliverable:** Investigation report with top 10 problematic commands

**Task 2: Code Review Audit (Effort: 8h)**
- Search codebase for dangerous commands
- Map to specific functions/services
- Identify business logic intent
- Flag for refactoring

**Deliverable:** Code audit report with locations and business logic

#### Week 3 (Optimization & Fix)

**Task 3: Design Refactoring (Effort: 6h)**

For each dangerous command usage - implement safer patterns with Go examples provided in detail.

**Deliverable:** Refactoring design document with Go code examples

**Task 4: Implementation (Effort: 16h - 2 days)**
- Implement refactored Redis usage patterns
- Add comprehensive unit tests
- Test with production-like data volume

**Code Review Checklist:**
- [ ] No KEYS command usage
- [ ] No SCAN usage in hot path
- [ ] HGETALL only on small hashes (<100 fields)
- [ ] SMEMBERS/ZREVRANGE only on small collections
- [ ] Pagination implemented for large collections
- [ ] New index structure properly maintained
- [ ] Unit tests cover edge cases
- [ ] Performance benchmarked

**Task 5: Staging Validation (Effort: 8h)**
- Deploy to staging environment
- Load test with production traffic patterns
- Monitor Redis command logs for dangerous commands
- Verify latency improvements

**Success Criteria:**
- [ ] Zero dangerous commands in logs (24h observation)
- [ ] P99 latency reduced by >30%
- [ ] All unit tests pass
- [ ] No functional regressions

#### Week 4 (Production Deployment & Monitoring)

**Task 6: Production Rollout (Effort: 4h)**
- Blue-green or canary deployment
- Monitor for errors & latency
- Gradual rollout: 10% → 50% → 100%

**Success Criteria:**
- [ ] Zero critical errors on rollout
- [ ] Latency improvement sustained >24h
- [ ] No increase in error rate

**Task 7: Post-Deployment Monitoring (Ongoing)**
- Daily check of Redis logs for dangerous commands
- Weekly performance report
- Alert if dangerous commands reappear

### Effort Estimate

| Task | Effort | Duration | Owner |
|------|--------|----------|-------|
| Log extraction & analysis | 4h | W2 Day 1 | Imam |
| Code audit | 8h | W2 Day 2-3 | Adam + Imam |
| Refactoring design | 6h | W2 Day 4 | Adam |
| Implementation | 16h | W3 Full week | Adam |
| Staging validation | 8h | W3 Day 4-5 | Adam + Imam |
| Prod deployment | 4h | W4 Day 1 | Adam |
| **Total** | **46 hours** | **3 weeks** | **Adam + Imam** |

### Dependencies

- [ ] Access to Redis operation logs (HWC console or CLI)
- [ ] MRG codebase available for audit
- [ ] Staging environment with production-like data
- [ ] Code review & approval from tech lead

### Success Criteria

- [ ] Zero dangerous Redis commands in logs (24h observation post-deploy)
- [ ] P99 latency reduced by minimum 30%
- [ ] P95 latency reduced by minimum 40%
- [ ] All unit tests pass
- [ ] No functional regressions reported
- [ ] Monitoring alerts configured

### Risks & Mitigation

| Risk | Probability | Impact | Mitigation |
|------|-------------|--------|-----------:|
| Missing refactoring in code | Medium | High | Comprehensive grep search, code review |
| Performance not improved | Low | High | Load test in staging first |
| Functional regressions | Medium | High | Comprehensive unit tests |
| Slow rollout | Low | Medium | Prepare rollback plan |

---

## Item #23 - DCS: Slow Commands O(n) Complexity

**Service:** DCS (Redis)  
**Risk Level:** 🔴 HIGH  
**Status:** ❌ NOT FIXED  
**Impact Score:** 8/10  
**Owner:** Adam + Imam  
**Timeline:** TBC Team W2  
**Related to:** #22 (Usually same root cause)

### Problem Description

Slow commands detected in Redis:
- Parameter is_slow_log_exist ≥ 1
- O(n) complexity commands in slow log
- Indicates blocking operations on large datasets

Slow Log Configuration: 
- Log threshold: 10,000 microseconds (10ms)
- Any command exceeding this appears in slow log

### Impact Analysis

**Severity:** 🔴 CRITICAL  
**Affected Services:** Services using MRG Redis  
**User Impact:**
- Request timeout (if exceeds timeout threshold)
- Service latency spikes
- Connection pool exhaustion (slow queries hold connections)

### Root Cause Analysis

**Most Likely:** Same as #22 - Dangerous O(n) commands

**Additional Causes:**
- Large dataset in Redis (millions of keys/entries)
- Burst traffic causing connection pool contention
- GC pauses on Redis server during operations

### Action Items

**Note:** #22 and #23 are typically related. Fix #22 likely resolves #23.

#### Specific Investigation

**Task 1: Slow Log Analysis (Effort: 4h)**

Analyze Redis slow log to identify top 20 slowest commands

**Deliverable:** Top 20 slowest commands analysis

**Task 2: Performance Profiling (Effort: 8h)**
- Profile Redis during peak traffic
- Identify O(n) commands and their datasets
- Estimate dataset size

**Deliverable:** Profiling report with dataset sizes

#### Implementation

Same as #22 - Implement refactored Redis patterns to remove slow commands.

**Additional Optimization:**
- Monitor slow log post-fix to ensure O(n) commands removed
- Consider Redis cluster if dataset continues growing
- Set up automated slow query alerts

### Effort Estimate

| Task | Effort | Owner |
|------|--------|-------|
| Slow log analysis | 4h | Imam |
| Performance profiling | 8h | Adam + Imam |
| Implementation (same as #22) | 16h | Adam |
| Validation | 8h | Adam + Imam |
| **Total** | **36 hours** | **Adam + Imam** |

### Success Criteria

- [ ] Zero slow commands in Redis slow log (24h observation)
- [ ] P99 latency < 100ms (from current spike baseline)
- [ ] No commands exceeding 10ms threshold
- [ ] Monitoring alerts configured for future slow commands

### Timeline Coordination

**With #22:** Coordinate implementation so both fixes deployed together (W3)
- Both involve same code refactoring
- Testing covers both scenarios
- Single deployment reduces risk

---

# P1: HIGH PRIORITY ITEMS

## Item #13 - ECS: Multiple ECS on Same Physical Host

**Service:** ECS (Virtual Machines)  
**Risk Level:** 🔴 HIGH  
**Status:** ❌ NOT FIXED  
**Impact Score:** 8/10  
**Owner:** Adam  
**Timeline:** W2-W4 January (Extended)  
**Related to:** #18 (ECS Groups configuration)

### Problem Description

Current State:
- Multiple MRG ECS instances deployed on same physical host
- No anti-affinity policy → Kubernetes can schedule all replicas on 1 host
- If physical host fails → ALL service instances go down (SPOF)

Risk: Single Point of Failure at infrastructure level

### Impact Analysis

**Severity:** 🔴 CRITICAL (Infrastructure level)  
**SLA Impact:** Service downtime during host failure  
**MTTR:** Recovery requires host restoration or failover

**Failure Scenario:**
Host X fails → API-Pod-1, API-Pod-2, API-Pod-3 all die → Zero replicas running → Service completely down until host recovers

### Root Cause

**Underlying Cause:** #18 - No ECS Groups configured

ECS Groups enforce anti-affinity at IaaS level (Huawei Cloud hypervisor):
- Kubernetes anti-affinity works at pod scheduler level
- But hypervisor can still place multiple VMs on same host
- ECS Groups prevent this at hypervisor level

### Action Items

#### Week 2 (Planning & Assessment)

**Task 1: Current State Assessment (Effort: 4h)**

Check current ECS placement distribution

**Deliverable:** Current placement distribution report

**Task 2: Design ECS Groups Strategy (Effort: 6h)**

**Option A: Host Group (Best Practice)** - Create dedicated physical hosts per tier
**Option B: Spreading (Default)** - Each instance on different host

**Recommendation:** Option A (Host Groups) - provides both HA and cost efficiency

**Deliverable:** ECS Groups configuration design document

#### Week 3-4 (Implementation & Migration)

**Task 3: Create ECS Groups (Effort: 2h)**

Create host groups for each service tier in Huawei Cloud

**Deliverable:** ECS Groups created in Huawei Cloud

**Task 4: Rolling Migration (Effort: 12h - 1-2 days)**

**Challenge:** Cannot change ECS group without recreating instance

**Migration Strategy:**
- Create replacement ECS in new group
- Migrate traffic/data to new ECS
- Terminate old ECS
- Repeat for all instances

**Timeline per Instance:** 60-90 minutes  
**Total for 6 instances:** 6-9 hours  
**With validation time:** 12-15 hours

**Task 5: Kubernetes Anti-Affinity (Parallel with ECS, Effort: 4h)**

Implement pod-level anti-affinity as backup to ECS Groups

**Deliverable:** Updated deployment manifests with anti-affinity

**Task 6: Validation & Testing (Effort: 6h)**

1. **Placement Verification (1h)** - Verify all instances on different hosts
2. **Failover Testing (4h)** - Simulate host failure and verify pod reschedules
3. **Load Test (1h)** - Run production traffic simulation

**Success Criteria:**
- [ ] All instances on different physical hosts
- [ ] Pod anti-affinity rules enforced
- [ ] Service survives single host failure
- [ ] No latency increase post-migration

### Effort Estimate

| Phase | Task | Effort | Duration |
|-------|------|--------|----------|
| W2 | Assessment | 4h | Day 1-2 |
| W2 | Design | 6h | Day 3 |
| W3 | Create ECS Groups | 2h | Day 1 |
| W3 | Rolling migration | 12h | Day 1-2 |
| W3 | Kubernetes anti-affinity | 4h | Day 2 |
| W4 | Validation | 6h | Day 1-2 |
| **Total** | | **34h** | **2.5 weeks** |

### Dependencies

- [ ] Access to Huawei Cloud ECS management
- [ ] Access to Kubernetes cluster configuration
- [ ] Load Balancer configuration authority
- [ ] Maintenance window (production cutover time)

### Risks & Mitigation

| Risk | Mitigation |
|------|-----------|
| Service downtime during migration | Run during maintenance window, have rollback plan |
| Data loss during stateful migration | Backup snapshots before migration |
| Migration slower than planned | Parallelize across services |
| Rollback failure | Test rollback procedure before production |

### Success Criteria

- [ ] Zero instances on same physical host
- [ ] Pod anti-affinity rules enforced
- [ ] Service survives host failure simulation
- [ ] No performance regression
- [ ] RTO < 5 minutes for single host failure

---

## Item #3 - RDS: MySQL Transaction Log Configuration

**Service:** RDS (MySQL Database)  
**Risk Level:** 🟡 LOW  
**Status:** ❌ NOT FIXED  
**Impact Score:** 6/10  
**Owner:** Adam + Imam  
**Timeline:** W2 January (Quick fix)

### Problem Description

MySQL durability settings not optimal:
- sync_binlog not set to 1 (default: 0)
- innodb_flush_logs_at_trx_commit not set to 1 (default: 0)

Risk: Data durability compromise
- Transactions may be lost on crash
- Replication lag potential

### Action Items

#### Investigation (Effort: 1h)

Check current MySQL settings

#### Implementation (Effort: 2h)

**Option 1: Via Huawei Cloud Console**
- Go to RDS instance details
- Modify parameter group
- Set sync_binlog = 1
- Set innodb_flush_logs_at_trx_commit = 1
- Apply changes (requires restart)

**Option 2: Via SQL** (if RDS allows)

**Note:** This requires RDS restart → plan for maintenance window

#### Validation (Effort: 1h)

Verify settings applied and replication healthy

### Effort Estimate

- Investigation: 1h
- Implementation: 2h (+ maintenance window)
- Validation: 1h
- **Total: 4 hours**

### Success Criteria

- [ ] sync_binlog = 1
- [ ] innodb_flush_logs_at_trx_commit = 1
- [ ] Replication continues normally
- [ ] No query performance regression

---

## Item #10 - CCE: Missing Health Probes

**Service:** CCE (Kubernetes)  
**Risk Level:** 🟡 LOW  
**Status:** ❌ NOT FIXED  
**Impact Score:** 7/10  
**Owner:** Adam  
**Timeline:** W2 - W3 January

### Problem Description

Applications lack health probes in Kubernetes:
- No liveness probes: Dead pods continue serving traffic
- No readiness probes: Unready pods receive traffic
- No startup probes: Slow-starting services wrongly marked unhealthy

Risk: Pod not restarted when crashed, traffic routed to unhealthy pods

### Action Items

#### Audit (Effort: 2h)

Check all deployments for missing probes

#### Implementation (Effort: 8h)

For each deployment, add startup, liveness, and readiness probes

**Note:** Requires health endpoints in application code:
- GET `/health/startup` - Indicates service ready to serve
- GET `/health/live` - Indicates service is alive
- GET `/health/ready` - Indicates service ready to accept traffic

#### Validation (Effort: 2h)

Verify probes deployed and working correctly

### Effort Estimate

- Audit: 2h
- Implementation: 8h (1-2h per deployment, 6-8 deployments)
- Validation: 2h
- **Total: 12 hours**

### Success Criteria

- [ ] All deployments have startup, liveness, readiness probes
- [ ] Health endpoints implemented in services
- [ ] Probes working correctly (verified via logs)
- [ ] Pod restarts on health check failure

---

# P2: MEDIUM PRIORITY ITEMS

## Items #2, #5, #7 - Already Mitigated (Verification Only)

### #2 - RDS: General-Purpose Instance Class
- **Status:** ✅ Already mitigated
- **Action:** Verify mitigation documented
- **Effort:** 30 min
- **Timeline:** W2 Jan

### #5 - EVS: Missing Disk Snapshots  
- **Status:** ✅ Already mitigated
- **Action:** Verify automated snapshot schedule
- **Effort:** 30 min
- **Timeline:** W2 Jan

### #7 - CCE: Single Pod Workload
- **Status:** ✅ Already mitigated
- **Action:** Verify minimum 2 replicas + PodDisruptionBudget
- **Effort:** 30 min
- **Timeline:** W2 Jan

## Item #9 - CCE: Anti-affinity Policies

**Timeline:** TBI W3 January (Best practice improvement)
**Effort:** 4 hours

## Items #17, #25 - Cost Optimization (Pending Approval)

**Status:** Pending cost analysis
**Timeline:** TBC Cost W2

## Item #18 - ECS: No ECS Groups Configured

**Related to:** #13 (Same implementation)
**Effort:** Included in #13 roadmap

---

# 📅 Weekly Timeline Summary

## Week 2 (Jan 20-24) - PLANNING & QUICK WINS

**Deliverables:**
- Redis log analysis report
- Code audit report
- Refactoring design document
- ECS Groups strategy document
- RDS config applied

## Week 3 (Jan 27-31) - IMPLEMENTATION & TESTING

**Deliverables:**
- Code deployed to staging
- Staging test results
- ECS migration completed
- Health probes configured

## Week 4 (Feb 3-7) - PRODUCTION DEPLOYMENT & VALIDATION

**Deliverables:**
- Production deployment complete
- Monitoring configured
- Validation report
- Risk mitigation confirmed

---

# 🎯 Success Metrics

## P0 Items (#22, #23) - Redis Performance

**Target Metrics (Post-Deployment):**
- [ ] Zero dangerous commands in Redis logs (24h observation)
- [ ] Zero slow commands in Redis slow log (24h observation)
- [ ] P99 latency: < 100ms
- [ ] P95 latency: < 50ms
- [ ] Request timeout rate: 0%
- [ ] Service uptime: 99.99%

## P1 Items (#13, #3, #10) - Infrastructure Stability

**#13 - ECS Placement:**
- [ ] All MRG ECS instances on different physical hosts
- [ ] Service survives single host failure (< 5min RTO)
- [ ] Zero latency increase post-migration
- [ ] No data loss during migration

**#3 - RDS Durability:**
- [ ] sync_binlog = 1
- [ ] innodb_flush_logs_at_trx_commit = 1
- [ ] Replication healthy
- [ ] Zero data loss scenarios

**#10 - Health Probes:**
- [ ] All deployments have startup, liveness, readiness probes
- [ ] Pod restarts < 10 seconds on failure
- [ ] Traffic not routed to unhealthy pods
- [ ] Service availability maintained during rolling updates

---

# 🔔 Approval Gates

| Gate | When | Owner | Approval |
|------|------|-------|----------|
| **P0 Implementation Start** | End of W2 | Adam | Lukman (Reviewer) |
| **P0 Staging Sign-off** | End of W3 Day 2 | Imam | Lukman (Reviewer) |
| **P0 Production Deployment** | W4 Monday | Adam | Lukman (Reviewer) |
| **P1 ECS Migration** | W3 Day 1 | Adam | Lukman (Reviewer) |
| **P1 Health Probes** | W3 Day 4 | Adam | Lukman (Reviewer) |
| **All Items Validation** | End of W4 | Adam | Lukman (Reviewer) |

---

# 📞 Escalation & Contacts

**Lead Engineer:** Adam  
**Code Reviewer:** Lukman (Architecture Engineer)  
**DB/Cache Specialist:** Imam  

**For Blockers:** Escalate to Lukman  
**For Infrastructure Access:** Contact Huawei Cloud team  

---

# 📝 Documentation

All action items should be documented:
- Code changes → GitHub PR with description
- Infrastructure changes → Update architecture documentation
- Configuration changes → Update runbooks
- Monitoring changes → Update alerting rules

---

**Status:** Planning Phase  
**Next Review:** End of W2 January  
**Final Sign-off Target:** W4 Completion

---

*Document maintained by: Adam (MRG Lead)*  
*Reviewed by: Lukman (Architecture Engineer)*  
*Last Updated: 26 January 2026*
