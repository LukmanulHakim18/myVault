# GCP to Huawei Cloud Migration - Runbook

**Purpose**: Operational guide untuk handling migration-related issues dan maintenance tasks  
**Audience**: Platform Engineers, DevOps, On-call Engineers  
**Last Updated**: 4 Agustus 2025

---

## 🚨 Emergency Contacts

| Role | Name | Contact | Availability |
|------|------|---------|--------------|
| Platform Architect | Lukmanul Hakim | [contact] | 24/7 |
| MRG Lead | [TBD] | [contact] | Business hours |
| UPG Lead | [TBD] | [contact] | Business hours |
| BBD Support | [TBD] | [contact] | 24/7 |
| Huawei Support | [TBD] | [contact] | Business hours |

---

## 🔧 Common Issues & Solutions

### Issue 1: Database Sequence Inconsistency

**Symptoms**:
- Duplicate order IDs
- Payment errors di UPG
- Database constraint violations

**Quick Diagnosis**:
```sql
-- Check current sequence value
SELECT last_value FROM order_id_seq;

-- Check max order ID in database
SELECT MAX(id) FROM orders;

-- Check gap
SELECT (SELECT last_value FROM order_id_seq) - (SELECT MAX(id) FROM orders) as gap;
```

**Solution**:
```sql
-- Set sequence to correct value
SELECT setval('order_id_seq', (SELECT MAX(id) FROM orders));

-- Verify
SELECT last_value FROM order_id_seq;
```

**Prevention**:
- Always sync sequence values during migration
- Add automated sequence validation checks
- Monitor sequence gaps in Grafana

---

### Issue 2: Marketing Service 500 Error

**Symptoms**:
- Promo code validation failures
- HTTP 500 responses dari marketing service
- Orders not created

**Quick Diagnosis**:
```bash
# Check marketing service health
curl -X GET https://marketing-service.hwc/health

# Check recent errors in logs
kubectl logs -n upg deployment/marketing-service --tail=100 | grep "ERROR"

# Check service connectivity
curl -X POST https://marketing-service.hwc/api/v1/promo/validate \
  -H "Content-Type: application/json" \
  -d '{"code":"TEST","user_id":"TEST"}'
```

**Solution**:
```bash
# 1. Check service status
kubectl get pods -n upg | grep marketing

# 2. Restart if unhealthy
kubectl rollout restart deployment/marketing-service -n upg

# 3. Monitor logs
kubectl logs -f deployment/marketing-service -n upg

# 4. Check config
kubectl get configmap marketing-config -n upg -o yaml

# 5. Verify database connection
kubectl exec -it deployment/marketing-service -n upg -- \
  psql -h $DB_HOST -U $DB_USER -d marketing_db -c "SELECT 1"
```

**Prevention**:
- Implement circuit breaker pattern
- Add proper error handling dan retry logic
- Monitor marketing service error rates

---

### Issue 3: EzPay IOT Race Condition

**Symptoms**:
- Multiple failed EzPay attempts
- Error: "Please make sure driver's argometer is activated"
- Customer complaints about street hailing

**Quick Diagnosis**:
```bash
# Check recent EzPay failures
kubectl logs -n mrg deployment/booking-service --tail=200 | \
  grep "retry_easy_ride_booking_error"

# Check IOT event lag
# Query monitoring dashboard: IOT Event Lag metric

# Find affected vehicle
grep "order_id=<ORDER_ID>" /var/log/booking-service/*.log
```

**Solution**:
```bash
# 1. Verify IOT status for vehicle
curl -X GET https://bbd-api/iot/status?car_number=PD2879

# 2. Check last known argo event
SELECT * FROM iot_events 
WHERE car_number = 'PD2879' 
ORDER BY created_at DESC 
LIMIT 5;

# 3. Manually verify with driver if needed
# (Contact driver operations team)

# 4. If IOT issue confirmed, escalate to BBD
# Use escalation template below
```

**Escalation Template to BBD**:
```
Subject: IOT Event Delay - Car [CAR_NUMBER]

Hi BBD Team,

We're experiencing IOT event delays for the following vehicle:
- Car Number: [CAR_NUMBER]
- Order ID: [ORDER_ID]
- Customer: [BB_ID]
- Time: [TIMESTAMP]
- Observed Delay: [XX] minutes

Impact:
- Customer unable to complete EzPay
- Multiple failed booking attempts

Please investigate IOT device and connectivity for this vehicle.

Tracking Link: [INCIDENT_LINK]
```

**Prevention**:
- Implement grace period for argo validation
- Add fallback validation mechanism
- Work with BBD on IOT reliability improvements

---

### Issue 4: Notification Service Failures

**Symptoms**:
- Notifications not delivered
- Chaining errors in notification logs
- RabbitMQ queue buildup

**Quick Diagnosis**:
```bash
# Check notification service health
kubectl get pods -n mrg | grep notification

# Check RabbitMQ queues
kubectl exec -n mrg rabbitmq-0 -- rabbitmqctl list_queues

# Check notification logs
kubectl logs -n mrg deployment/notification-center --tail=100 | grep "ERROR"

# Check topic prefix configuration
kubectl get configmap notification-config -n mrg -o yaml | grep "topic_prefix"
```

**Solution**:
```bash
# 1. Verify RabbitMQ connection
kubectl exec -it deployment/notification-center -n mrg -- \
  curl http://rabbitmq:15672/api/overview

# 2. Check auto-ack configuration
kubectl get configmap notification-config -n mrg -o yaml | grep "auto_ack"

# 3. Restart notification services
kubectl rollout restart deployment/notification-center -n mrg
kubectl rollout restart deployment/notification-forward -n mrg

# 4. Monitor queue recovery
watch -n 5 'kubectl exec rabbitmq-0 -n mrg -- rabbitmqctl list_queues'
```

**Prevention**:
- Fix auto-ack handling mechanism
- Proper topic prefix configuration
- Enable dead letter queues

---

### Issue 5: Service Can't Connect to Database

**Symptoms**:
- Service crash loops
- "Connection refused" errors
- Database timeout errors

**Quick Diagnosis**:
```bash
# Check service logs
kubectl logs -n mrg deployment/booking-service --tail=50 | grep -i "database\|connection"

# Check database endpoint in config
kubectl get configmap booking-config -n mrg -o yaml | grep -i "db_host\|database"

# Test database connectivity from pod
kubectl exec -it deployment/booking-service -n mrg -- \
  nc -zv $DB_HOST 5432

# Check DNS resolution
kubectl exec -it deployment/booking-service -n mrg -- \
  nslookup $DB_HOST
```

**Solution**:
```bash
# If using IP instead of DNS (BAD):
# Update configmap to use DNS name
kubectl edit configmap booking-config -n mrg
# Change: DB_HOST=10.x.x.x → DB_HOST=postgres.hwc.internal

# If DNS not resolving:
# Check DNS configuration
kubectl get svc -n mrg | grep postgres
kubectl describe svc postgres -n mrg

# Restart service after config change
kubectl rollout restart deployment/booking-service -n mrg
```

**Prevention**:
- **ALWAYS use DNS names, NEVER IP addresses**
- Exception: Kafka (doesn't support DNS)
- Document all service endpoints

---

## 📋 Maintenance Procedures

### Enabling Maintenance Mode

**Use Case**: Planned maintenance, emergency fixes, data migration

**Steps**:
```bash
# 1. Set maintenance flags in Redis
redis-cli -h redis.hwc.internal SET maintenance:advance_order "true"
redis-cli -h redis.hwc.internal SET maintenance:immediate_order "true"

# 2. Verify
redis-cli -h redis.hwc.internal GET maintenance:advance_order
redis-cli -h redis.hwc.internal GET maintenance:immediate_order

# 3. Monitor impact
# Check active orders dashboard
# Monitor customer support tickets

# 4. Disable maintenance mode when done
redis-cli -h redis.hwc.internal SET maintenance:advance_order "false"
redis-cli -h redis.hwc.internal SET maintenance:immediate_order "false"
```

### Rolling Back Service Deployment

**When to use**: Bad deployment, critical bugs, performance issues

**Steps**:
```bash
# 1. Check deployment history
kubectl rollout history deployment/booking-service -n mrg

# 2. View specific revision
kubectl rollout history deployment/booking-service -n mrg --revision=2

# 3. Rollback to previous version
kubectl rollout undo deployment/booking-service -n mrg

# 4. Rollback to specific revision
kubectl rollout undo deployment/booking-service -n mrg --to-revision=2

# 5. Monitor rollback progress
kubectl rollout status deployment/booking-service -n mrg

# 6. Verify service health
kubectl get pods -n mrg | grep booking-service
curl https://booking-service.hwc/health
```

### Database Failover to GCP (Emergency)

**Scenario**: Huawei database complete failure, need immediate failover

⚠️ **WARNING**: Only use in extreme emergency. Requires coordination with all teams.

**Steps**:
```bash
# 1. STOP all services writing to database
kubectl scale deployment --all --replicas=0 -n mrg
kubectl scale deployment --all --replicas=0 -n upg

# 2. Update DNS to point to GCP
# (Contact infrastructure team)

# 3. Update service configs to use GCP database
for ns in mrg upg; do
  kubectl get configmap -n $ns -o yaml | \
  sed 's/db.hwc.internal/db.gcp.internal/g' | \
  kubectl apply -f -
done

# 4. Restart services with new config
kubectl scale deployment --all --replicas=1 -n mrg
kubectl scale deployment --all --replicas=1 -n upg

# 5. Verify connectivity
kubectl get pods -A | grep -v "Running\|Completed"

# 6. Monitor logs for errors
kubectl logs -f deployment/booking-service -n mrg
```

---

## 📊 Monitoring & Alerts

### Critical Metrics to Watch

1. **Database**
   - Connection pool utilization
   - Query latency P50/P95/P99
   - Sequence values
   - Replication lag (if applicable)

2. **Services**
   - Error rate (target: <1%)
   - Response time P95 (target: <500ms)
   - Success rate (target: >99.5%)
   - Pod health and restarts

3. **External Dependencies**
   - Marketing service availability
   - IOT event lag
   - BBD API response time
   - Payment gateway success rate

4. **Infrastructure**
   - Kafka lag
   - RabbitMQ queue depth
   - Redis hit rate
   - Network latency

### Alert Thresholds

| Metric | Warning | Critical |
|--------|---------|----------|
| Error rate | >1% | >5% |
| Response time P95 | >500ms | >1000ms |
| Database connections | >70% | >85% |
| IOT event lag | >5 min | >15 min |
| Queue depth | >1000 | >5000 |

---

## 🔍 Debugging Tools

### Log Analysis
```bash
# Search for errors in last hour
kubectl logs --since=1h deployment/booking-service -n mrg | grep ERROR

# Follow logs in real-time
stern -n mrg booking-service

# Search across all pods
kubectl logs -n mrg -l app=booking-service --tail=1000 | grep "order_id=12345"
```

### Database Queries
```sql
-- Find orders by customer
SELECT * FROM orders WHERE customer_id = 'BB00215513' ORDER BY created_at DESC LIMIT 10;

-- Check recent errors
SELECT * FROM error_logs WHERE created_at > NOW() - INTERVAL '1 hour' ORDER BY created_at DESC;

-- Find duplicate orders
SELECT order_id, COUNT(*) FROM orders GROUP BY order_id HAVING COUNT(*) > 1;

-- Check IOT events for vehicle
SELECT * FROM iot_events WHERE car_number = 'PD2879' AND created_at > NOW() - INTERVAL '24 hours';
```

### Network Diagnostics
```bash
# Test connectivity from pod
kubectl run -it --rm debug --image=nicolaka/netshoot --restart=Never -- /bin/bash

# Inside pod:
curl -I https://marketing-service.hwc/health
nc -zv postgres.hwc.internal 5432
ping redis.hwc.internal
traceroute booking-service.mrg.svc.cluster.local
```

---

## 📞 Escalation Path

1. **Level 1**: On-call Engineer (< 15 minutes)
   - Check runbook
   - Follow standard procedures
   - Basic troubleshooting

2. **Level 2**: Team Lead (< 30 minutes)
   - Complex issues
   - Configuration changes
   - Cross-team coordination

3. **Level 3**: Platform Architect (< 1 hour)
   - Architecture decisions
   - Major incidents
   - Rollback decisions

4. **Level 4**: Management + Vendors
   - Service provider issues (Huawei, BBD)
   - Business impact decisions
   - External escalations

---

## 📚 Additional Resources

- [Migration Main Document](../../initiatives/gcp-to-hwc-migration.md)
- [Incident Reports](../incidents/gcp-hwc-migration/)
- [Service Documentation](../../Teams/)
- Grafana Dashboards: [Link to dashboards]
- PagerDuty: [Link to alerts]

---

**Document Owner**: Platform Engineering Team  
**Review Frequency**: Monthly or after each incident  
**Next Review**: September 2025

**Tags**: #runbook #operations #troubleshooting #migration #platform-engineering
