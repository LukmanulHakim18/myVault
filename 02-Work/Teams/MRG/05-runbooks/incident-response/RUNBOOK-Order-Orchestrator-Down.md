---
tags:
  - runbook
  - incident-response
  - order-orchestrator
  - p0
  - critical
type: runbook
severity: P0
created: '2026-01-08'
updated: '2026-01-08'
owner: platform-engineering
---
# 🚨 RUNBOOK: Order Orchestrator Down

**Severity**: P0 - Critical  
**Blast Radius**: 90% (all new bookings)  
**RTO Target**: < 3 minutes  
**Business Impact**: ZERO new orders possible

---

## 📋 Quick Reference

| Item | Value |
|------|-------|
| Service | `orderorchestrator` |
| Namespace | `mrg-production` |
| Replicas | 3 (minimum) |
| Health Endpoint | `/health` (liveness), `/ready` (readiness) |
| Grafana Dashboard | [Order Orchestrator](https://grafana.bluebird.id/d/order-orch) |
| Repository | `git.bluebird.id/mybb-ms/orderorchestrator` |
| On-Call | #mrg-oncall Slack |

---

## 🔍 Detection & Symptoms

### Alerts That Trigger This Runbook
```
- OrderOrchestratorDown: up{job="orderorchestrator"} == 0
- OrderOrchestratorHighErrorRate: error_rate > 0.05 for 2m
- OrderCreationZero: order_created_total rate == 0 for 5m
- CartCreationFailing: cart_creation_errors_total > 10 for 1m
```

### Symptoms
- [ ] Mobile app showing "Unable to create order" errors
- [ ] Cart creation returning 500/503 errors
- [ ] Payment callbacks failing (stuck in WAITING_FOR_PAYMENT)
- [ ] Grafana showing zero order creation rate
- [ ] Customer complaints spike

### Quick Diagnosis
```bash
# Check pod status
kubectl -n mrg-production get pods -l app=orderorchestrator

# Expected: All pods Running, Ready 1/1
# Problem indicators:
# - CrashLoopBackOff
# - ImagePullBackOff  
# - Pending (no resources)
# - Running but 0/1 Ready
```

---

## 🚀 Immediate Actions (First 3 Minutes)

### Step 1: Assess Pod Status (30 seconds)
```bash
# Get pod status
kubectl -n mrg-production get pods -l app=orderorchestrator -o wide

# Check recent events
kubectl -n mrg-production get events --sort-by='.lastTimestamp' | grep orderorchestrator | tail -20
```

### Step 2: Identify Root Cause

#### Scenario A: Pods CrashLoopBackOff
```bash
# Check pod logs
kubectl -n mrg-production logs -l app=orderorchestrator --tail=100

# Common causes:
# - Database connection failed
# - Redis connection failed
# - Config/secret missing
# - OOM killed

# Check previous pod logs
kubectl -n mrg-production logs -l app=orderorchestrator --previous --tail=50
```

**Quick Fixes:**
```bash
# If OOM - increase memory
kubectl -n mrg-production patch deployment orderorchestrator -p '{"spec":{"template":{"spec":{"containers":[{"name":"orderorchestrator","resources":{"limits":{"memory":"2Gi"}}}]}}}}'

# If config issue - check configmap/secrets
kubectl -n mrg-production get configmap orderorchestrator-config -o yaml
kubectl -n mrg-production get secret orderorchestrator-secrets -o yaml

# Force restart all pods
kubectl -n mrg-production rollout restart deployment/orderorchestrator
```

#### Scenario B: Pods Running but Not Ready
```bash
# Check readiness probe failures
kubectl -n mrg-production describe pod -l app=orderorchestrator | grep -A 10 "Readiness"

# Test health endpoint directly
kubectl -n mrg-production exec -it deploy/orderorchestrator -- curl -s localhost:8080/ready | jq

# Common causes:
# - Database not reachable
# - Redis not reachable
# - Downstream service down
```

**Quick Fixes:**
```bash
# Check database connectivity
kubectl -n mrg-production exec -it deploy/orderorchestrator -- nc -zv pgbouncer.mrg.internal 6432

# Check Redis connectivity  
kubectl -n mrg-production exec -it deploy/orderorchestrator -- nc -zv redis.mrg.internal 6379

# If downstream issue, check dependencies
curl -s http://userservice:8080/health
curl -s http://sessionmanager:8080/health
curl -s http://paymentprocessor:8080/health
```

#### Scenario C: Pods Pending (No Resources)
```bash
# Check node resources
kubectl top nodes

# Check pending reason
kubectl -n mrg-production describe pod -l app=orderorchestrator | grep -A 5 "Events"

# Scale down non-critical workloads temporarily
kubectl -n mrg-production scale deployment analytics-worker --replicas=0
```

#### Scenario D: Recent Deployment Broke It
```bash
# Check deployment history
kubectl -n mrg-production rollout history deployment/orderorchestrator

# Rollback to previous version
kubectl -n mrg-production rollout undo deployment/orderorchestrator

# Verify rollback
kubectl -n mrg-production rollout status deployment/orderorchestrator
```

### Step 3: Scale Up for Faster Recovery
```bash
# Temporarily increase replicas
kubectl -n mrg-production scale deployment/orderorchestrator --replicas=5

# Watch recovery
kubectl -n mrg-production get pods -l app=orderorchestrator -w
```

---

## 📊 Verification Steps

### Verify Service is Healthy
```bash
# All pods ready
kubectl -n mrg-production get pods -l app=orderorchestrator
# Should show: 3/3 Running, Ready

# Health check passing
curl -s http://orderorchestrator.mrg-production:8080/ready | jq
# Should show: {"status": "healthy", "checks": {...}}

# Test order creation (staging/canary)
curl -X POST http://orderorchestrator:8080/api/v1/carts \
  -H "Content-Type: application/json" \
  -d '{"user_id": "test-user", "session_id": "test-session"}'
```

### Verify Metrics Recovering
```bash
# Check Prometheus metrics
curl -s http://orderorchestrator:8080/metrics | grep order_created_total

# Check error rate dropping
curl -s http://orderorchestrator:8080/metrics | grep http_requests_total | grep 500
```

### Verify Downstream Integration
```bash
# Test payment callback processing
curl -X POST http://orderorchestrator:8080/api/v1/webhooks/payment \
  -H "Content-Type: application/json" \
  -d '{"order_id": "test", "status": "success"}'

# Check saga processing
kubectl -n mrg-production logs -l app=orderorchestrator --tail=20 | grep saga
```

---

## 🔄 Recovery of Stuck Orders

### Find Stuck Orders
```sql
-- Orders stuck in WAITING_FOR_PAYMENT (payment callback may have failed)
SELECT id, status, created_at, updated_at 
FROM orders 
WHERE status = 'WAITING_FOR_PAYMENT' 
AND created_at > NOW() - INTERVAL '1 hour'
ORDER BY created_at DESC;

-- Carts that never converted (checkout may have failed)
SELECT id, user_id, created_at 
FROM shopping_carts 
WHERE status = 'ACTIVE' 
AND created_at > NOW() - INTERVAL '2 hours'
AND created_at < NOW() - INTERVAL '30 minutes';
```

### Retry Failed Payment Callbacks
```bash
# List failed webhooks in DLQ
rabbitmqctl list_queues | grep payment-webhook-dlq

# Replay DLQ messages
# (Use admin tool or custom script)
./scripts/replay-dlq.sh payment-webhook-dlq
```

### Notify Affected Users
```bash
# Generate list of affected users
psql -h pgbouncer -d mrg -c "
SELECT DISTINCT u.email, u.phone, o.id as order_id
FROM orders o
JOIN users u ON o.user_id = u.id
WHERE o.status = 'WAITING_FOR_PAYMENT'
AND o.created_at BETWEEN '[INCIDENT_START]' AND '[INCIDENT_END]'
" > affected_users.csv

# Trigger notification (via admin tool)
./scripts/send-incident-notification.sh affected_users.csv
```

---

## 📢 Communication

### Stakeholder Notification Template
```
🚨 P0 INCIDENT: Order Service Disruption

Impact: New order creation affected
Start Time: [TIME] WIB
Current Status: Investigating / Mitigating / Resolved

Affected Flows:
- ❌ Create new bookings
- ❌ Add items to cart
- ❌ Checkout/payment
- ✅ Existing order tracking (unaffected)
- ✅ Driver dispatch for existing orders (unaffected)

Root Cause: [BRIEF DESCRIPTION]
ETA to Resolution: [X] minutes
Next Update: [TIME]

On-Call: @[NAME]
```

### Escalation Path
1. **0-5 min**: On-call engineer
2. **5-15 min**: Platform team lead
3. **15-30 min**: Engineering manager
4. **30+ min**: VP Engineering + Business stakeholders

---

## 🛡️ Preventive Measures

### Pre-deployment Checklist
- [ ] Load test passed with expected traffic
- [ ] Canary deployment showing healthy metrics
- [ ] Database migrations tested
- [ ] Rollback plan documented
- [ ] Feature flags ready for quick disable

### Monitoring to Add/Verify
- [ ] Order creation rate alert (< 50% baseline)
- [ ] Cart creation error rate alert
- [ ] P95 latency alert (> 2s)
- [ ] Pod restart alert (> 2 in 5min)

---

## 📚 Related Documents

- [[MRG SPOF Assessment & Mitigation Strategy]]
- [[02-Work/Teams/MRG/02-services/orderorchestrator/README|Order Orchestrator Service]]
- [[Circuit Breaker Implementation Guide]]
- [[Saga Pattern Recovery Procedures]]

---

**Last Updated**: 2026-01-08  
**Owner**: Platform Engineering Team  
**Review Cycle**: Quarterly
