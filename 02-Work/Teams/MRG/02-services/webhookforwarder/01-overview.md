# Webhook Forwarder Service - Overview

## Service Information
- **Service Name**: webhookforwarder
- **Team**: MRG (Meta Reservation Gateway)
- **Domain**: Webhook Management & Event Routing
- **Repository**: `git.bluebird.id/mybb-ms/webhookforwarder`
- **Language**: Go 1.24
- **Architecture Pattern**: Event Router / Facade Pattern
- **Local Path**: `D:\code\go\mybb-ms\webhookforwarder`

## Purpose
**Webhook Forwarder** adalah **centralized callback receiver** yang menerima semua webhook callbacks dari ekosistem MyBB external systems dan men-forward mereka ke internal microservices melalui message broker. Service ini bertindak sebagai **single entry point** untuk semua external callbacks.

### Core Responsibilities
1. **Webhook Reception** - Receive callbacks dari external systems
2. **Request Transformation** - Transform external format ke internal format
3. **Event Publishing** - Publish events ke message broker topics
4. **Backward Compatibility** - Support legacy systems dengan dual routing
5. **Callback Validation** - Validate incoming webhook requests
6. **Audit Logging** - Log all incoming webhooks untuk compliance

## Architecture Overview

### Event Router Pattern
Service ini menggunakan **Facade/Router pattern** untuk mengorganisir external callbacks:

```
┌──────────────────────────────────────────────────────────────┐
│                 EXTERNAL SYSTEMS                              │
│  (Payment Gateways, Order Systems, Document Generators)      │
└────────────────┬─────────────────────────────────────────────┘
                 │ Webhooks/Callbacks
                 │
                 ▼
┌────────────────────────────────────────────────────────────┐
│            WEBHOOK FORWARDER SERVICE                        │
│                                                             │
│  ┌─────────────────────────────────────────────────────┐  │
│  │            gRPC + REST Gateway                       │  │
│  │  - CititransRefundStatus                            │  │
│  │  - PaymentRefundStatus                              │  │
│  │  - PreAuthCallback                                  │  │
│  │  - OrderCallbackWeb/Mobile                          │  │
│  │  - PaymentNicepayCallback                           │  │
│  │  - DocumentGeneratorCallback                        │  │
│  │  - ReceiveNotificationCallback                      │  │
│  └────────────────┬────────────────────────────────────┘  │
│                   │                                         │
│  ┌────────────────▼────────────────────────────────────┐  │
│  │         Transformation Layer                        │  │
│  │  - Validate incoming requests                       │  │
│  │  - Transform external → internal format             │  │
│  │  - Enrich with metadata                             │  │
│  └────────────────┬────────────────────────────────────┘  │
│                   │                                         │
│  ┌────────────────▼────────────────────────────────────┐  │
│  │         Legacy Support (Optional)                   │  │
│  │  - Sync call ke legacy systems                      │  │
│  │  - Backward compatibility                           │  │
│  └────────────────┬────────────────────────────────────┘  │
│                   │                                         │
│  ┌────────────────▼────────────────────────────────────┐  │
│  │         Message Broker Publisher                    │  │
│  │  - Publish to PubSub topics                         │  │
│  │  - Async event delivery                             │  │
│  └─────────────────────────────────────────────────────┘  │
└────────────────┬────────────────────────────────────────────┘
                 │ Events
                 │
    ┌────────────┼────────────┬────────────┬─────────────┐
    ▼            ▼            ▼            ▼             ▼
┌──────┐    ┌──────┐    ┌──────┐    ┌──────┐    ┌──────────┐
│ TOP  │    │ COP  │    │ UPG  │    │Notif │    │Document  │
│      │    │      │    │      │    │Center│    │Service   │
└──────┘    └──────┘    └──────┘    └──────┘    └──────────┘
```

### Key Design Patterns

#### 1. Facade Pattern
Single unified interface untuk multiple callback sources:

```go
service Webhook {
    // Payment callbacks
    rpc PreAuthCallback(PreAuthCallbackRequest) returns (PreAuthCallbackResponse)
    rpc PaymentNicepayCallback(PaymentNicepayCallbackRequest) returns (DefaultResponse)
    rpc PaymentRefundStatus(PaymentRefundStatusRequest) returns (DefaultResponse)
    
    // Order callbacks
    rpc OrderCallbackWeb(OrderCallbackRequest) returns (DefaultResponse)
    rpc OrderCallbackMobile(OrderCallbackRequest) returns (DefaultResponse)
    
    // Refund callbacks
    rpc CititransRefundStatus(CititransStatusRequest) returns (DefaultResponse)
    
    // Document callbacks
    rpc DocumentGeneratorCallback(DocumentGeneratorCallbackRequest) returns (DefaultResponse)
    rpc GenerateDocumentCallback(DocumentCallbackRequest) returns (DefaultResponse)
    
    // Notification callbacks
    rpc ReceiveNotificationCallback(ReceiveNotificationCallbackRequest) returns (DefaultResponse)
}
```

**Benefits**:
- ✅ Single entry point untuk external systems
- ✅ Centralized validation dan logging
- ✅ Easy to add new callback types
- ✅ Decoupled external systems dari internal services

#### 2. Async Event Publishing Pattern
All callbacks published ke message broker untuk async processing:

```go
func (u *UseCase) OrderCallbackMobile(ctx context.Context, req *contract.OrderCallbackRequest) (*contract.DefaultResponse, error) {
    // 1. Log incoming request
    logger.Debug("incoming_request", req)
    
    // 2. Transform to internal format
    jsonReq := protojson.Marshal(req)
    
    // 3. Publish to message broker
    topicName := broker.NameWithEnv(TOPIC_ORDER_CALLBACK)
    err := u.Repo.Publisher.PublishMessage(ctx, topicName, jsonReq)
    
    // 4. Return immediately (async processing)
    return &DefaultResponse{Code: "WHFW-20102", Message: "success"}, nil
}
```

**Benefits**:
- ✅ Fast response time (no blocking)
- ✅ Decoupled from consumer processing
- ✅ Retry capability via message broker
- ✅ Multiple consumers can subscribe

#### 3. Transformation Layer Pattern
External format → Internal format transformation:

```go
func mappingCititransStatusRequest(req *contract.CititransStatusRequest) *pubsub.CititransRefundStatusRequest {
    return &pubsub.CititransRefundStatusRequest{
        BookingId:               req.GetBookingId(),
        OrderId:                 req.GetOrderId(),
        RefundStatusId:          req.GetRefundStatusId(),
        RefundStatusDescription: req.GetRefundStatusDescription(),
        UpdatedAt:               req.GetUpdatedAt(),
    }
}
```

## Callback Types

### 1. Payment Callbacks

#### PreAuthCallback
**Source**: UPG (Universal Payment Gateway)  
**Purpose**: Credit card pre-authorization result  
**Flow Type**: **Hybrid** (Sync legacy + Async pubsub)

**Process**:
```
1. Receive pre-auth callback dari UPG
2. SYNC call to OldPaymentProc (backward compatibility)
3. ASYNC publish to message broker
4. Return success response
```

**Request**:
```json
{
  "order_id": "123456",
  "user_id": "user_123",
  "payment_method": "credit_card",
  "payment_status": "approved",
  "payment_identifier": "card_token_abc",
  "transaction_id": "txn_456",
  "state": "success",
  "error_message": "",
  "error_code": ""
}
```

**Routing**:
- Sync → `OldPaymentProc.PreAuthCCCallback()`
- Async → Topic: `webhook_forwarder.pre_auth_callback`

---

#### PaymentNicepayCallback
**Source**: Nicepay Payment Gateway  
**Purpose**: Payment transaction status updates  
**Flow Type**: **Async**

**Request**:
```json
{
  "transaction_id": "NP123456",
  "partner_transaction_id": "order_789",
  "order_id": "123456",
  "transaction_type": "payment",
  "amount": 55000,
  "status": "success",
  "payment_method": "credit_card",
  "payment_method_name": "Visa",
  "bank_name": "BCA",
  "register_time": "2025-01-07T10:00:00Z",
  "paid_time": "2025-01-07T10:05:00Z",
  "card_number": "411111******1111",
  "approval_code": "APP123",
  "mdr_data": {
    "amount": 55000,
    "net_amount": 54450,
    "mdr_amount": 550,
    "remark": "MDR 1%"
  }
}
```

**Routing**:
- Topic: `webhook_forwarder.payment_nicepay_callback`

---

#### PaymentRefundStatus
**Source**: UPG (Universal Payment Gateway)  
**Purpose**: Refund transaction status updates  
**Flow Type**: **Async**

**Request**:
```json
{
  "merchant_metadata": {
    "request_id": "req_123"
  },
  "transaction_id": "txn_456",
  "payment_method_identifier": "gopay_wallet_789",
  "transaction_time": "2025-01-07T10:00:00Z",
  "provider": {
    "name": "GoPay",
    "payment_gateway": "midtrans",
    "transaction_id": "mid_txn_123",
    "state": "success"
  }
}
```

**Routing**:
- Topic: `payment.refund_status` (via commonmessaging)

---

### 2. Order Callbacks

#### OrderCallbackWeb
**Source**: BB-Order (Web Orders)  
**Purpose**: Order state updates dari web reservations  
**Flow Type**: **Async**

**Request**:
```json
{
  "event": {
    "id": "evt_123",
    "type": "driver_assigned",
    "time": "2025-01-07T10:00:00Z"
  },
  "order": {
    "order_id": 123456,
    "status": 1,
    "request_id": "req_abc"
  },
  "vehicle": {
    "vehicle_no": "B 1234 ABC",
    "phone": "+628123456789",
    "model": "Toyota Camry",
    "type": 1
  },
  "driver": {
    "nip": "DRV001",
    "name": "Pak Driver",
    "photo_url": "https://cdn.bluebird.id/drivers/001.jpg",
    "rating": 4.8
  },
  "state": 1
}
```

**Routing**:
- Topic: `webhook_forwarder.order_callback`
- Consumer: TOP (Taxi Order Processor)

---

#### OrderCallbackMobile
**Source**: BB-Order (Mobile Orders)  
**Purpose**: Order state updates dari mobile app orders  
**Flow Type**: **Async**

**Request**: Same structure as OrderCallbackWeb

**Routing**:
- Topic: `webhook_forwarder.order_callback`
- Consumer: TOP (Taxi Order Processor)

**Note**: Web dan Mobile callbacks have same topic but different success codes.

---

### 3. Refund Callbacks

#### CititransRefundStatus
**Source**: COP (Cititrans Order Processor)  
**Purpose**: Refund status updates untuk shuttle orders  
**Flow Type**: **Async** (dengan 1 detik delay)

**Request**:
```json
{
  "booking_id": "CTT-123456",
  "order_id": "456789",
  "refund_status_id": 2,
  "refund_status_description": "Refund approved",
  "updated_at": "2025-01-07T10:00:00Z"
}
```

**Special Handling**:
```go
// Delay untuk prevent race condition
time.Sleep(1 * time.Second)
```

**Routing**:
- Topic: `cititrans.refund_status` (via commonmessaging)
- Consumer: COP (Cititrans Order Processor)

---

### 4. Document Callbacks

#### DocumentGeneratorCallback
**Source**: Document Generator Service  
**Purpose**: Document generation completion notification  
**Flow Type**: **Async**

**Request**:
```json
{
  "booking_id": "BKG-123456",
  "order_id": "456789",
  "file_url": "https://cdn.bluebird.id/documents/invoice_123.pdf",
  "code": "success"
}
```

**Routing**:
- Topic: `documentgenerator.callback_listener`

---

#### GenerateDocumentCallback (Legacy)
**Source**: Legacy Document Generator  
**Purpose**: Legacy document generation callback  
**Flow Type**: **Async**

**Request**: Same structure as DocumentGeneratorCallback

**Routing**:
- Topic: `documentgenerator.callback_listener`

**Note**: Both callbacks route to same topic for unified processing.

---

### 5. Notification Callbacks

#### ReceiveNotificationCallback
**Source**: BB-One Notification Service  
**Purpose**: Notification delivery status dan token management  
**Flow Type**: **Async**

**Request**:
```json
{
  "recipient_id": "user_123_device_456",
  "client_id": "mybb_app",
  "title": "Order Update",
  "body": "Your driver has arrived",
  "message": "Driver: Pak Driver (B 1234 ABC)",
  "priority": "high",
  "status": "TOKEN_EXPIRED"
}
```

**Status Types**:
```go
const (
    RECEIVE_NOTIFICATION_CALLBACK_STATUS_TOKEN_EXPIRED = "TOKEN_EXPIRED"
    // Other statuses...
)
```

**Process** (for TOKEN_EXPIRED):
```
1. Extract user_id dari recipient_id (format: user_{id}_device_{id})
2. Validate recipient_id format
3. Publish delete token request
4. Return success
```

**Routing**:
- Topic: `notification.delete_bbone_user_recipient_id` (via commonmessaging)
- Consumer: NotificationCenter

---

## Message Broker Topics

### Topic Registry

| Topic Name | Purpose | Producer | Consumer |
|------------|---------|----------|----------|
| `webhook_forwarder.order_callback` | Order state updates | webhookforwarder | TOP |
| `webhook_forwarder.payment_nicepay_callback` | Nicepay payment updates | webhookforwarder | UPG/TOP |
| `webhook_forwarder.pre_auth_callback` | Pre-auth results | webhookforwarder | TOP |
| `documentgenerator.callback_listener` | Document generation | webhookforwarder | Document Service |
| `payment.refund_status` | Payment refunds | webhookforwarder | UPG |
| `cititrans.refund_status` | Shuttle refunds | webhookforwarder | COP |
| `notification.delete_bbone_user_recipient_id` | Token cleanup | webhookforwarder | NotificationCenter |

### Topic Naming Convention
```
{service_name}.{event_type}
```

Examples:
- `webhook_forwarder.order_callback` - webhook forwarder's order events
- `documentgenerator.callback_listener` - document generator's callback events
- `payment.refund_status` - payment domain refund events

---

## Integration Architecture

### Service Dependencies

```
┌─────────────────────────────────────────────────────────────┐
│                  WEBHOOK FORWARDER                           │
│                                                              │
│  External Integrations:                                      │
│  ┌──────────────────────────────────────────────────────┐  │
│  │  - UPG (Payment Gateway)                             │  │
│  │  - Nicepay (Payment Gateway)                         │  │
│  │  - BB-Order (Web/Mobile)                             │  │
│  │  - COP (Cititrans Order Processor)                   │  │
│  │  - Document Generator Service                        │  │
│  │  - BB-One Notification Service                       │  │
│  └──────────────────────────────────────────────────────┘  │
│                                                              │
│  Internal Dependencies:                                      │
│  ┌──────────────────────────────────────────────────────┐  │
│  │  - OldPaymentProc (Legacy, HTTP)                     │  │
│  │  - Message Broker (PubSub)                           │  │
│  └──────────────────────────────────────────────────────┘  │
└─────────────────────────────────────────────────────────────┘
```

### Integration Matrix

| External System | Protocol | Purpose | Criticality |
|----------------|----------|---------|-------------|
| **UPG** | gRPC/REST | Payment callbacks | **CRITICAL** |
| **Nicepay** | REST | Payment gateway | HIGH |
| **BB-Order** | gRPC/REST | Order callbacks | **CRITICAL** |
| **COP** | gRPC/REST | Shuttle refunds | HIGH |
| **Document Generator** | gRPC/REST | Document callbacks | MEDIUM |
| **BB-One Notification** | gRPC/REST | Notification status | MEDIUM |
| **OldPaymentProc** | HTTP | Legacy payment (sync) | HIGH |
| **Message Broker** | PubSub | Event publishing | **CRITICAL** |

---

## Error Handling

### Success Codes

```go
const (
    DefaultSuccess             = "WHFW-20000"  // Default success
    OrderCallbackWebSuccess    = "WHFW-20101"  // Web order callback
    OrderCallbackMobileSuccess = "WHFW-20102"  // Mobile order callback
    PaymentNicepaySuccess      = "WHFW-20103"  // Nicepay payment
)

// Legacy success response (backward compatibility)
var OldSuccessResponse = &DefaultResponse{
    Code:    "WFWD-2000",
    Message: "success",
}
```

### Error Codes

| Code | Description | HTTP |
|------|-------------|------|
| WHFW-4000 | Invalid request | 400 |
| WHFW-5000 | Internal error | 500 |
| WHFW-5001 | Failed to publish message | 500 |
| WHFW-5002 | Failed pre-auth callback | 500 |

### Error Response Format

```json
{
  "code": "WHFW-5001",
  "message": "Failed to publish message to broker",
  "details": {
    "topic": "webhook_forwarder.order_callback",
    "error": "connection timeout"
  }
}
```

---

## Observability

### APM Integration
```go
// gRPC tracing
apmgrpc.NewUnaryServerInterceptor()

// HTTP tracing
apmhttp.Wrap(gwMux)
```

**Tracked Operations**:
- Webhook reception
- Message publishing
- Legacy system calls
- Transformation operations

### Prometheus Metrics
```prometheus
# Webhook reception
whfw_webhooks_received_total{type="order_callback"} 5432
whfw_webhooks_received_total{type="payment_callback"} 3210

# Message publishing
whfw_messages_published_total{topic="order_callback",status="success"} 5430
whfw_messages_published_total{topic="order_callback",status="failed"} 2

# Legacy calls (for pre-auth)
whfw_legacy_calls_total{endpoint="pre_auth",status="success"} 890
whfw_legacy_calls_duration_seconds{endpoint="pre_auth"} 0.15
```

### Logging Strategy

**Debug Level**:
```go
logger.Debug("incoming_request", 
    "request", request,
    "method", "OrderCallbackMobile")
```

**Info Level**:
```go
logger.Info("success.PublishOrderCallbackMobile",
    "order_id", request.OrderId,
    "topic", topicName)
```

**Error Level**:
```go
logger.Error("error.PublishMessage",
    "error", err,
    "topic", topicName,
    "request", request)
```

---

## Performance Characteristics

### Response Times
- **Average Latency**: <50ms (webhook reception + publish)
- **P95 Latency**: <100ms
- **P99 Latency**: <200ms

### Throughput
- **Peak Rate**: ~500 webhooks/second
- **Average Rate**: ~100 webhooks/second
- **Daily Volume**: ~8M webhooks/day

### Reliability
- **Availability**: 99.9% SLA
- **Success Rate**: >99.5%
- **Message Loss**: <0.01% (with broker retry)

---

## Configuration

### Environment Variables

```bash
# Service Config
GRPC_PORT=50051
REST_PORT=8051
LOG_LEVEL=INFO

# Message Broker
PUBSUB_PROJECT_ID=mybb-project
PUBSUB_CREDENTIALS_FILE=/path/to/credentials.json

# Legacy System (OldPaymentProc)
OLD_PAYMENT_PROC_URL=http://old-payment-proc:8080

# Elastic APM
ELASTIC_APM_SERVER_URL=http://apm-server:8200
ELASTIC_APM_SERVICE_NAME=webhookforwarder
ELASTIC_APM_ENVIRONMENT=production

# Prometheus
PROMETHEUS_PORT=9090
```

---

## Best Practices

### 1. Always Return Immediately
```go
// ✅ GOOD: Quick response
func (u *UseCase) OrderCallback(ctx context.Context, req *Request) (*Response, error) {
    // Publish to broker (async)
    u.Repo.Publisher.PublishMessage(ctx, topic, data)
    
    // Return immediately
    return &Response{Code: "200", Message: "success"}, nil
}

// ❌ BAD: Blocking response
func (u *UseCase) OrderCallback(ctx context.Context, req *Request) (*Response, error) {
    // Wait for processing (blocks webhook sender)
    result := u.ProcessOrder(req)  // DON'T DO THIS
    return result, nil
}
```

### 2. Validate Before Publishing
```go
// Validate incoming request
if req.OrderId == "" {
    return nil, errors.New("order_id required")
}

// Transform and validate
message := transformRequest(req)
if err := validateMessage(message); err != nil {
    return nil, err
}

// Then publish
u.Repo.Publisher.PublishMessage(ctx, topic, message)
```

### 3. Comprehensive Logging
```go
// Log incoming
logger.Debug("incoming_request", "request", req)

// Log publishing
logger.Info("publishing_message", "topic", topic, "order_id", orderId)

// Log errors with context
logger.Error("publish_failed", 
    "error", err,
    "topic", topic,
    "request", req)
```

### 4. Graceful Error Handling
```go
// Publish error doesn't fail webhook
err := u.Repo.Publisher.PublishMessage(ctx, topic, data)
if err != nil {
    // Log error
    logger.Error("publish_failed", "error", err)
    
    // But still return success to sender
    // (prevent webhook retry spam)
}

return &Response{Code: "200", Message: "success"}, nil
```

---

## Service Maturity

### Current Status: **Production-Ready** ⭐⭐⭐⭐

**Strengths**:
- ✅ Single entry point for all webhooks
- ✅ Decoupled architecture (async processing)
- ✅ Backward compatibility support
- ✅ Comprehensive logging
- ✅ Fast response times

**Areas for Improvement**:
- 📊 Test coverage (current: ~60%, target: 85%)
- 🔄 Add webhook retry mechanism
- 📈 Enhanced metrics and monitoring
- 🔐 Webhook signature verification
- 📝 OpenAPI documentation

---

## Deployment

### Docker
```bash
docker build -t webhookforwarder:latest .
docker run -p 50051:50051 -p 8051:8051 \
  --env-file .env \
  webhookforwarder:latest
```

### Kubernetes
```bash
kubectl apply -f k8s/huawei-application.yaml
kubectl get pods -l app=webhookforwarder
kubectl logs -f deployment/webhookforwarder
```

---

## Related Documentation
- [[02-api-reference|API Reference]] - Complete API documentation
- [[03-webhook-routing|Webhook Routing]] - Routing logic details
- [[04-message-topics|Message Topics]] - PubSub topics reference

---
**Last Updated**: 2025-01-07
**Service Owner**: MRG Team
**Documentation Status**: Complete