# HO Gateway - API Reference

[[README|← Back to Overview]]

---

## gRPC Service Definition

**Package**: `hogateway`  
**Go Package**: `git.bluebird.id/mybb-ms/mybb-ho-gateway/contract`

---

## Service: HOGateway

```protobuf
service HOGateway {
    rpc HealthCheck(EmptyMessage) returns (DefaultResponse);
    rpc PublishMessage(PublishMessageRequest) returns (PublishMessageResponse);
    rpc SendMessage(SendMessageRequest) returns (SendMessageResponse);
    rpc HoEWalletTransaction(HoEWalletRequest) returns (DefaultResponse);
}
```

---

## Methods

### 1. HealthCheck

**Purpose**: Check service health status

**Request**: `EmptyMessage`
```protobuf
message EmptyMessage {}
```

**Response**: `DefaultResponse`
```protobuf
message DefaultResponse {
    string code = 1;      // Response code
    string message = 2;   // Response message
}
```

**Example Response**:
```json
{
  "code": "HOG-1001",
  "message": "Service is healthy"
}
```

**Usage**:
- Kubernetes liveness probe
- Service readiness check
- Monitoring health status

---

### 2. HoEWalletTransaction

**Purpose**: Process e-wallet payment transaction to HO system

**Request**: `HoEWalletRequest`
```protobuf
message HoEWalletRequest {
    string payment_method = 1;      // Payment method: gopay, ovo, dana, shopeepay, tcash
    string trx_date_time = 2;       // Transaction datetime (RFC3339 format)
    string itop_id = 3;             // iTOP operational city ID
    string external_order_id = 4;   // External order reference ID
    int64 order_id = 5;             // Internal order ID (unique)
    string currency = 6;            // Currency code (e.g., "IDR")
    double amount = 7;              // Payment amount
    double discount_amount = 8;     // Discount amount applied
    string promo_code = 9;          // Promo code used (optional)
    string bbid = 10;               // Customer phone number
    string transaction_id = 11;     // Payment gateway transaction ID
    string bank = 12;               // Bank/payment provider name
    double platform_fee = 13;       // Platform fee charged
    double sustainability = 14;     // Sustainability fee
    int32 attempt_count = 15;       // Current retry attempt count (0-based)
}
```

**Response**: `DefaultResponse`
```protobuf
message DefaultResponse {
    string code = 1;      // Response code (HOG-2001 = success, HOG-2002 = retry)
    string message = 2;   // Response message
}
```

**Payment Method Mapping**:
| payment_method | PayGateway Code (HO) |
|----------------|----------------------|
| `gopay` | `MIDGPY` |
| `ovo` | `OVO` |
| `dana` | `DANA` |
| `shopeepay` | `SHOPAY` |
| `tcash` | `TCASH` |

**Example Request**:
```json
{
  "payment_method": "gopay",
  "trx_date_time": "2025-01-08T10:30:00+07:00",
  "itop_id": "JKT01",
  "external_order_id": "ORD-123456",
  "order_id": 789012,
  "currency": "IDR",
  "amount": 150000,
  "discount_amount": 10000,
  "promo_code": "PROMO10",
  "bbid": "081234567890",
  "transaction_id": "TXN-GOPAY-123",
  "bank": "GoPay",
  "platform_fee": 2000,
  "sustainability": 1000,
  "attempt_count": 0
}
```

**Example Success Response**:
```json
{
  "code": "HOG-2001",
  "message": "Berhasil"
}
```

**Example Retry Response**:
```json
{
  "code": "HOG-2002",
  "message": "Mengulang transaksi"
}
```

**Response Codes**:
| Code | Description |
|------|-------------|
| `HOG-2001` | Transaction processed successfully |
| `HOG-2002` | Transaction failed, retrying |
| `InvalidArgument` | Validation error in request |
| `Internal` | Internal server error or max retry reached |

**Error Scenarios**:

1. **Validation Error** (InvalidArgument)
   - Missing required fields
   - Invalid datetime format
   - Invalid payment_method

2. **Config Service Error** (Internal)
   - Failed to get operational city mapping
   - iTOP ID not found

3. **Database Error** (Internal)
   - Failed to create transaction record
   - Failed to create attempt record

4. **HO Service Error** (Internal → Retry)
   - HO system returns non-success response
   - Network timeout
   - Connection refused

5. **Max Retry Reached** (Internal)
   - All retry attempts exhausted
   - Returns error to caller

**Processing Flow**:

```
1. Validate request
2. Parse transaction datetime
3. Get operational city from Config Service (iTOP ID → Area Code)
4. If attempt_count == 0:
   - Create ho_transaction record
   - Create ho_transaction_attempt record (attempt #0)
   Else:
   - Get existing ho_transaction_attempt
   - Update retry_count
5. Send request to HO Service
6. Update ho_transaction_attempt with response
7. If failed AND retry < max_attempt:
   - Sleep 1 second
   - Republish to RabbitMQ with attempt_count++
   - Return HOG-2002 (Retry)
8. If failed AND retry >= max_attempt:
   - Return error "Max attempt reached"
9. If success:
   - Return HOG-2001 (Success)
```

**Database Records Created**:

**ho_transactions** (once per order):
```sql
INSERT INTO ho_transactions (
    order_id, bbid, approval_code, bank,
    amount, discount_amount, promo_code
) VALUES (
    789012, '081234567890', 'TXN-GOPAY-123', 'GoPay',
    150000, 10000, 'PROMO10'
);
```

**ho_transaction_attempts** (once per attempt):
```sql
INSERT INTO ho_transaction_attempts (
    order_id, retry_count, request_data, 
    response_data, is_success
) VALUES (
    789012, 0, '{"TrxType":"RSV_DP",...}',
    '{"code":200,"TrxID":"HO123",...}', true
);
```

**Usage**:
- Called via RabbitMQ consumer
- Triggered by Payment Processor (UPG) webhook
- Should NOT be called directly via gRPC

**Testing**:
- Set `transaction_id = "TEST"` to bypass actual HO call
- Returns mock success response

---

### 3. PublishMessage

**Purpose**: Publish message to specified topic

**Request**: `PublishMessageRequest`
```protobuf
message PublishMessageRequest {
    string topic = 1;     // Topic name to publish to
    string message = 2;   // Message content (JSON string)
}
```

**Response**: `PublishMessageResponse`
```protobuf
message PublishMessageResponse {
    string status = 1;    // Status: "success" or "failed"
    string message = 2;   // Status message
}
```

**Example Request**:
```json
{
  "topic": "ho_e_wallet_transaction",
  "message": "{\"order_id\":123,\"payment_method\":\"gopay\"}"
}
```

**Example Response**:
```json
{
  "status": "success",
  "message": "Message published successfully"
}
```

**Available Topics**:
- `ho_e_wallet_transaction` - E-wallet transaction events

**Usage**:
- Internal message publishing
- Manual republish for debugging
- Testing message flow

---

### 4. SendMessage

**Purpose**: Send direct message (implementation-specific)

**Request**: `SendMessageRequest`
```protobuf
message SendMessageRequest {
    string message = 1;   // Message content
}
```

**Response**: `SendMessageResponse`
```protobuf
message SendMessageResponse {
    string status = 1;    // Status: "success" or "failed"
    string message = 2;   // Status message
}
```

**Example Request**:
```json
{
  "message": "Test message"
}
```

**Example Response**:
```json
{
  "status": "success",
  "message": "Message sent successfully"
}
```

**Usage**:
- Direct messaging
- Testing purposes

---

## REST API Endpoints

Service also exposes REST API via gRPC-Gateway:

**Base URL**: `http://<host>:8082`

### Endpoints

| Method | Path | gRPC Method |
|--------|------|-------------|
| `GET` | `/health` | HealthCheck |
| `POST` | `/v1/publish` | PublishMessage |
| `POST` | `/v1/send` | SendMessage |
| `POST` | `/v1/transaction/ewallet` | HoEWalletTransaction |

### Example REST Call

**POST /v1/transaction/ewallet**

```bash
curl -X POST http://localhost:8082/v1/transaction/ewallet \
  -H "Content-Type: application/json" \
  -d '{
    "payment_method": "gopay",
    "trx_date_time": "2025-01-08T10:30:00+07:00",
    "itop_id": "JKT01",
    "external_order_id": "ORD-123456",
    "order_id": 789012,
    "currency": "IDR",
    "amount": 150000,
    "discount_amount": 10000,
    "promo_code": "PROMO10",
    "bbid": "081234567890",
    "transaction_id": "TXN-GOPAY-123",
    "bank": "GoPay",
    "platform_fee": 2000,
    "sustainability": 1000,
    "attempt_count": 0
  }'
```

---

## HO Service Integration

### HO E-Wallet API

**Endpoint**: Configured via `HO_SERVICE_URL`  
**Method**: `POST`  
**Content-Type**: `application/json`

**Request Format** (sent to HO):
```json
{
  "TrxType": "RSV_DP",
  "TrxDate": "20250108",
  "TrxTime": "103000",
  "TrxStat": "1",
  "Area": "JKT",
  "OrderServerId": "JKT01",
  "OrderId": "ORD-123456",
  "TrxRef1": "789012",
  "AppCode": "LMO",
  "Currency": "IDR",
  "JumlahDP": 150000,
  "Amount": 143000,
  "DiscountAmount": 10000,
  "PromoCode": "PROMO10",
  "PhoneNo": "081234567890",
  "ApprovalCode": "TXN-GOPAY-123",
  "Bank": "GoPay",
  "PayGateway": "MIDGPY",
  "PlatformFee": 2000,
  "Sustainability": 1000
}
```

**Field Transformations**:
```
TrxDate = trx_date_time.Format("20060102")
TrxTime = trx_date_time.Format("150405")
Area = operational_city.area_code (from Config Service)
TrxRef1 = order_id (as string)
Amount = amount - discount_amount + platform_fee + sustainability
PayGateway = PayGatewayMap[payment_method]
```

**Response Format** (from HO):
```json
{
  "code": 200,
  "TrxID": "HO-TXN-123",
  "error": "",
  "Message": "Success"
}
```

**Success Criteria**:
```go
isSuccess := response.TrxID != "" && 
             (response.Error == "" || response.Message == "") && 
             response.Code == 200
```

---

## Message Queue Integration

### Topic: ho_e_wallet_transaction

**Message Format**: Same as `HoEWalletRequest`

**Consumer**:
- Service: HO Gateway
- Handler: `usecase.HoEWalletTransaction`
- Auto-retry on failure with attempt_count increment

**Publisher**:
- Service: Payment Processor (UPG)
- Trigger: Payment webhook success
- Service: HO Gateway (for retry)
- Trigger: HO service failure + retry available

---

## Error Handling

### gRPC Status Codes

| gRPC Code | Scenario | HTTP Status |
|-----------|----------|-------------|
| `OK` | Success | 200 |
| `InvalidArgument` | Validation error | 400 |
| `Internal` | Server error | 500 |
| `Unavailable` | Service unavailable | 503 |

### Custom Response Codes

| Code | Meaning |
|------|---------|
| `HOG-1001` | Health check OK |
| `HOG-2001` | Transaction success |
| `HOG-2002` | Transaction retry |

---

## Related Documentation

- [[README|Service Overview]]
- [[dependencies|Dependencies]]
- [[orderorchestrator]] - Order creation flow
- [[paymentprocessor]] - Payment processing

---

*Last Updated*: 2025-01-08
