---
type: meeting
date: '2026-01-27'
attendees:
  - VP
  - Lukmanul Hakim
teams:
  - MRG
  - UPG
status: active
projects:
  - MRG
  - UPG
  - MyBB
topics:
  - business-improvement
  - database-assessment
  - load-testing
priority: high
---
# Meeting Update Arsitek - Summary
**Date:** 27 Januari 2026  
**Attendees:** VP, Lukmanul Hakim (Architecture Engineer)  
**Teams:** MRG (Meta Reservation Gateway), UPG (Universal Payment Gateway)

---

## 1. Business Improvement Request: Reduce Process Failure Rate

### VP Directive
Improve business metrics by **reducing flow agal (failures)** and **simplifying processes** to minimize complexity.

### Action Items

#### 1.1 Identify Failure Points
- Analyze current failure rate in booking flow
- Identify bottlenecks in payment pipeline (UPG)
- Review redundant retry logic vs direct dependencies
- Leverage existing SPOF (Single Point of Failure) analysis

#### 1.2 Simplify Order Orchestration
- Evaluate service call batching opportunities
- Assess synchronous vs asynchronous flow optimization
- Flatten dependency chains where possible
- Current flow validation: User → MRG → UPG → downstream services

#### 1.3 Payment Gateway Optimization
- Reduce retry logic complexity
- Implement upstream validation for early failure detection
- Simplify reconciliation process

#### 1.4 Quick Wins to Highlight
- Notification strategy pattern refactoring (already completed)
- Rule engine implementation for reduced manual intervention
- Recent optimization results quantification

### Expected Outcome
- **Metric target:** Reduce failed transactions by X% (TBD - need baseline data)
- **Timeline:** TBD based on technical assessment
- **Effort:** To be estimated by team

---

## 2. Database Assessment: MongoDB vs PostgreSQL for Historical Order Data

### VP Concern
**Cost:** MongoDB pricing adalah 3x lebih mahal dibanding PostgreSQL

### Assessment Framework

#### 2.1 Use Case Analysis
**Questions to answer:**
- Read-heavy atau write-heavy pattern?
- Developer requirements: apa specific functionality?
- Data volume: growth rate per bulan?
- Query patterns: simple lookups atau complex aggregations?
- Data retention: berapa lama disimpan?
- Access frequency: real-time atau batch analysis?

#### 2.2 Technical Comparison

| Aspek | PostgreSQL | MongoDB |
|-------|-----------|---------|
| **Cost** | ✅ Standard (baseline) | ❌ 3x lebih mahal |
| **Schema flexibility** | Schema-strict | Schema-flexible |
| **Complex queries** | Mature JOINs + aggregation | Aggregation pipeline |
| **Historical data** | Excellent (partitioning) | Overkill untuk read-heavy |
| **Indexing optimization** | Highly optimizable | Limited nested doc indexing |
| **Developer ramp-up** | Higher learning curve | More intuitive initial |

#### 2.3 Recommendation
**PostgreSQL recommended untuk historical order data** karena:
- ✅ Cost-effective untuk data yang jarang update
- ✅ Proven untuk large historical datasets
- ✅ Better query optimization untuk analytics
- ✅ 3x cost savings untuk scale besar
- ✅ Team familiar dengan technology stack

#### 2.4 Alternative: Hybrid Approach
- PostgreSQL: primary historical data storage
- Redis/Cache: untuk hot queries
- MongoDB: hanya jika specific flexible schema use case exists

### Action Items
- [ ] Clarify developer-specific requirements yang push MongoDB
- [ ] Quantify cost impact dengan current data size
- [ ] Document decision rationale untuk stakeholders
- [ ] Create migration/optimization plan if PostgreSQL approved

---

## 3. Load Testing Initiative: Production Capacity Assessment

### Objective
Understand system capacity ceiling untuk MyBB di production environment dengan real data dan production specifications di staging environment.

### Load Test Strategy

#### 3.1 Test Scenarios

**Baseline Scenarios:**
1. **Normal Daily Load** - typical RPS saat peak hours (jam rush pagi/sore)
2. **Peak Load** - maksimal expected RPS saat surge (event, promo)
3. **Stress Test** - beyond peak until breaking point
4. **Sustained Load** - durasi maintenance di peak tanpa degradation

**Example metrics (to be confirmed):**
- Normal: 1,000 RPS
- Peak: 5,000 RPS  
- Stress: 10,000+ RPS (breaking point)

#### 3.2 Critical Paths to Test

**Primary flows (MRG + UPG):**
1. Booking Flow: /create-order → payment → confirmation
2. Payment Processing: validation → gateway → reconciliation
3. Order Status: polling/webhooks untuk real-time updates
4. User Queries: get orders, order history, payment status

**Secondary flows:**
- Concurrent user sessions
- Database query load (order lookup, payment check)
- Cache hit rates (Redis stress)
- Notification system under load

#### 3.3 Key Metrics to Measure

**Performance Metrics:**
- Response time: P50, P95, P99 latency
- Throughput: RPS capacity before degradation
- Error rate: % request failures at each load level
- Critical success rate: payment success %, booking confirmation %

**Infrastructure Metrics:**
- CPU utilization (target: 70% safe, 85% concerning)
- Memory usage trends
- Disk I/O (especially PostgreSQL query load)
- Network bandwidth usage
- Database connection pool status

**Business Metrics:**
- Payment success rate under load
- Order completion rate
- Booking confirmation latency

#### 3.4 Test Data Preparation

**Real data considerations:**
- Production-like dataset size (10M+ historical orders, 1M+ users)
- Realistic data distribution dan user behavior patterns
- Edge cases: power users, large transactions
- Sensitive data anonymization

#### 3.5 Tools & Environment

**Load testing tools options:**
- Locust (Python, complex scenarios)
- k6 (JavaScript, cloud-native, distributed)
- JMeter (traditional, mature)
- Artillery (JavaScript, simple)

**Staging setup:**
- Mirror production topology (K8s, service count)
- Real database replicas or realistic schema
- Isolated environment (no production impact)
- Monitoring tools ready

### Execution Timeline

| Phase | Duration | Activities |
|-------|----------|-----------|
| **Phase 1: Preparation** | 1-2 minggu | Define scenarios, prepare data, setup staging, develop test scripts |
| **Phase 2: Execution** | 1 minggu | Run baseline, gradual load increase, document results |
| **Phase 3: Analysis** | 3-5 hari | Bottleneck identification, breaking point analysis, resource metrics |
| **Phase 4: Recommendations** | Concurrent | Scaling strategy, optimization opportunities, capacity planning |
| **Total Duration** | ~3-4 minggu | End-to-end comprehensive assessment |

### Action Items

#### 3.6 Information Needed
- [ ] Current peak RPS di production?
- [ ] Growth trend per bulan?
- [ ] Target RPS untuk next 6-12 bulan?
- [ ] Staging environment readiness (spec match production)?
- [ ] Database replica availability untuk testing?
- [ ] Monitoring tools status?
- [ ] Known bottlenecks atau previous incidents?

#### 3.7 Deliverables
- Clear understanding of system capacity ceiling
- Bottleneck identification dengan remediation plan
- Data-driven scaling recommendations
- Risk mitigation strategy untuk production stability
- Capacity planning untuk growth 6-12 bulan ke depan

### Business Value to VP
- **Timeline:** ~3-4 minggu untuk comprehensive assessment
- **Cost:** Minimal (leverage existing staging environment)
- **Prevention:** Avoid outages, ensure reliability, data-driven decisions
- **Growth enablement:** Clear roadmap untuk scale-up dengan confidence

---

## Summary of Action Items

### Immediate (This Week)
- [ ] Business improvement: Schedule detailed failure analysis session dengan MRG/UPG teams
- [ ] Database assessment: Gather developer requirements untuk MongoDB request
- [ ] Load testing: Confirm current production baseline metrics (peak RPS, growth rate)

### Short-term (1-2 Weeks)
- [ ] Business improvement: Document current failure points dan quick wins
- [ ] Database assessment: Quantify cost impact, prepare recommendation
- [ ] Load testing: Finalize test scenarios, prepare test data, setup staging environment

### Medium-term (2-4 Weeks)
- [ ] All initiatives: Execute testing dan analysis phases
- [ ] Prepare findings untuk VP review

---

## Key Takeaways

1. **Business Improvement:** Data-driven approach untuk reduce failures dan simplify processes - kuantifikasi impact sebelum implementation

2. **Database Decision:** PostgreSQL recommended untuk historical data dengan 3x cost savings - aligns dengan financial constraints

3. **Load Testing:** Critical untuk understand capacity dan plan growth - 3-4 minggu timeline, minimal cost, high value untuk operational stability

---

## Status Tracking
- **Created:** 27 Januari 2026
- **Last Updated:** 27 Januari 2026
- **Owner:** Lukmanul Hakim
- **Related Projects:** MRG, UPG, MyBB
