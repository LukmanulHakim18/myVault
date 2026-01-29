---
tags:
  - mrg
  - service
  - api
  - rest
  - order-processing
parent: '[[README]]'
created: '2026-01-29'
updated: '2026-01-29'
---
# Order Processing - API Reference

**Parent**: [[README|Order Processing Service]]

---

## 🌐 REST API

**Port**: `REST_API_PORT` (from environment)  
**Base URL**: `http://order-processing:{port}`

> ⚠️ **Note**: REST API adalah secondary transport, primarily untuk internal/debugging. Primary transport adalah Kafka.

---

## 📡 Endpoints

### Health Check

```http
GET /ping
```

**Response:**
```json
{
    "status": "ok"
}
```

---

### Create Order

```http
POST /orders
Content-Type: application/json
```

**Request Body:**
```json
{
    "user_id": "BB00123456",
    "pickup_latitude": -6.2088,
    "pickup_longitude": 106.8456,
    "pickup_address": "Jl. Sudirman No. 1",
    "dropoff_latitude": -6.1753,
    "dropoff_longitude": 106.8271,
    "dropoff_address": "Jl. Gatot Subroto",
    "service_type_id": 0,
    "payment_method": "cash",
    "is_on_street_ride": false,
    "note_to_driver": "Depan gedung tinggi"
}
```

**Response:**
```json
{
    "order_id": 12345,
    "state": "INITIAL",
    "pickup_address": "Jl. Sudirman No. 1",
    "dropoff_address": "Jl. Gatot Subroto",
    "service_type_id": 0,
    "payment_method": "cash",
    "created_at": "2026-01-29T10:00:00Z"
}
```

---

### Get Active Orders

```http
GET /active_orders
X-User-Id: BB00123456
```

**Response:**
```json
{
    "orders": [
        {
            "order_id": 12345,
            "state": "ON_TRIP",
            "service_type_id": 0,
            "payment_method": "cash",
            "driver_name": "John Doe",
            "car_number": "B 1234 CD",
            "trip_fare": 50000
        }
    ]
}
```

---

### Get Last Order

```http
GET /last_order
X-User-Id: BB00123456
```

**Response:**
```json
{
    "order_id": 12345,
    "state": "COMPLETED",
    "service_type_id": 0,
    "payment_method": "cash",
    "final_fare": 55000,
    "rating": 5,
    "created_at": "2026-01-29T10:00:00Z",
    "completed_at": "2026-01-29T11:00:00Z"
}
```

---

### Get Cancellation Fee

```http
GET /cancellation_fee/{order_id}
X-User-Id: BB00123456
```

**Parameters:**
| Name | Type | Location | Description |
|------|------|----------|-------------|
| order_id | int | path | Order ID |

**Response:**
```json
{
    "order_id": 12345,
    "cancellation_fee": 15000,
    "can_cancel": true,
    "reason": ""
}
```

---

### Get Order Info All

```http
GET /order_info/all
X-User-Id: BB00123456
X-Order-Id: 12345
```

**Response:**
```json
{
    "order_id": 12345,
    "order_info": {
        "pickup_place_id": "ChIJ...",
        "dropoff_place_id": "ChIJ...",
        "is_allow_tips": true,
        "identifier": "BB001",
        "voucher_id": "",
        "company_id": ""
    },
    "bbd_info": {
        "external_order_id": 67890,
        "itop_id": "IT001",
        "pass_code": "1234"
    },
    "upg_info": {
        "charge_status": "success",
        "transaction_id": "TXN001"
    }
}
```

---

### Get Order Info BBD

```http
GET /order_info/bbd
X-Order-Id: 12345
```

**Response:**
```json
{
    "order_id": 12345,
    "external_order_id": 67890,
    "itop_id": "IT001",
    "pass_code": "1234",
    "driver_id": "D001",
    "car_number": "B 1234 CD"
}
```

---

### Get Order Info UPG

```http
GET /order_info/upg
X-Order-Id: 12345
```

**Response:**
```json
{
    "order_id": 12345,
    "payment_method": "cc",
    "charge_status": "success",
    "transaction_id": "TXN001",
    "pre_auth_amount": 200000,
    "final_amount": 55000
}
```

---

### Get Order Info State

```http
GET /order_info/state
X-Order-Id: 12345
```

**Response:**
```json
{
    "order_id": 12345,
    "state": "ON_TRIP",
    "state_history": [
        {"state": "INITIAL", "timestamp": "2026-01-29T10:00:00Z"},
        {"state": "LOOKING_FOR_DRIVER", "timestamp": "2026-01-29T10:01:00Z"},
        {"state": "DRIVER_FOUND", "timestamp": "2026-01-29T10:05:00Z"},
        {"state": "ON_TRIP", "timestamp": "2026-01-29T10:10:00Z"}
    ]
}
```

---

### Update Order Info

```http
POST /order_info
Content-Type: application/json
X-User-Id: BB00123456
```

**Request Body:**
```json
{
    "order_id": 12345,
    "note_to_driver": "Updated note",
    "trip_purpose": "business"
}
```

**Response:**
```json
{
    "success": true,
    "message": "Order info updated"
}
```

---

### Update Order Subscription

```http
POST /order_subscription
Content-Type: application/json
```

**Request Body:**
```json
{
    "order_id": 12345,
    "subscription_status": "active",
    "subscription_title_en": "Monthly Pass",
    "subscription_title_id": "Paket Bulanan"
}
```

**Response:**
```json
{
    "success": true,
    "message": "Subscription updated"
}
```

---

### Check Order Upsell

```http
POST /order_upsell
Content-Type: application/json
X-User-Id: BB00123456
X-Metadata: {...}
```

**Request Body:**
```json
{
    "order_id": 12345,
    "current_service_type": 0
}
```

**Response:**
```json
{
    "eligible": true,
    "upsell_options": [
        {
            "service_type_id": 1,
            "service_name": "SilverBird",
            "price_difference": 15000,
            "eta_difference": -5
        }
    ]
}
```

---

### Update Rent Multidays

```http
POST /order/rent_multidays/update
Content-Type: application/json
```

**Request Body:**
```json
{
    "order_id": 12345,
    "rent_hours": 8,
    "pickup_time": "2026-01-30T08:00:00Z",
    "dropoff_time": "2026-01-30T16:00:00Z"
}
```

**Response:**
```json
{
    "success": true,
    "order_id": 12345,
    "updated_rent_hours": 8
}
```

---

### Update Rent Multidays Override

```http
POST /order/rent_multidays/update/override
Content-Type: application/json
```

**Request Body:**
```json
{
    "order_id": 12345,
    "rent_hours": 10,
    "override_reason": "Customer request",
    "approved_by": "admin@bluebird.id"
}
```

**Response:**
```json
{
    "success": true,
    "order_id": 12345,
    "updated_rent_hours": 10,
    "override_applied": true
}
```

---

## 🔧 Middleware

### Request ID Middleware
```go
// middlewareRequestID extracts request ID and trace ID from headers
func middlewareRequestID(next http.Handler) http.Handler {
    return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
        requestID := r.Header.Get("X-Request-Id")
        traceID := r.Header.Get("X-Trace-Id")
        ctx := context.WithValue(r.Context(), "request_id", requestID)
        ctx = context.WithValue(ctx, "trace_id", traceID)
        next.ServeHTTP(w, r.WithContext(ctx))
    })
}
```

### Metadata Middleware
```go
// middlewareMetadata parses metadata from headers
func middlewareMetadata(next http.Handler) http.Handler {
    return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
        metadata := r.Header.Get("X-Metadata")
        // Parse and validate metadata
        next.ServeHTTP(w, r)
    })
}
```

---

## 🔗 Related Documentation

- [[README|Order Processing Service]]
- [[dependencies|Dependencies]]
- [[order-flows|Order Flows]]
- [[kafka-topics|Kafka Topics Reference]]

---

#mrg #service #api #rest #order-processing

---

*Last Updated*: 2026-01-29
