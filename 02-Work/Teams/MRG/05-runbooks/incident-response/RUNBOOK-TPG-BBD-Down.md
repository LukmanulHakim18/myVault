---
tags:
  - runbook
  - incident-response
  - taxipartnergateway
  - bbd
  - p0
  - critical
  - external-dependency
type: runbook
severity: P0
created: '2026-01-08'
updated: '2026-01-08'
owner: platform-engineering
---
# 🚨 RUNBOOK: Taxi Partner Gateway / BBD System Down

**Severity**: P0 - Critical  
**Blast Radius**: 60% (taxi orders = ~60% volume)  
**RTO Target**: < 5 minutes (for fallback activation)  
**Business Impact**: All taxi bookings blocked

---

## 📋 Quick Reference

| Item | Value |
|------|-------|
| Service | `taxipartnergateway` |
| Namespace | `mrg-production` |
| BBD Endpoint | `https://bbd.bluebird.id/api/v2` |
| BBD Health Check | `https://bbd.bluebird.id/health` |
| Grafana Dashboard | [TPG Monitoring](https://grafana.bluebird.id/d/tpg) |
| Repository | `git.bluebird.id/mybb-ms/taxipartnergateway` |
| BBD Contact | Operations Center: +62-21-xxx-xxxx |
| On-Call | #mrg-oncall Slack |

---

## 🔍 Detection & Symptoms

### Alerts That Trigger This Runbook
```
- TPGDown: up{job="taxipartnergateway"} == 0
- BBDConnectionLost: tpg_bbd_connection_status == 0
- TaxiOrderCreationFailing: tpg_order_creation_errors_total > 10 for 1m
- BBDLatencyHigh: tpg_bbd_request_duration_p95 > 10s
- VehicleSearchFailing: tpg_vacant_vehicle_errors_total > 5 for 1m
```

### Symptoms
- [ ] "No vehicles available" even in high-density areas
- [ ] Taxi orders stuck in PENDING state
- [ ] Vehicle tracking not updating
- [ ] Driver assignment failing
- [ ] Customer complaints: "Can't book taxi"

### Distinguish: TPG Down vs BBD Down

```bash
# Check TPG service health
curl -s http://taxipartnergateway:8080/health | jq

# Check BBD connectivity from TPG
kubectl -n mrg-production exec -it deploy/taxipartnergateway -- \
  curl -s -w "\n%{http_code}" https://bbd.bluebird.id/health

# If TPG healthy but BBD unreachable = BBD issue
# If TPG unhealthy = TPG issue (restart pods)
```

---

## 🚀 Immediate Actions

### Step 1: Identify Root Cause (1 minute)

```bash
# Check TPG pod status
kubectl -n mrg-production get pods -l app=taxipartnergateway

# Check TPG logs for BBD errors
kubectl -n mrg-production logs -l app=taxipartnergateway --tail=50 | grep -i "bbd\|error\|timeout"

# Test BBD directly
curl -s -w "\nHTTP Code: %{http_code}\nTime: %{time_total}s\n" \
  https://bbd.bluebird.id/health
```

### Step 2: Handle Based on Scenario

#### Scenario A: TPG Service Down (Not BBD)
```bash
# Restart TPG pods
kubectl -n mrg-production rollout restart deployment/taxipartnergateway

# Scale up if needed
kubectl -n mrg-production scale deployment/taxipartnergateway --replicas=5

# Watch recovery
kubectl -n mrg-production get pods -l app=taxipartnergateway -w
```

#### Scenario B: BBD System Down (External)

**Immediate Actions:**
```bash
# 1. Activate manual dispatch queue (if implemented)
kubectl -n mrg-production set env deployment/taxipartnergateway \
  ENABLE_MANUAL_DISPATCH_QUEUE=true

# 2. Enable degraded mode notice to customers
kubectl -n mrg-production set env deployment/orderorchestrator \
  TAXI_SERVICE_DEGRADED=true \
  TAXI_DEGRADED_MESSAGE="Taxi booking experiencing delays. Your order will be processed shortly."
```

**Contact BBD Operations:**
```
📞 BBD Operations Center: +62-21-xxx-xxxx
📧 Email: ops@bluebird.co.id
🎫 Ticket: Create in ServiceNow

Information to provide:
- Incident start time
- Error messages from TPG logs
- HTTP response codes from BBD
- Number of affected orders
```

**Monitor BBD Recovery:**
```bash
# Continuous health check
while true; do
  echo "$(date): $(curl -s -o /dev/null -w '%{http_code}' https://bbd.bluebird.id/health)"
  sleep 10
done
```

#### Scenario C: BBD Latency Issues (Slow but Not Down)
```bash
# Increase timeout temporarily
kubectl -n mrg-production set env deployment/taxipartnergateway \
  BBD_TIMEOUT_SECONDS=30

# Enable circuit breaker half-open mode
# (Allow some requests through to test recovery)
kubectl -n mrg-production set env deployment/taxipartnergateway \
  CIRCUIT_BREAKER_HALF_OPEN_REQUESTS=5
```

#### Scenario D: BBD Auth Issues
```bash
# Check if auth token expired
kubectl -n mrg-production logs -l app=taxipartnergateway --tail=20 | grep -i "401\|403\|auth\|token"

# Refresh BBD credentials
kubectl -n mrg-production delete pod -l app=taxipartnergateway
# Pods will restart and re-authenticate

# If credential issue, update secret
kubectl -n mrg-production edit secret bbd-credentials
# Then restart pods
```

---

## 🔄 Manual Dispatch Fallback Procedure

When BBD is down and orders are queued, operations team can manually dispatch:

### Access Manual Dispatch Dashboard
```
URL: https://ops.bluebird.id/manual-dispatch
Login: Operations team credentials
```

### View Queued Orders
```sql
-- Query queued orders awaiting dispatch
SELECT 
  id,
  pickup_location,
  dropoff_location,
  user_phone,
  vehicle_type,
  created_at,
  retry_count
FROM taxi_dispatch_queue
WHERE status = 'PENDING'
ORDER BY created_at ASC;
```

### Manual Dispatch Process
1. Open order in dashboard
2. Contact dispatch center via radio/phone
3. Get driver assignment manually
4. Update order with driver info:
```bash
curl -X PUT http://taxipartnergateway:8080/admin/orders/{order_id}/assign \
  -H "Authorization: Bearer $ADMIN_TOKEN" \
  -d '{"driver_id": "DRV123", "vehicle_number": "B 1234 ABC", "manual_dispatch": true}'
```

### Bulk Process After Recovery
```bash
# When BBD recovers, replay queued orders
./scripts/replay-dispatch-queue.sh

# Monitor queue drain
watch "psql -c \"SELECT COUNT(*) FROM taxi_dispatch_queue WHERE status = 'PENDING'\""
```

---

## 📊 Verification Steps

### Verify TPG Recovery
```bash
# Check all pods healthy
kubectl -n mrg-production get pods -l app=taxipartnergateway

# Test health endpoint
curl -s http://taxipartnergateway:8080/ready | jq

# Test BBD connectivity
curl -s http://taxipartnergateway:8080/health | jq '.checks.bbd'
```

### Verify Order Flow
```bash
# Test vacant vehicle search
curl -X POST http://taxipartnergateway:8080/api/v1/vehicles/vacant \
  -H "Content-Type: application/json" \
  -d '{"lat": -6.2088, "lng": 106.8456, "radius_km": 5}'

# Test order creation (staging)
curl -X POST http://taxipartnergateway:8080/api/v1/orders \
  -H "Content-Type: application/json" \
  -d '{"pickup": {...}, "dropoff": {...}, "vehicle_type": "reguler"}'
```

### Verify Metrics
```bash
# Check success rate recovering
curl -s http://taxipartnergateway:8080/metrics | grep tpg_order_creation

# Check BBD latency normalizing
curl -s http://taxipartnergateway:8080/metrics | grep tpg_bbd_request_duration
```

---

## 🔄 Recovery of Stuck Orders

### Find Stuck Taxi Orders
```sql
-- Orders stuck in dispatch
SELECT id, status, created_at, updated_at, error_message
FROM orders
WHERE service_type = 'TAXI'
AND status IN ('PENDING_DISPATCH', 'DISPATCH_FAILED')
AND created_at > NOW() - INTERVAL '2 hours'
ORDER BY created_at DESC;
```

### Retry Failed Orders
```bash
# Retry specific order
curl -X POST http://taxipartnergateway:8080/admin/orders/{order_id}/retry-dispatch \
  -H "Authorization: Bearer $ADMIN_TOKEN"

# Bulk retry all failed orders
./scripts/retry-failed-taxi-orders.sh --since "2026-01-08 10:00:00"
```

### Refund/Cancel Stuck Orders
```bash
# If BBD outage was extended, some orders may need cancellation
./scripts/cancel-stuck-orders.sh \
  --status PENDING_DISPATCH \
  --older-than 30m \
  --reason "System issue - automatic refund"
```

---

## 📢 Communication

### Stakeholder Notification Template
```
🚨 P0 INCIDENT: Taxi Booking Service Disruption

Impact: Taxi bookings affected
Start Time: [TIME] WIB
Current Status: Investigating / BBD Issue / Mitigating / Resolved

Affected Flows:
- ❌ New taxi bookings
- ❌ Vehicle search
- ⚠️ Driver tracking (delayed updates)
- ✅ Rent car bookings (unaffected)
- ✅ Shuttle bookings (unaffected)
- ✅ Existing trip tracking (unaffected)

Root Cause: [TPG service issue / BBD system issue]
Workaround: [Manual dispatch activated / Customers advised to retry]
ETA to Resolution: [Dependent on BBD / X minutes]
Next Update: [TIME]

On-Call: @[NAME]
BBD Contact: [NAME] from BBD Ops
```

### Customer-Facing Message
```
Taxi booking is temporarily unavailable. 
Please try again in a few minutes or use our Rent Car service.
We apologize for the inconvenience.

Pemesanan taksi sedang tidak tersedia sementara.
Silakan coba beberapa menit lagi atau gunakan layanan Rent Car kami.
Mohon maaf atas ketidaknyamanannya.
```

---

## 🛡️ Preventive Measures

### Recommended Improvements (From SPOF Assessment)

| Priority | Improvement | Status | Sprint |
|----------|-------------|--------|--------|
| P0 | Circuit breaker for BBD calls | ❌ Not impl | Sprint 1 |
| P0 | Manual dispatch queue | ⚠️ Partial | Sprint 1 |
| P1 | BBD health check with alerting | ✅ Done | - |
| P1 | Graceful degradation messaging | ❌ Not impl | Sprint 2 |
| P2 | Alternative dispatch provider | ❌ Not started | Q2 |

### Monitoring to Verify
- [ ] BBD connectivity alert (connection_status == 0)
- [ ] BBD latency alert (p95 > 5s)
- [ ] Order dispatch failure rate (> 5%)
- [ ] Queue depth alert (> 100 orders)

---

## 📚 Related Documents

- [[MRG SPOF Assessment & Mitigation Strategy]]
- [[02-Work/Teams/MRG/02-services/taxipartnergateway/README|Taxi Partner Gateway Service]]
- [[BBD Integration Specification]]
- [[Manual Dispatch Operations Guide]]

---

**Last Updated**: 2026-01-08  
**Owner**: Platform Engineering Team  
**Review Cycle**: Quarterly
