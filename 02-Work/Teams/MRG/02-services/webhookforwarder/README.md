# Webhook Forwarder - Service Documentation

## Quick Links
- [[01-overview|📋 Service Overview]] - Architecture, patterns, dan responsibilities
- [[02-api-reference|📡 API Reference]] - Complete API documentation (9 endpoints)

## Service Summary

**Webhook Forwarder** adalah **centralized callback receiver hub** yang menerima semua webhook callbacks dari external systems dalam ekosistem MyBB dan men-route mereka ke appropriate internal microservices via message broker.

### Key Capabilities
- 🎯 **Single Entry Point** - Unified interface untuk 9+ callback types
- ⚡ **Async Processing** - Immediate response dengan async event publishing
- 🔄 **Event Routing** - Smart routing ke appropriate message topics
- 🔙 **Backward Compatible** - Support legacy systems dengan dual routing
- 📊 **Full Observability** - Comprehensive logging dan metrics

### Architecture Pattern
```
External Callbacks → Webhook Forwarder → Message Broker → Internal Services
```

### Supported Callbacks
1. **Payment Callbacks** (3 types)
   - Pre-authorization results
   - Nicepay payment gateway
   - Refund status updates

2. **Order Callbacks** (2 types)
   - Web order updates
   - Mobile order updates

3. **Refund Callbacks** (1 type)
   - Cititrans shuttle refunds

4. **Document Callbacks** (2 types)
   - Document generation completion
   - Legacy document callback

5. **Notification Callbacks** (1 type)
   - FCM token management

### Technology Stack
- **Language**: Go 1.24
- **Protocol**: gRPC + REST Gateway
- **Message Broker**: Google Cloud Pub/Sub
- **Observability**: Elastic APM + Prometheus

### Performance
- **Latency P95**: <100ms
- **Throughput**: ~500 webhooks/second
- **Availability**: 99.9% SLA
- **Daily Volume**: ~8M webhooks/day

---

## Documentation Structure

### 01-overview.md
Comprehensive service overview covering:
- Service purpose dan responsibilities
- Architecture patterns (Facade, Async Publishing)
- Complete callback types dengan routing details
- Integration architecture
- Message broker topics
- Error handling strategies
- Observability configuration
- Performance characteristics
- Best practices

### 02-api-reference.md
Complete API documentation including:
- All 9 webhook endpoints dengan detailed examples
- Request/response formats untuk each callback type
- Success dan error code reference
- Message broker topic routing table
- Testing commands (cURL examples)
- Best practices untuk webhook integration

---

## Quick Start

### Local Development
```bash
# Clone repository
git clone git@git.bluebird.id:mybb-ms/webhookforwarder.git
cd webhookforwarder

# Setup environment
cp .envsample .env
# Edit .env dengan proper configuration

# Run service
go run main.go
```

### Testing
```bash
# Test health check
curl http://localhost:8051/health

# Test webhook callback
curl -X POST http://localhost:8051/v1/webhooks/order-callback-mobile \
  -H "Content-Type: application/json" \
  -d '{"event":{"id":"test","type":"test","time":"2025-01-07T10:00:00Z"},"order":{"order_id":123,"status":1},"state":1}'
```

---

## Key Features

### 1. Centralized Webhook Management
Single service manages all external callbacks:
- Simplified webhook endpoint configuration untuk external systems
- Centralized logging dan monitoring
- Consistent error handling
- Unified authentication/authorization

### 2. Async Event Processing
Fast webhook response dengan async processing:
```
Response Time: <50ms average
Processing: Async via message broker
Reliability: Broker retry mechanism
Decoupling: Services process independently
```

### 3. Backward Compatibility
Support legacy systems:
- Dual routing (sync legacy + async pubsub)
- Gradual migration path
- No breaking changes untuk external systems

### 4. Smart Routing
Automatic routing ke appropriate consumers:
```
Payment → UPG/TOP
Orders → TOP
Refunds → COP
Documents → Document Service
Notifications → NotificationCenter
```

---

## Integration Guide

### For External Systems (Webhook Senders)

**1. Configure Webhook URL**:
```
Production: https://webhookforwarder.bluebird.id/v1/webhooks/{endpoint}
Staging: https://webhookforwarder-stg.bluebird.id/v1/webhooks/{endpoint}
```

**2. Implement Retry Logic**:
```
Max Retries: 3
Backoff: Exponential (1s, 2s, 4s)
Timeout: 30s per attempt
```

**3. Handle Response**:
```json
// Success (always 200 OK)
{
  "code": "WHFW-20xxx",
  "message": "success"
}
```

**4. Common Endpoints**:
- `/v1/webhooks/pre-auth-callback` - Payment pre-auth
- `/v1/webhooks/order-callback-mobile` - Mobile orders
- `/v1/webhooks/order-callback-web` - Web orders
- `/v1/webhooks/payment-nicepay-callback` - Nicepay
- `/v1/webhooks/cititrans-refund-status` - Cititrans refunds

### For Internal Services (Message Consumers)

**1. Subscribe to Topics**:
```go
// Example: Subscribe to order callbacks
topic := "webhook_forwarder.order_callback"
subscription := client.Subscription(topic)
```

**2. Process Messages**:
```go
subscription.Receive(ctx, func(ctx context.Context, msg *pubsub.Message) {
    // Parse message
    var callback OrderCallback
    json.Unmarshal(msg.Data, &callback)
    
    // Process callback
    processOrderCallback(callback)
    
    // Acknowledge
    msg.Ack()
})
```

**3. Available Topics**:
- `webhook_forwarder.order_callback` - Order updates
- `webhook_forwarder.pre_auth_callback` - Pre-auth results
- `webhook_forwarder.payment_nicepay_callback` - Nicepay payments
- `cititrans.refund_status` - Shuttle refunds
- `documentgenerator.callback_listener` - Documents

---

## Troubleshooting

### Common Issues

#### 1. Webhook Not Received
**Check**:
- Service health: `curl http://webhookforwarder:8051/health`
- Network connectivity
- Firewall rules
- Correct endpoint URL

#### 2. Message Not Published
**Check Logs**:
```bash
kubectl logs -f deployment/webhookforwarder | grep "error.PublishMessage"
```

**Common Causes**:
- Broker connection issues
- Invalid message format
- Topic permissions

#### 3. Legacy System Timeout (PreAuth)
**Symptoms**: PreAuthCallback returns timeout error

**Solution**:
```bash
# Check legacy system health
curl http://old-payment-proc:8080/health

# Increase timeout if needed
LEGACY_TIMEOUT=60s  # in config
```

---

## Monitoring

### Key Metrics
```prometheus
# Webhook reception
whfw_webhooks_received_total{type="order_callback"}

# Publishing success rate
rate(whfw_messages_published_total{status="success"}[5m]) /
rate(whfw_messages_published_total[5m])

# Response time
whfw_webhook_duration_seconds{endpoint="order_callback"}
```

### Alerts
```yaml
- alert: HighWebhookFailureRate
  expr: rate(whfw_messages_published_total{status="failed"}[5m]) > 0.01
  severity: warning

- alert: WebhookServiceDown
  expr: up{job="webhookforwarder"} == 0
  severity: critical
```

---

## Security

### Authentication
- **Incoming**: Validate webhook signatures (implementation pending)
- **Outgoing**: Service account for message broker

### Authorization
- **Endpoints**: Public (but should implement signature verification)
- **Message Topics**: Topic-level IAM controls

### Best Practices
- [ ] Implement webhook signature verification
- [ ] Rate limiting per sender
- [ ] IP whitelist for known external systems
- [ ] TLS for all connections

---

## Related Services
- **TOP** (Taxi Order Processor) - Consumes order callbacks
- **COP** (Cititrans Order Processor) - Sends/consumes shuttle callbacks
- **UPG** (Universal Payment Gateway) - Sends payment callbacks
- **NotificationCenter** - Consumes notification callbacks
- **Document Service** - Consumes document callbacks

---

## Maintenance

### Deployment
```bash
# Build
./build.sh

# Deploy to staging
./deploy.sh staging

# Deploy to production
./deploy.sh production
```

### Rolling Back
```bash
kubectl rollout undo deployment/webhookforwarder
kubectl rollout status deployment/webhookforwarder
```

---

**Service Status**: Production (Stable)  
**Team**: MRG (Meta Reservation Gateway)  
**Last Updated**: 2025-01-07