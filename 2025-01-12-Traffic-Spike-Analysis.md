---
date: '2025-01-12'
tags:
  - incident
  - performance
  - payment
status: investigating
severity: high
---
# Traffic Spike Analysis - Order Gantung Issue

**Tanggal Kejadian**: 12 Januari 2025  
**Peak Traffic**: 77k concurrent users (normal: <30k users)  
**Impact**: Order gantung massal dengan payment tidak tercatat di HO

## Problem Statement

### Symptoms
1. **Order Status Issue**
   - Order tidak selesai hingga status COMPLETE
   - Order stuck di status ACTIVE
   
2. **Payment Recording Issue**
   - Payment tidak tercatat di Head Office
   - Kemungkinan payment sudah di-process di UPG tapi tidak sampai ke HO
   - Potensi data inconsistency antara UPG dan HO

## Root Cause Hypothesis

### 1. Database Connection Pool Exhaustion (HIGH Priority)
**Reasoning**: 77k user = 2.5x normal load, kemungkinan connection pool tidak cukup  
**Impact**: Order processing delay → timeout → order stuck di ACTIVE

**Validation**:
- [ ] Check database connection pool metrics
- [ ] Review connection pool configuration
- [ ] Analyze connection wait time

### 2. Payment Callback Failure (MEDIUM-HIGH Priority)
**Reasoning**: Payment processed di UPG tapi callback ke HO gagal karena overload  
**Impact**: Payment tidak tercatat di HO meski sudah di-charge

**Validation**:
- [ ] Check callback queue/retry mechanism
- [ ] Verify callback endpoint availability
- [ ] Review callback logs untuk error rate

### 3. Circuit Breaker Activation (MEDIUM Priority)
**Reasoning**: Under high load, circuit breaker mungkin open, blocking requests  
**Impact**: Requests ditolak, order tidak bisa complete

**Validation**:
- [ ] Check circuit breaker metrics
- [ ] Review circuit breaker threshold configuration
- [ ] Analyze service dependency health

### 4. Timeout Configuration Issue (MEDIUM Priority)
**Reasoning**: Default timeout terlalu pendek untuk handle increased load  
**Impact**: Premature timeout → incomplete transactions

**Validation**:
- [ ] Review timeout configuration di semua service
- [ ] Check actual response time vs timeout setting
- [ ] Analyze timeout error logs

### 5. Race Condition di Payment Flow (LOW-MEDIUM Priority)
**Reasoning**: High concurrency bisa trigger race condition yang jarang terjadi di normal load  
**Impact**: Payment state inconsistency

**Validation**:
- [ ] Review payment state machine implementation
- [ ] Check for proper locking mechanism
- [ ] Analyze concurrent transaction patterns

## Investigation Checklist

### Phase 1: Data Collection (URGENT)
- [ ] Collect metrics dari Huawei Cloud monitoring (12 Jan, full day)
- [ ] Export application logs dari MRG, UPG, Order Orchestrator
- [ ] Query database untuk list order gantung dengan order_id
- [ ] Collect payment transaction records dari UPG
- [ ] Check HO records untuk missing payments

### Phase 2: Analysis (HIGH)
- [ ] Analyze logs untuk error patterns
- [ ] Correlate order_id antara MRG, UPG, dan HO
- [ ] Identify common characteristics dari order gantung
- [ ] Map timeline: traffic spike → error occurrence → service degradation

### Phase 3: Root Cause Identification (HIGH)
- [ ] Validate each hypothesis dengan data
- [ ] Identify primary root cause
- [ ] Document contributing factors
- [ ] Estimate impact scope (financial, user, reputation)

### Phase 4: Remediation Plan (MEDIUM)
- [ ] Short-term fix untuk recover order gantung
- [ ] Payment reconciliation process
- [ ] Customer communication plan

### Phase 5: Prevention (MEDIUM)
- [ ] Load testing dengan 100k+ users
- [ ] Capacity planning update
- [ ] Auto-scaling configuration review
- [ ] Circuit breaker threshold tuning
- [ ] Database connection pool optimization

## Technical Investigation Areas

### MRG Service
- [ ] Connection pool exhaustion (database)
- [ ] Goroutine leak
- [ ] Memory pressure / GC pauses
- [ ] Network timeout ke downstream services
- [ ] Circuit breaker activation

### UPG Service
- [ ] Payment gateway timeout (vendor side)
- [ ] Callback mechanism failure
- [ ] Race condition di payment processing
- [ ] Database write bottleneck
- [ ] Exponential backoff retry mechanism status

### Order Orchestrator
- [ ] SPOF yang sudah teridentifikasi sebelumnya
- [ ] State machine transition failure
- [ ] Saga pattern compensation tidak trigger
- [ ] Database sequence/ID generation issue

### Database
- [ ] Connection pool: max vs active connections
- [ ] Slow query log analysis
- [ ] Lock contention (deadlocks, wait events)
- [ ] Transaction rollback rate
- [ ] CPU utilization, IOPS metrics

### Integration Points
- [ ] MRG → UPG: timeout, retry mechanism
- [ ] UPG → Payment Gateway: vendor response time, error rate
- [ ] UPG → HO: API call success rate, network latency

## Go Microservices Specific Checks

```go
// 1. Check goroutine leak
runtime.NumGoroutine()

// 2. Memory stats
runtime.ReadMemStats()

// 3. GC pauses
// Check GC logs untuk pause time during spike

// 4. HTTP client timeout
// Review http.Client configuration di semua services
```

## Database Queries for Investigation

```sql
-- Check active connections
SELECT count(*) FROM pg_stat_activity;

-- Check locks
SELECT * FROM pg_locks WHERE NOT granted;

-- Check slow queries
SELECT * FROM pg_stat_statements 
ORDER BY total_time DESC LIMIT 20;

-- Payment reconciliation query
SELECT 
    COUNT(*) as total_payments,
    payment_status,
    created_at
FROM payments
WHERE DATE(created_at) = '2025-01-12'
GROUP BY payment_status, DATE(created_at);
```

## Data Consistency Questions

- [ ] Berapa total order gantung?
- [ ] Berapa yang sudah bayar tapi stuck?
- [ ] Berapa yang belum bayar dan stuck?
- [ ] Apakah ada pattern tertentu (service type, payment method, region)?

## Use mcpmrg for Detective Work

- [ ] Search error logs: "timeout", "connection", "circuit", "payment"
- [ ] Analyze error frequency by time window
- [ ] Correlate errors dengan order_id yang gantung
- [ ] Check stack traces yang indicate root cause

## Questions for Stakeholders

**Product/Business**:
1. Apakah ada campaign/promo di tanggal 12 Jan?
2. Apakah ada partnership baru yang drive traffic?
3. Berapa estimated financial impact?

**Operations**:
1. Apakah ada maintenance/deployment di tanggal tersebut?
2. Apakah monitoring alert trigger?
3. Berapa lama response time untuk incident?

**Engineering**:
1. Apakah ada recent changes di MRG/UPG sebelum tanggal 12?
2. Apakah pernah load testing untuk >50k users?
3. Apakah ada known issues yang relate?

## Next Steps

**Immediate (Today)**:
- Setup war room dengan tim MRG & UPG
- Collect initial metrics dan logs
- Identify scope: total affected orders

**Short-term (This Week)**:
- Complete root cause analysis
- Design remediation plan
- Execute payment reconciliation

**Long-term (This Month)**:
- Implement preventive measures
- Conduct load testing
- Update capacity planning
- Create runbook untuk similar incidents

---
Status: Investigation in progress  
Owner: Lukmanul Hakim  
Last Updated: 2025-01-19
