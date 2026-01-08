# API Reference - Webhook Forwarder

## Overview
Webhook Forwarder menyediakan unified API untuk menerima callbacks dari berbagai external systems dalam ekosistem MyBB. All endpoints support both gRPC dan REST protocols.

## Base URLs

### gRPC
```
grpc://webhookforwarder:50051
```

### REST (via Gateway)
```
http://webhookforwarder:8051
```

---

## API Categories

- [[#Health Check|Health Check]]
- [[#Payment Callbacks|Payment Callbacks]]
- [[#Order Callbacks|Order Callbacks]]
- [[#Refund Callbacks|Refund Callbacks]]
- [[#Document Callbacks|Document Callbacks]]
- [[#Notification Callbacks|Notification Callbacks]]

---

## Health Check

### HealthCheck
Check service health status.

**gRPC**: `Webhook.HealthCheck`  
**REST**: `GET /health`

**Request**:
```json
{}
```

**Response**:
```json
{
  "code": "200",
  "message": "OK"
}
```

---

## Payment Callbacks

### PreAuthCallback
Handle credit card pre-authorization result callback dari UPG.

**gRPC**: `Webhook.PreAuthCallback`  
**REST**: `POST /v1/webhooks/pre-auth-callback`

**Called By**: UPG (Universal Payment Gateway)

**Flow**:
```
1. Receive pre-auth result dari UPG
2. SYNC call to OldPaymentProc (backward compatibility)
3. ASYNC publish to message broker
4. Return success
```

**Request**:
```json
{
  "order_id": "123456",
  "user_id": "user_123",
  "payment_method": "credit_card",
  "payment_status": "approved",
  "payment_identifier": "card_token_abc123",
  "transaction_id": "txn_456789",
  "state": "success",
  "error_message": "",
  "error_code": ""
}
```

**Response** (Success):
```json
{
  "success": true,
  "error": ""
}
```

**Response** (Failed):
```json
{
  "success": false,
  "error": "Pre-auth callback to legacy system failed"
}
```

**Published To**:
- Topic: `webhook_forwarder.pre_auth_callback`
- Consumer: TOP (Taxi Order Processor)

**Legacy Compatibility**:
- Sync HTTP call to OldPaymentProc before publishing
- If legacy call fails, return error (don't publish)

**Error Codes**:
```json
{
  "code": "WHFW-5002",
  "message": "Failed pre-auth callback",
  "details": {
    "error": "legacy system timeout"
  }
}
```

---

### PaymentNicepayCallback
Handle payment transaction callback dari Nicepay gateway.

**gRPC**: `Webhook.PaymentNicepayCallback`  
**REST**: `POST /v1/webhooks/payment-nicepay-callback`

**Called By**: Nicepay Payment Gateway

**Request**:
```json
{
  "transaction_id": "NP20250107123456",
  "partner_transaction_id": "order_123456",
  "order_id": "123456",
  "transaction_type": "payment",
  "amount": 55000,
  "status": "success",
  "payment_method": "credit_card",
  "payment_method_name": "Visa",
  "bank_name": "BCA",
  "register_time": "2025-01-07T10:00:00Z",
  "deadline_time": "2025-01-07T11:00:00Z",
  "paid_time": "2025-01-07T10:05:00Z",
  "webhook_url": "https://webhookforwarder.bluebird.id/payment-nicepay-callback",
  "reference_number": "REF123456",
  "card_number": "411111******1111",
  "approval_code": "APP123456",
  "payment_gateway": "nicepay",
  "acquire_bank_name": "BCA",
  "billing_name": "John Doe",
  "acquirer_code": "BCA001",
  "mdr_data": {
    "amount": 55000,
    "net_amount": 54450,
    "mdr_amount": 550,
    "remark": "MDR 1%"
  }
}
```

**Response**:
```json
{
  "code": "WHFW-20103",
  "message": "success payment nicepay callback"
}
```

**Published To**:
- Topic: `webhook_forwarder.payment_nicepay_callback`
- Consumer: UPG / TOP

**Payment Statuses**:
- `success` - Payment completed successfully
- `pending` - Payment pending confirmation
- `failed` - Payment failed
- `expired` - Payment deadline expired

---

### PaymentRefundStatus
Handle refund transaction status callback dari UPG.

**gRPC**: `Webhook.PaymentRefundStatus`  
**REST**: `POST /v1/webhooks/payment-refund-status`

**Called By**: UPG (Universal Payment Gateway)

**Request**:
```json
{
  "merchant_metadata": {
    "request_id": "req_abc123"
  },
  "transaction_id": "txn_456789",
  "payment_method_identifier": "gopay_wallet_123456",
  "transaction_time": "2025-01-07T10:00:00Z",
  "provider": {
    "name": "GoPay",
    "payment_gateway": "midtrans",
    "transaction_id": "mid_txn_789012",
    "issuer_name": "GoPay",
    "acquirer_name": "Midtrans",
    "state": "success"
  }
}
```

**Response**:
```json
{
  "code": "WFWD-2000",
  "message": "success"
}
```

**Published To**:
- Topic: `payment.refund_status` (via commonmessaging)
- Consumer: UPG

**Provider States**:
- `success` - Refund completed
- `pending` - Refund in progress
- `failed` - Refund failed

---

## Order Callbacks

### OrderCallbackWeb
Handle order state updates dari web reservation system.

**gRPC**: `Webhook.OrderCallbackWeb`  
**REST**: `POST /v1/webhooks/order-callback-web`

**Called By**: BB-Order (Web Reservations)

**Request**:
```json
{
  "event": {
    "id": "evt_123456",
    "type": "driver_assigned",
    "time": "2025-01-07T10:00:00Z"
  },
  "order": {
    "order_id": 123456,
    "status": 1,
    "request_id": "req_abc123"
  },
  "track": {
    "location": {
      "latitude": -6.2088,
      "longitude": 106.8456
    }
  },
  "payments": [
    {
      "type": 1,
      "currency": "IDR",
      "amount": 55000
    }
  ],
  "fares": [
    {
      "type": 0,
      "currency": "IDR",
      "amount": 50000,
      "remark": "Base fare",
      "promo": {
        "code": "PROMO10",
        "type": 1,
        "name": "10% Discount",
        "discount": 5000
      }
    }
  ],
  "extras": [
    {
      "type": 1,
      "value": "toll_fee:5000"
    }
  ],
  "vehicle": {
    "vehicle_no": "B 1234 ABC",
    "phone": "+628123456789",
    "model": "Toyota Camry",
    "type": 1,
    "vehicle_attributes": [
      {
        "vehicle_id": 1,
        "key": "color",
        "value": "black"
      }
    ]
  },
  "driver": {
    "nip": "DRV001",
    "name": "Pak Driver",
    "photo_url": "https://cdn.bluebird.id/drivers/001.jpg",
    "rating": 4.8
  },
  "cancel": null,
  "trip": null,
  "evidences": [],
  "partner": {
    "customer_id": "cust_123",
    "order_id": "order_456",
    "service_id": "service_789"
  },
  "state": 1
}
```

**Response**:
```json
{
  "code": "WHFW-20101",
  "message": "success create callback order for web"
}
```

**Published To**:
- Topic: `webhook_forwarder.order_callback`
- Consumer: TOP (Taxi Order Processor)

**Event Types**:
- `driver_assigned` - Driver ditemukan dan assigned
- `driver_arrived` - Driver sudah sampai pickup
- `trip_started` - Trip dimulai
- `trip_completed` - Trip selesai
- `order_cancelled` - Order dibatalkan

**Order States**:
- `-3` - ORDER_TIMEOUT
- `-2` - ORDER_INITIAL
- `-1` - ORDER_FAILED
- `0` - LOOKING_FOR_DRIVER
- `1` - EN_ROUTE
- `2` - CANCELED
- `3` - NO_TAXI
- `4` - ON_TRIP
- `5` - TAXI_NO_SHOW
- `6` - TRACKING
- `7` - ECV_DROP_OFF
- `8` - DROP_OFF

---

### OrderCallbackMobile
Handle order state updates dari mobile app.

**gRPC**: `Webhook.OrderCallbackMobile`  
**REST**: `POST /v1/webhooks/order-callback-mobile`

**Called By**: BB-Order (Mobile App)

**Request**: Same structure as OrderCallbackWeb

**Response**:
```json
{
  "code": "WHFW-20102",
  "message": "success create callback order for mobile"
}
```

**Published To**:
- Topic: `webhook_forwarder.order_callback` (same as web)
- Consumer: TOP (Taxi Order Processor)

**Note**: Web dan Mobile callbacks use same message topic, only success codes differ.

---

## Refund Callbacks

### CititransRefundStatus
Handle refund status updates untuk Cititrans shuttle orders.

**gRPC**: `Webhook.CititransRefundStatus`  
**REST**: `POST /v1/webhooks/cititrans-refund-status`

**Called By**: COP (Cititrans Order Processor)

**Request**:
```json
{
  "booking_id": "CTT-20250107-123456",
  "order_id": "456789",
  "refund_status_id": 2,
  "refund_status_description": "Refund approved by finance team",
  "updated_at": "2025-01-07T10:00:00Z"
}
```

**Response**:
```json
{
  "code": "WFWD-2000",
  "message": "success"
}
```

**Published To**:
- Topic: `cititrans.refund_status` (via commonmessaging)
- Consumer: COP (Cititrans Order Processor)

**Refund Status IDs**:
```
1 - Pending review
2 - Approved
3 - Rejected
4 - Processing
5 - Completed
6 - Failed
```

**Special Handling**:
```go
// 1 second delay untuk prevent race condition
time.Sleep(1 * time.Second)
```

**Use Case**: Race condition bisa terjadi jika:
- User submit refund request
- System processes immediately
- Callback arrives before DB commit
- Delay memastikan DB state sudah consistent

---

## Document Callbacks

### DocumentGeneratorCallback
Handle document generation completion callback.

**gRPC**: `Webhook.DocumentGeneratorCallback`  
**REST**: `POST /v1/webhooks/document-generator-callback`

**Called By**: Document Generator Service

**Request**:
```json
{
  "booking_id": "BKG-20250107-123456",
  "order_id": "456789",
  "file_url": "https://cdn.bluebird.id/documents/invoice_123456.pdf",
  "code": "success"
}
```

**Response**:
```json
{
  "code": "WFWD-2000",
  "message": "success"
}
```

**Published To**:
- Topic: `documentgenerator.callback_listener`
- Consumer: Document Service

**Document Types**:
- `invoice` - Invoice/receipt
- `itinerary` - Trip itinerary
- `report` - Trip report
- `voucher` - Voucher document

**Callback Codes**:
- `success` - Document generated successfully
- `failed` - Document generation failed
- `pending` - Document generation in progress

---

### GenerateDocumentCallback (Legacy)
Handle document generation callback dari legacy system.

**gRPC**: `Webhook.GenerateDocumentCallback`  
**REST**: `POST /v1/webhooks/generate-document-callback`

**Called By**: Legacy Document Generator

**Request**: Same structure as DocumentGeneratorCallback

**Response**:
```json
{
  "code": "WFWD-2000",
  "message": "success"
}
```

**Published To**:
- Topic: `documentgenerator.callback_listener` (same as new)
- Consumer: Document Service

**Note**: Legacy dan new callbacks route to same topic for unified processing.

---

## Notification Callbacks

### ReceiveNotificationCallback
Handle notification delivery status dan FCM token management.

**gRPC**: `Webhook.ReceiveNotificationCallback`  
**REST**: `POST /v1/webhooks/receive-notification-callback`

**Called By**: BB-One Notification Service

**Request**:
```json
{
  "recipient_id": "user_123_device_456",
  "client_id": "mybb_mobile_app",
  "title": "Driver Arrived",
  "body": "Your driver has arrived at pickup location",
  "message": "Driver: Pak Driver (B 1234 ABC)",
  "priority": "high",
  "status": "TOKEN_EXPIRED"
}
```

**Response**:
```json
{
  "code": "WFWD-2000",
  "message": "success"
}
```

**Status Types**:
```
TOKEN_EXPIRED - FCM token sudah expired/invalid
DELIVERED     - Notification delivered successfully
FAILED        - Notification delivery failed
```

**Processing for TOKEN_EXPIRED**:
```
1. Parse recipient_id format: "user_{user_id}_device_{device_id}"
2. Validate format (must have exactly 2 underscores)
3. Extract user_id dan device_id
4. Publish delete token request
5. NotificationCenter akan remove expired token
```

**Published To** (for TOKEN_EXPIRED):
- Topic: `notification.delete_bbone_user_recipient_id`
- Consumer: NotificationCenter

**Error Cases**:
```json
// Empty recipient_id
{
  "error": "empty recipient_id"
}

// Invalid format
{
  "error": "invalid recipient_id",
  "detail": "expected format: user_{id}_device_{id}"
}

// Invalid status
{
  "error": "invalid 'status' input",
  "detail": "currently only TOKEN_EXPIRED is supported"
}
```

**Recipient ID Format**:
```
Pattern: user_{user_id}_device_{device_id}
Example: user_123_device_456

Parts:
- user_123: User identifier
- device_456: Device/FCM token identifier
```

---

## Common Response Formats

### Success Response
```json
{
  "code": "WHFW-20000",
  "message": "success"
}
```

### Error Response
```json
{
  "code": "WHFW-5001",
  "message": "Failed to publish message",
  "details": {
    "topic": "webhook_forwarder.order_callback",
    "error": "broker connection timeout",
    "retry_available": true
  }
}
```

---

## Success Code Reference

| Code | Description | Usage |
|------|-------------|-------|
| `WHFW-20000` | Default success | Generic success response |
| `WHFW-20101` | Order callback web success | Web order callback processed |
| `WHFW-20102` | Order callback mobile success | Mobile order callback processed |
| `WHFW-20103` | Payment Nicepay success | Nicepay callback processed |
| `WFWD-2000` | Legacy success | Legacy endpoints (backward compat) |

---

## Error Code Reference

| Code | HTTP | Description |
|------|------|-------------|
| WHFW-4000 | 400 | Invalid request format |
| WHFW-4001 | 400 | Missing required field |
| WHFW-4002 | 400 | Invalid field value |
| WHFW-5000 | 500 | Internal server error |
| WHFW-5001 | 500 | Failed to publish message |
| WHFW-5002 | 500 | Failed pre-auth callback |
| WHFW-5003 | 500 | Broker connection error |

---

## Message Broker Topics

### Topic Routing Table

| Webhook Endpoint | Message Topic | Consumer Service |
|-----------------|---------------|------------------|
| PreAuthCallback | `webhook_forwarder.pre_auth_callback` | TOP |
| PaymentNicepayCallback | `webhook_forwarder.payment_nicepay_callback` | UPG/TOP |
| PaymentRefundStatus | `payment.refund_status` | UPG |
| OrderCallbackWeb | `webhook_forwarder.order_callback` | TOP |
| OrderCallbackMobile | `webhook_forwarder.order_callback` | TOP |
| CititransRefundStatus | `cititrans.refund_status` | COP |
| DocumentGeneratorCallback | `documentgenerator.callback_listener` | Document Service |
| GenerateDocumentCallback | `documentgenerator.callback_listener` | Document Service |
| ReceiveNotificationCallback | `notification.delete_bbone_user_recipient_id` | NotificationCenter |

---

## Best Practices

### 1. Webhook Retry Strategy

**Sender Side**:
```
Retry Policy:
- Max retries: 3
- Backoff: Exponential (1s, 2s, 4s)
- Timeout per attempt: 30s
- Consider success if any attempt succeeds
```

### 2. Idempotency

All webhook endpoints are **idempotent**:
- Multiple identical requests produce same result
- Safe to retry
- No duplicate processing due to async pubsub

**Implementation**:
```
Webhook → Publish to broker → Consumer uses deduplication key
```

### 3. Timeout Handling

**Webhook sender timeout recommendations**:
```
Connection timeout: 5s
Read timeout: 30s
Total timeout: 35s
```

**webhookforwarder response time**:
```
Average: <50ms
P95: <100ms
P99: <200ms
```

### 4. Error Handling

**Always return 200 OK**:
```go
// Even if publish fails, return success
// to prevent webhook retry spam
err := publisher.Publish(topic, message)
if err != nil {
    logger.Error("publish_failed", err)
    // Still return success to sender
}
return &Response{Code: "200", Message: "success"}, nil
```

**Why?**
- Publishing errors are logged internally
- Message broker has its own retry mechanism
- Prevents exponential webhook retry load
- Sender gets immediate confirmation

---

## Testing

### Sample cURL Commands

**PreAuthCallback**:
```bash
curl -X POST http://webhookforwarder:8051/v1/webhooks/pre-auth-callback \
  -H "Content-Type: application/json" \
  -d '{
    "order_id": "123456",
    "user_id": "user_123",
    "payment_method": "credit_card",
    "payment_status": "approved",
    "payment_identifier": "card_token_abc",
    "transaction_id": "txn_456",
    "state": "success"
  }'
```

**OrderCallbackMobile**:
```bash
curl -X POST http://webhookforwarder:8051/v1/webhooks/order-callback-mobile \
  -H "Content-Type: application/json" \
  -d '{
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
    "state": 1
  }'
```

**CititransRefundStatus**:
```bash
curl -X POST http://webhookforwarder:8051/v1/webhooks/cititrans-refund-status \
  -H "Content-Type: application/json" \
  -d '{
    "booking_id": "CTT-123456",
    "order_id": "456789",
    "refund_status_id": 2,
    "refund_status_description": "Approved",
    "updated_at": "2025-01-07T10:00:00Z"
  }'
```

---

## Related Documentation
- [[01-overview|Service Overview]]
- [[03-webhook-routing|Webhook Routing Details]]
- [[04-message-topics|Message Topics Reference]]

---
**Last Updated**: 2025-01-07
**API Version**: v1
**Protocol**: gRPC + REST Gateway