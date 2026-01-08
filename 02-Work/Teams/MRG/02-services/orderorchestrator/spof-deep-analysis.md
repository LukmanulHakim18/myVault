---
tags:
  - orderorchestrator
  - spof
  - critical
  - incident-response
  - architecture
  - reliability
type: deep-analysis
status: active
severity: critical
created: '2025-01-08'
updated: '2025-01-08'
---
# Order Orchestrator - Deep SPOF Analysis

**Service**: [[README|Order Orchestrator]]  
**Analysis Date**: 2025-01-08  
**Severity**: 🔴 **CRITICAL SPOF**  
**Business Impact**: $$$$ (Revenue-Blocking)

---

## 🎯 Executive Summary

Order Orchestrator adalah **single most critical service** dalam arsitektur MRG dengan **ZERO alternative paths**. Service ini mengkoordinasi 100% dari order lifecycle dan kegagalannya menyebabkan **complete booking freeze** - tidak ada workaround atau fallback mechanism.

**Key Metrics**:
- **Criticality Score**: 10/10
- **Blast Radius**: ALL booking channels (App, Web, Corporate)
- **Revenue Impact**: ~$15,000 per hour downtime (estimated)
- **Customer Impact**: 100% users cannot create bookings
- **Alternative Paths**: NONE

---

## 🏗️ Architecture Analysis

### Central Orchestration Point

```
┌─────────────────────────────────────────────────────────┐
│                  Order Orchestrator                      │
│                   (SINGLE POINT)                         │
├─────────────────────────────────────────────────────────┤
│                                                          │
│  ┌───────────┐  ┌───────────┐  ┌──────────────┐       │
│  │   Cart    │  │  Payment  │  │     Saga     │       │
│  │  Manager  │  │   State   │  │   Process    │       │
│  │           │  │  Machine  │  │              │       │
│  └───────────┘  └───────────┘  └──────────────┘       │
│                                                          │
└─────────────────────────────────────────────────────────┘
         │                │                │
         ▼                ▼                ▼
   ┌─────────┐    ┌──────────────┐  ┌───────────┐
   │ Session │    │   Payment    │  │   Taxi    │
   │ Manager │    │  Processor   │  │  Partner  │
   └─────────┘    └──────────────┘  │  Gateway  │
                                     └───────────┘
```

**Critical Characteristics**:
1. **Stateful Operations**: Shopping cart dengan TTL 15 menit
2. **Synchronous Flows**: Blocking calls ke 7+ downstream services
3. **Complex State Machine**: Payment states dengan strict transitions
4. **Distributed Transaction**: Saga pattern untuk multi-service bookings
5. **No Bypass**: Semua channels (App, Web, Corporate) depend on this

---

## 💥 Complete Failure Scenario Matrix

### Scenario 1: Service Pod Crash

**Trigger**: OOMKill, panic, segfault, deployment

```
┌──────────────────────────────────────────────┐
│ Order Orchestrator Pods: 3 running → 0      │
└──────────────────────────────────────────────┘
                    ↓
┌──────────────────────────────────────────────┐
│ IMMEDIATE IMPACT (T+0s)                      │
├──────────────────────────────────────────────┤
│ • CreateCart: FAIL ❌                         │
│ • AddCartItem: FAIL ❌                        │
│ • CreateOrder: FAIL ❌                        │
│ • Payment Callbacks: DROPPED ❌              │
└──────────────────────────────────────────────┘
                    ↓
┌──────────────────────────────────────────────┐
│ CASCADING IMPACT (T+30s)                     │
├──────────────────────────────────────────────┤
│ • MyBB App: Error on checkout                │
│ • Web Reservation: 503 Service Unavailable   │
│ • In-flight payments: Stuck in WAITING       │
│ • Active carts: Lost (Redis keys with TTL)   │
└──────────────────────────────────────────────┘
                    ↓
┌──────────────────────────────────────────────┐
│ BUSINESS IMPACT (T+5min)                     │
├──────────────────────────────────────────────┤
│ • ZERO new bookings                          │
│ • Customer complaints spike                  │
│ • Call center overwhelmed                    │
│ • Revenue loss: ~$1,250 per 5 minutes        │
└──────────────────────────────────────────────┘
```

**Recovery Time**: 
- Pod restart: 30-60 seconds (if image cached)
- Service ready: +30 seconds (health checks)
- **Total**: 1-2 minutes

---

### Scenario 2: Database Connection Failure

**Trigger**: PostgreSQL down, network partition, connection pool exhausted

```
PostgreSQL UNAVAILABLE
        ↓
Order Orchestrator CANNOT:
├─ Read existing bookings
├─ Create new bookings
├─ Update payment status
├─ Query order history
└─ Execute Saga compensation
        ↓
RESULT: Service UP but BROKEN
        ├─ Health check: GREEN (misleading)
        ├─ All operations: FAIL
        └─ Error rate: 100%
```

**Affected Operations**:
- `CreateOrder`: Cannot insert booking record → BLOCKED ❌
- `GetBookingDetail`: Cannot query → FAIL ❌
- `CallbackPaymentInformation`: Cannot update payment → STUCK ❌
- `Saga Execution`: Cannot update order items → COMPENSATION BROKEN ❌

**Data Consistency Risks**:
- Payment success webhook received but not processed
- Refunds not triggered due to saga failure
- Order status inconsistent across systems

**Recovery Time**:
- Database recovery: 5-30 minutes (depending on issue)
- **Data reconciliation**: Manual effort, hours to days

---

### Scenario 3: Redis Failure (Cart Cache)

**Trigger**: Redis down, memory limit reached, network issue

```
Redis DOWN
    ↓
Cart Operations:
├─ CreateCart: FAIL (cannot store) ❌
├─ AddCartItem: FAIL (cart not found) ❌
├─ GetCart: FAIL (cannot retrieve) ❌
└─ CreateOrder: PARTIAL (can proceed if cart_id valid)
    ↓
Session Data Lost:
├─ Active carts: GONE (15 min TTL)
├─ Session tracking: LOST
└─ Cart items: Cannot retrieve
    ↓
User Experience:
├─ "Cart not found" errors
├─ Must recreate cart from scratch
└─ Frustration & abandonment
```

**Impact Level**: HIGH (not CRITICAL)
- Existing bookings: Continue ✓
- New cart creation: Blocked ❌
- Carts created before Redis failure: Lost ❌

**Workaround**: Use PostgreSQL for cart storage (slower, but works)

**Recovery Time**: 
- Redis restart: 1-5 minutes
- Cart data: LOST (ephemeral)
- Users must recreate: Manual

---

### Scenario 4: Payment Processor Unavailable

**Trigger**: UPG service down, network timeout

```
Payment Processor DOWN
        ↓
Order Orchestrator FLOW:
CreateOrder → Create Booking ✓
           → Request Payment Link ❌ TIMEOUT
           → PARTIAL SUCCESS (booking created, no payment link)
        ↓
User Experience:
├─ Booking ID: Generated
├─ Payment URL: null or error
└─ Order status: CREATED (stuck)
        ↓
Business Impact:
├─ Orders created but unpayable
├─ Customer cannot proceed
└─ Manual intervention required
```

**Affected Operations**:
- `CreateOrder`: Partial success (booking exists, payment link missing)
- `CallbackPaymentInformation`: Not received (Payment Processor down)
- Saga execution: Never triggered (payment never succeeds)

**Mitigation**:
- Timeout settings: 5 seconds (prevent long hangs)
- Fallback: Return booking_id with "payment link unavailable" message
- Retry mechanism: Background job to request payment link

**Recovery**:
- **Automatic**: Payment Processor recovery + retry job
- **Manual**: Customer service generates payment link manually

---

### Scenario 5: Saga Compensation Failure

**Trigger**: Downstream service (TPG, Rent) fails after payment success

```
Payment SUCCESS
    ↓
Saga Execution:
├─ Book Taxi #1: SUCCESS ✓
├─ Book Taxi #2: FAIL ❌
└─ Compensation Required
    ↓
Compensation Process:
├─ Cancel Taxi #1: SUCCESS ✓
├─ Request Refund: TIMEOUT ❌ (Payment Processor unreachable)
└─ STUCK STATE
    ↓
Result:
├─ Customer charged
├─ Services partially booked
├─ Refund not processed
└─ Data inconsistency
```

**Critical Failure Points**:
1. Compensation timeout (saga abandoned)
2. Partial compensation (some services cancelled, others not)
3. Refund service unavailable (customer charged, no refund)

**Data Consistency Issues**:
```
Order Status: FAILED
├─ Payment: PAID (money collected)
├─ Order Items: MIXED (some cancelled, some active)
└─ Refund Status: PENDING (never processed)
```

**Recovery**:
- **Automatic Retry**: RabbitMQ queue for retry (if configured)
- **Manual Intervention**: Required if retry exhausted
- **Timeframe**: Can take hours to days if complex

---

### Scenario 6: Message Broker Failure (RabbitMQ)

**Trigger**: RabbitMQ down, queue full, consumer lag

```
RabbitMQ DOWN
    ↓
Event Publishing:
├─ Booking events: LOST ❌
├─ Email notifications: NOT SENT ❌
└─ Saga retry: BLOCKED ❌
    ↓
Immediate Impact:
├─ Order creation: WORKS ✓ (sync operation)
├─ Payment callback: WORKS ✓ (direct HTTP)
└─ Saga execution: WORKS ✓ (sync)
    ↓
Delayed Impact:
├─ Retry mechanism: BROKEN (cannot republish)
├─ Event consumers: STARVED (no events)
└─ Async workflows: PAUSED
```

**Impact Level**: MEDIUM (async failures)
- Core booking flow: Continues ✓
- Event-driven workflows: Broken ❌
- Retry on failure: Blocked ❌

**Affected Components**:
- Notification Center: No booking events → no emails
- Analytics: No events for tracking
- Saga retry: Cannot republish for compensation

---

### Scenario 7: Cascading Failure (Multiple Dependencies)

**Trigger**: Infrastructure issue affecting multiple services

```
Network Partition or Cluster Failure
        ↓
┌─────────────────────────────────────┐
│ Order Orchestrator: UP              │
│ ├─ Session Manager: DOWN ❌         │
│ ├─ Payment Processor: DOWN ❌       │
│ ├─ Taxi Partner Gateway: DOWN ❌    │
│ └─ Config Service: DOWN ❌          │
└─────────────────────────────────────┘
        ↓
COMPLETE SYSTEM FAILURE
├─ Cannot price carts (Session Manager)
├─ Cannot create payments (Payment Processor)
├─ Cannot book services (TPG)
└─ Cannot load config (Config Service)
        ↓
Result: ZERO functionality
```

**Blast Radius**: MAXIMUM
- All downstream services unavailable
- Order Orchestrator operational but useless
- Requires full infrastructure recovery

**Recovery Strategy**:
1. Identify root cause (network, K8s cluster, cloud provider)
2. Restore infrastructure
3. Services auto-recover (if healthy)
4. Verify end-to-end flow

---

## 📊 Dependency Failure Impact Matrix

| Dependency | Criticality | Impact if Down | Workaround Available | Recovery Time |
|------------|-------------|----------------|----------------------|---------------|
| **PostgreSQL** | 🔴 CRITICAL | Complete failure | ❌ None | 5-30 min |
| **Redis** | 🟠 HIGH | Cart operations blocked | ⚠️ Fallback to DB | 1-5 min |
| **Payment Processor** | 🔴 CRITICAL | Cannot generate payment link | ❌ None | 5-15 min |
| **Session Manager** | 🔴 CRITICAL | Cannot price carts | ❌ None | 5-15 min |
| **Taxi Partner Gateway** | 🔴 CRITICAL | Cannot book taxi | ❌ None | 5-15 min |
| **Promo Gateway** | 🟡 MEDIUM | Promo validation fails | ✅ Skip promo | 5-15 min |
| **Config Service** | 🟡 MEDIUM | Use cached config | ✅ Default config | 5-15 min |
| **Notification Service** | 🟢 LOW | No notifications sent | ✅ Async retry | 15-30 min |
| **RabbitMQ** | 🟡 MEDIUM | Async workflows broken | ⚠️ Degraded | 5-15 min |
| **Google PubSub** | 🟢 LOW | Some events lost | ✅ Optional | 15-30 min |

---

## 🛡️ Current Mitigation Measures

### Infrastructure Level

✅ **Kubernetes Deployment**
```yaml
replicas: 3
strategy:
  type: RollingUpdate
  rollingUpdate:
    maxUnavailable: 1
    maxSurge: 1
```
- **Benefit**: Zero downtime deployments
- **Limitation**: All pods can still fail simultaneously (OOM, panic)

✅ **Health Checks**
```yaml
livenessProbe:
  httpGet:
    path: /health
    port: 8000
  initialDelaySeconds: 30
  periodSeconds: 10
readinessProbe:
  httpGet:
    path: /ready
    port: 8000
  initialDelaySeconds: 10
  periodSeconds: 5
```
- **Benefit**: Auto-restart unhealthy pods
- **Limitation**: Doesn't check downstream dependencies

✅ **Resource Limits**
```yaml
resources:
  requests:
    memory: "512Mi"
    cpu: "500m"
  limits:
    memory: "1Gi"
    cpu: "1000m"
```
- **Benefit**: Prevents resource starvation
- **Limitation**: Can cause OOMKill under high load

### Application Level

✅ **Connection Pooling**
```go
// Database
MaxIdleConns: 10
MaxOpenConns: 100
ConnMaxLifetime: 1 hour

// Redis
PoolSize: 50
MinIdleConns: 10
```
- **Benefit**: Efficient connection reuse
- **Limitation**: Exhausted pool = blocked requests

✅ **Timeouts**
```go
// gRPC client
Timeout: 5 seconds

// HTTP client
Timeout: 10 seconds
```
- **Benefit**: Prevents hanging requests
- **Limitation**: Fast failures but no retry

✅ **Logging & Tracing**
```go
// Elastic APM
Spans for all operations
Transaction tracing
Error tracking
```
- **Benefit**: Detailed observability
- **Limitation**: Reactive (doesn't prevent failures)

---

## 🚨 Critical Gaps

### ❌ No Circuit Breaker

**Problem**: 
```go
// Current: Direct call without protection
resp, err := s.Repo.PaymentProcessor.RequestPaymentLink(ctx, req)
if err != nil {
    return nil, err // Propagates error directly
}
```

**Impact**:
- Downstream service slow → All requests slow
- Downstream service down → All requests fail
- No automatic recovery mechanism

**Solution Needed**:
```go
// With circuit breaker
circuit := circuitbreaker.NewCircuitBreaker(
    MaxFailures: 5,
    Timeout: 10 * time.Second,
    ResetInterval: 60 * time.Second,
)

resp, err := circuit.Call(func() (interface{}, error) {
    return s.Repo.PaymentProcessor.RequestPaymentLink(ctx, req)
})

if circuit.State() == circuitbreaker.Open {
    // Return cached response or fallback
    return s.fallbackPaymentLink(ctx, req)
}
```

---

### ❌ No Graceful Degradation

**Problem**: All-or-nothing approach

**Impact**:
- Payment Processor down → Cannot create order (even though booking data valid)
- Promo Gateway down → Cannot create order (even if no promo used)
- Notification down → Order creation blocks (even though notification is async)

**Solution Needed**:
```go
// Feature flags for degradation
if paymentProcessor.IsAvailable() {
    payment, err := s.requestPaymentLink(ctx, booking)
    if err != nil {
        // Soft failure: Create booking without payment link
        log.Warn("Payment link unavailable, creating booking anyway")
        booking.PaymentStatus = "PAYMENT_LINK_PENDING"
    }
}

if promoGateway.IsAvailable() {
    s.applyPromo(ctx, booking)
} else {
    // Degraded: Skip promo, customer can apply later
    log.Warn("Promo validation skipped due to service unavailability")
}
```

---

### ❌ No Async Processing for Non-Critical Paths

**Problem**: Synchronous flow for everything

```
CreateOrder Flow (Current - Synchronous):
1. Create booking (DB) ← MUST SUCCEED
2. Request payment link (Payment Processor) ← BLOCKS if slow
3. Send notifications (Notification Center) ← BLOCKS if slow
4. Publish events (RabbitMQ) ← BLOCKS if down
```

**Impact**:
- Any step failure → Entire order creation fails
- Slow downstream → Slow user experience

**Solution Needed**:
```
CreateOrder Flow (Improved - Async where possible):
1. Create booking (DB) ← MUST SUCCEED (sync)
2. Request payment link (Payment Processor) ← MUST SUCCEED (sync)
3. Queue notification job ← ASYNC ✓
4. Queue event publishing ← ASYNC ✓
```

---

### ❌ No Database Read-Write Separation

**Problem**: All queries use single master DB connection

```go
// Current: All queries to master
func (r *BookingRepo) GetBookingDetail(ctx, bookingID) {
    return r.DB.QueryRow(ctx, sql, bookingID) // Master DB
}
```

**Impact**:
- High read load → Affects write performance
- Master DB slow → All operations slow
- No read scaling

**Solution Needed**:
```go
// Read from replica
func (r *BookingRepo) GetBookingDetail(ctx, bookingID) {
    return r.DBReplica.QueryRow(ctx, sql, bookingID)
}

// Write to master
func (r *BookingRepo) CreateBooking(ctx, booking) {
    return r.DBMaster.Exec(ctx, sql, booking)
}
```

---

### ❌ No Request Rate Limiting

**Problem**: No protection against traffic spikes

**Impact**:
- Sudden load → Service overwhelmed
- All pods OOMKill
- Complete service failure

**Solution Needed**:
```go
// Rate limiter middleware
rateLimiter := ratelimit.New(
    PerSecond: 100,
    Burst: 200,
)

// Apply to high-traffic endpoints
func (t *Transport) CreateOrder(ctx, req) {
    if err := rateLimiter.Wait(ctx); err != nil {
        return nil, status.Error(codes.ResourceExhausted, "rate limit exceeded")
    }
    return t.usecase.CreateOrder(ctx, req)
}
```

---

## 🎯 Recommended Improvements

### Priority 1: Immediate (Week 1-2)

#### 1.1 Enhanced Health Checks with Dependency Validation

**Current**:
```go
func HealthCheck() bool {
    return true // Only checks if service is up
}
```

**Improved**:
```go
func HealthCheck() HealthStatus {
    status := HealthStatus{Healthy: true, Dependencies: map[string]bool{}}
    
    // Check critical dependencies
    status.Dependencies["postgresql"] = s.checkPostgreSQL()
    status.Dependencies["redis"] = s.checkRedis()
    status.Dependencies["payment_processor"] = s.checkPaymentProcessor()
    status.Dependencies["session_manager"] = s.checkSessionManager()
    
    // Overall health
    status.Healthy = allHealthy(status.Dependencies)
    return status
}
```

**Benefit**: K8s can restart pod if dependencies unhealthy

---

#### 1.2 Circuit Breaker for Critical Dependencies

**Implementation**:
```go
// Add to repository initialization
func NewRepository() *Repository {
    return &Repository{
        PaymentProcessor: circuitbreaker.Wrap(
            paymentprocessorclient.New(),
            circuitbreaker.Config{
                MaxFailures: 5,
                Timeout: 5 * time.Second,
                ResetInterval: 60 * time.Second,
            },
        ),
        // ... other repos
    }
}
```

**Benefit**: Prevent cascading failures, fail-fast behavior

---

#### 1.3 Timeout Configuration Review

**Audit all timeouts**:
```go
// Current timeouts
PaymentProcessor: 5s
SessionManager: 5s
TaxiPartnerGateway: 10s
Notification: 5s

// Recommended (based on SLA analysis)
PaymentProcessor: 3s (fast fail, retry elsewhere)
SessionManager: 5s (pricing must be fast)
TaxiPartnerGateway: 10s (external system, tolerate latency)
Notification: 2s (async, fail fast)
```

**Benefit**: Reduce blocking time, improve responsiveness

---

### Priority 2: Short-term (Month 1-2)

#### 2.1 Async Event Publishing

**Separate critical path from nice-to-have**:
```go
func (u *UseCase) CreateOrder(ctx, req) (*Order, error) {
    // CRITICAL PATH (synchronous)
    booking := u.createBooking(ctx, req)
    paymentLink := u.requestPaymentLink(ctx, booking)
    
    // NON-CRITICAL (async via worker queue)
    u.queueNotificationJob(booking)
    u.queueEventPublishing(booking)
    u.queueAnalyticsTracking(booking)
    
    return &Order{
        BookingID: booking.ID,
        PaymentURL: paymentLink,
    }, nil
}
```

**Benefit**: Faster response time, reduced failure points

---

#### 2.2 Cart Persistence Fallback

**Implement PostgreSQL fallback for Redis**:
```go
func (r *CartRepo) GetCart(ctx, cartID) (*Cart, error) {
    // Try Redis first (fast)
    cart, err := r.Redis.Get(ctx, cartID)
    if err == nil {
        return cart, nil
    }
    
    // Fallback to PostgreSQL (slower but reliable)
    log.Warn("Redis unavailable, falling back to PostgreSQL")
    return r.DB.GetCart(ctx, cartID)
}
```

**Benefit**: Cart operations continue even if Redis down

---

#### 2.3 Database Read Replica

**Setup read-write separation**:
```yaml
# K8s ConfigMap
DB_HOST_MASTER: "postgres-master.default.svc"
DB_HOST_REPLICA: "postgres-replica.default.svc"
```

```go
// Repository initialization
DB: DB{
    Master: connectPostgreSQL(cfg.DBHostMaster),
    Replica: connectPostgreSQL(cfg.DBHostReplica),
}

// Use cases
func GetBookingDetail() {
    return r.DB.Replica.Query(...) // Read from replica
}

func CreateBooking() {
    return r.DB.Master.Exec(...) // Write to master
}
```

**Benefit**: Scale reads independently, reduce master load

---

#### 2.4 Request Rate Limiting

**Implement per-endpoint rate limits**:
```go
// Rate limiter middleware
rateLimiters := map[string]*ratelimit.Limiter{
    "CreateCart": ratelimit.New(200), // 200 req/s
    "AddCartItem": ratelimit.New(500), // 500 req/s
    "CreateOrder": ratelimit.New(100), // 100 req/s (expensive)
    "GetBookingDetail": ratelimit.New(1000), // 1000 req/s (read)
}
```

**Benefit**: Prevent overload, protect from traffic spikes

---

### Priority 3: Medium-term (Month 3-6)

#### 3.1 Saga Timeout & Recovery

**Implement saga timeout with manual intervention**:
```go
type SagaConfig struct {
    Timeout: 5 * time.Minute
    MaxRetries: 3
    ManualInterventionQueue: "saga-failed"
}

func (s *SagaProcess) Execute(ctx) error {
    ctx, cancel := context.WithTimeout(ctx, s.Config.Timeout)
    defer cancel()
    
    if err := s.executeSteps(ctx); err != nil {
        if err == context.DeadlineExceeded {
            // Saga timeout - push to manual queue
            s.pushToManualQueue(s.booking)
            return ErrSagaTimeout
        }
        // Normal compensation
        return s.compensate(ctx)
    }
    return nil
}
```

**Benefit**: Prevent stuck sagas, enable manual resolution

---

#### 3.2 Observability Dashboard

**Create dedicated Order Orchestrator dashboard**:
```
Metrics to track:
- Order creation rate (per minute)
- Order success rate (%)
- Payment link generation latency (P50, P95, P99)
- Saga execution time (P50, P95, P99)
- Saga compensation rate (%)
- Cart cache hit rate (%)
- Dependency health status (green/yellow/red)
- Circuit breaker states (open/half-open/closed)
```

**Benefit**: Proactive issue detection, faster MTTR

---

#### 3.3 Chaos Engineering

**Regular failure injection tests**:
```
Test Scenarios:
1. Kill 1 pod during high traffic
2. Network partition (isolate from PostgreSQL)
3. Redis failure simulation
4. Payment Processor 10s latency injection
5. Concurrent Saga execution under load
```

**Benefit**: Validate resilience, identify weaknesses

---

### Priority 4: Long-term (6-12 months)

#### 4.1 Event-Driven Architecture

**Decouple via event bus**:
```
Current (Synchronous):
App → OrderOrchestrator → [7 services synchronously]

Proposed (Event-Driven):
App → OrderOrchestrator → Event Bus
                             ├─→ Payment Service
                             ├─→ Saga Coordinator
                             ├─→ Notification Service
                             └─→ Analytics Service
```

**Benefit**: Reduce coupling, improve resilience

---

#### 4.2 CQRS Pattern

**Separate command and query responsibilities**:
```
Command Side (Write):
OrderOrchestrator → PostgreSQL (Master)

Query Side (Read):
OrderQuery Service → PostgreSQL (Replica)
                  → Elasticsearch (search)
                  → Redis (cache)
```

**Benefit**: Scale reads and writes independently

---

#### 4.3 Multi-Region Deployment

**Geographic redundancy**:
```
Region 1 (Jakarta):
├─ OrderOrchestrator (active)
├─ PostgreSQL (master)
└─ Dependencies (active)

Region 2 (Singapore):
├─ OrderOrchestrator (standby)
├─ PostgreSQL (replica)
└─ Dependencies (standby)

Failover: DNS switch + DB promotion
```

**Benefit**: Disaster recovery, near-zero downtime

---

## 📊 Monitoring & Alerting

### Critical Metrics

#### Service Health

```promql
# Service availability
up{job="orderorchestrator"} == 0

# Alert: Service Down
Alert if: up == 0 for 1 minute
Severity: P0 (CRITICAL)
Action: Page on-call immediately
```

#### Order Creation Rate

```promql
# Orders created per minute
rate(orderorchestrator_orders_created_total[1m])

# Alert: Order Creation Dropped to Zero
Alert if: rate == 0 for 5 minutes
Severity: P0 (CRITICAL)
Action: Investigate immediately
```

#### Error Rate

```promql
# Error rate percentage
rate(orderorchestrator_errors_total[5m]) 
/ 
rate(orderorchestrator_requests_total[5m]) * 100

# Alert: High Error Rate
Alert if: error_rate > 5% for 5 minutes
Severity: P1 (HIGH)
Action: Investigate within 15 minutes
```

#### Payment Link Generation

```promql
# Payment link latency P95
histogram_quantile(0.95, 
  rate(orderorchestrator_payment_link_duration_seconds_bucket[5m]))

# Alert: Slow Payment Link
Alert if: P95 > 3s for 10 minutes
Severity: P2 (MEDIUM)
Action: Check Payment Processor health
```

#### Saga Execution

```promql
# Saga compensation rate
rate(orderorchestrator_saga_compensation_total[5m])

# Alert: High Compensation Rate
Alert if: compensation_rate > 10 per minute
Severity: P1 (HIGH)
Action: Check downstream service health
```

#### Database Connections

```promql
# Database connection pool usage
orderorchestrator_db_connections_in_use 
/ 
orderorchestrator_db_connections_max * 100

# Alert: Connection Pool Exhausted
Alert if: pool_usage > 90% for 5 minutes
Severity: P1 (HIGH)
Action: Scale database connections or pods
```

#### Redis Cache

```promql
# Cache hit rate
rate(orderorchestrator_cache_hits_total[5m]) 
/ 
rate(orderorchestrator_cache_requests_total[5m]) * 100

# Alert: Low Cache Hit Rate
Alert if: hit_rate < 70% for 10 minutes
Severity: P2 (MEDIUM)
Action: Check Redis health
```

---

## 🚨 Incident Response Playbook

### Phase 1: Detection & Triage (0-5 min)

**Automated Alerts**:
1. PagerDuty notification
2. Slack #mrg-alerts
3. Email to on-call

**On-Call Actions**:
```
1. Acknowledge alert in PagerDuty
2. Check Grafana dashboard: http://grafana/d/order-orchestrator
3. Quick triage:
   ├─ Service pods running? (kubectl get pods)
   ├─ Error logs? (kubectl logs -f deployment/orderorchestrator)
   ├─ Database accessible? (Check DB metrics)
   └─ Dependencies healthy? (Check dependency dashboard)
```

---

### Phase 2: Immediate Mitigation (5-15 min)

**Scenario A: Service Pods Down**
```bash
# Check pod status
kubectl get pods -l app=orderorchestrator

# If CrashLoopBackOff
kubectl describe pod <pod-name>

# Quick fix: Force restart
kubectl rollout restart deployment/orderorchestrator

# If OOMKilled: Increase memory limits (emergency)
kubectl patch deployment orderorchestrator \
  -p '{"spec":{"template":{"spec":{"containers":[{"name":"orderorchestrator","resources":{"limits":{"memory":"2Gi"}}}]}}}}'
```

**Scenario B: Database Connection Issues**
```bash
# Check database pods
kubectl get pods -l app=postgresql

# Check connection from Order Orchestrator
kubectl exec -it <oo-pod> -- nc -zv <db-host> 5432

# If network issue: Check network policies
kubectl get networkpolicies

# Emergency: Scale database connection pool
# Update ConfigMap and restart
```

**Scenario C: Redis Failure**
```bash
# Check Redis
kubectl get pods -l app=redis

# Temporary: Disable Redis in Order Orchestrator (fallback to DB)
kubectl set env deployment/orderorchestrator USE_REDIS=false

# Restart pods to pick up change
kubectl rollout restart deployment/orderorchestrator
```

---

### Phase 3: Root Cause Analysis (15-60 min)

**Data Collection**:
```bash
# Collect logs (last 1 hour)
kubectl logs --since=1h -l app=orderorchestrator > logs.txt

# Export metrics
curl http://orderorchestrator:8000/metrics > metrics.txt

# Database slow queries
SELECT * FROM pg_stat_statements ORDER BY mean_time DESC LIMIT 20;

# Check Elastic APM traces
# Look for slow transactions, error traces
```

**Analysis Checklist**:
- [ ] Error pattern (specific endpoint, all endpoints?)
- [ ] Timing (sudden spike, gradual degradation?)
- [ ] External factors (deployment, infrastructure change?)
- [ ] Dependency health (which service failed first?)
- [ ] Data consistency (any stuck orders?)

---

### Phase 4: Recovery & Validation (60+ min)

**Post-Recovery Checklist**:
```
1. Service Health
   ├─ All pods running? ✓
   ├─ Health checks passing? ✓
   ├─ Error rate normal? ✓
   └─ Response times normal? ✓

2. End-to-End Testing
   ├─ Create cart ✓
   ├─ Add items ✓
   ├─ Create order ✓
   ├─ Payment link generated ✓
   └─ Receive confirmation ✓

3. Data Consistency
   ├─ Check stuck orders (status = CREATED > 30 min)
   ├─ Check pending payments (waiting > 30 min)
   └─ Check saga compensations (any failures?)

4. Communication
   ├─ Update incident ticket
   ├─ Notify stakeholders (all clear)
   └─ Post-mortem meeting scheduled
```

---

## 📝 Post-Incident Review Template

**Incident**: Order Orchestrator Downtime  
**Date**: YYYY-MM-DD  
**Duration**: X minutes  
**Severity**: P0/P1/P2  
**Cause**: Root cause description

### Timeline
- **T+0**: Alert triggered
- **T+5**: On-call acknowledged
- **T+10**: Mitigation started
- **T+X**: Service recovered

### Impact
- Orders affected: X
- Revenue loss: $X
- Customer complaints: X

### Root Cause
[Detailed technical analysis]

### Action Items
- [ ] Immediate fix (owner, due date)
- [ ] Monitoring improvement (owner, due date)
- [ ] Process improvement (owner, due date)
- [ ] Architecture change (owner, due date)

---

## 🔗 Related Documentation

- [[README|Order Orchestrator Overview]]
- [[dependencies|Service Dependencies]]
- [[api-reference|API Reference]]
- [[/02-Work/Teams/MRG/01-architecture/single-point-of-failure-analysis|MRG SPOF Analysis]]

---

**Last Updated**: 2025-01-08  
**Next Review**: Q1 2025  
**Owner**: MRG Platform Team
