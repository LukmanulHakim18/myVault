# API Reference

## Overview
OrderQuery service menyediakan RESTful API via gRPC Gateway dan native gRPC endpoints untuk querying order data. Semua endpoints (kecuali HealthCheck) require authentication dan metadata headers.

## Base URLs

### gRPC
```
grpc://orderquery:6005
```

### REST (via Gateway)
```
http://orderquery:8005
```

## Authentication

### Required Headers
All endpoints (except HealthCheck) require these metadata headers:

| Header | Description | Example | Required |
|--------|-------------|---------|----------|
| `x-app-version` | App version | `2.5.0` | Yes |
| `x-device-id` | Device UUID | `uuid-device-123` | Yes |
| `x-manufacturer` | Device manufacturer | `Samsung` | Yes |
| `x-device-type` | Device type | `android` or `ios` | Yes |
| `x-device-model` | Device model | `SM-G973F` | Yes |
| `x-device-time` | Device timestamp | `1704614400` | Yes |
| `x-os-version` | OS version | `13` | Yes |
| `x-operating-system` | OS name | `Android` | Yes |
| `x-user-info` | User info from auth | Base64 encoded user data | Yes |

**Note**: Headers are validated by `MetadataValidator` interceptor.

---

## API Categories

- [[#Health & System|Health & System]]
- [[#Order Queries|Order Queries]]
- [[#Trip Management|Trip Management]]
- [[#Group Orders (Shuttle)|Group Orders]]
- [[#Address & Location|Address & Location]]
- [[#Billing & Payment|Billing & Payment]]
- [[#Chat|Chat Integration]]
- [[#Reschedule|Reschedule Operations]]
- [[#Status Checks|Status Checks]]

---

## Health & System

### HealthCheck
Check service health status.

**gRPC**: `OrderQuery.HealthCheck`  
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

### RoutineOrderHistoryUpdate
Trigger manual order history update (typically run as cron job).

**gRPC**: `OrderQuery.RoutineOrderHistoryUpdate`  
**REST**: Not exposed via REST

**Request**:
```json
{}
```

**Response**:
```json
{}
```

---

## Order Queries

### GetOrders
Get paginated list of orders with filtering by category.

**gRPC**: `OrderQuery.GetOrders`  
**REST**: `POST /v1/orders`

**Request**:
```json
{
  "limit": 20,
  "category": "ride",  // "ride", "shuttle", "rent", "delivery"
  "cursor": "base64_encoded_cursor"  // for pagination
}
```

**Response**:
```json
{
  "metadata": {
    "count_data": 20,
    "has_outstanding": true,
    "next_cursor": "base64_next_cursor",
    "total_data": 150
  },
  "records": [
    {
      "id": "internal_id_123",
      "order_id": "ORDER123",
      "category": "ride",
      "category_icon": "https://cdn.bluebird.id/icons/ride.png",
      "product_type": "bluebird",
      "status": 5,  // order status code
      "is_rated": false,
      "car_number": "B 1234 ABC",
      "driver_name": "Pak Driver",
      "pickup_address": "Jl. Sudirman",
      "dropoff_address": "Jl. Thamrin",
      "final_fare": 150000,
      "payment_status": "paid",
      "created_at": 1704614400,
      // ... more fields
    }
  ]
}
```

---

### GetActiveOrders
Get currently active (in-progress) orders.

**gRPC**: `OrderQuery.GetActiveOrders`  
**REST**: `POST /v1/orders/active`

**Request**: Same as `GetOrders`

**Response**: Same format as `GetOrders`, but filtered to status = active

---

### GetUnratedOrders
Get completed orders that haven't been rated yet.

**gRPC**: `OrderQuery.GetUnratedOrders`  
**REST**: `POST /v1/orders/unrated`

**Request**: Same as `GetOrders`

**Response**: Same format as `GetOrders`, filtered to `is_rated: false` and `status: completed`

---

### GetLatestOrder
Get the most recent order for a user.

**gRPC**: `OrderQuery.GetLatestOrder`  
**REST**: `POST /v1/orders/latest`

**Request**:
```json
{
  "internal_user_id": "user_123",
  "category": "ride"  // optional: filter by category
}
```

**Response**:
```json
{
  "id": "internal_id_456",
  "order_id": "ORDER456",
  "category": "ride",
  "product_type": "bluebird",
  "status": 7,  // completed
  "car_number": "B 5678 DEF",
  "driver_name": "Pak Driver 2",
  "driver_phone": "+628123456789",
  "pickup_address": "Jl. Gatot Subroto",
  "dropoff_address": "Jl. Rasuna Said",
  "final_fare": 200000,
  "payment_status": "paid",
  "created_at": 1704700800,
  "dropoff_at": 1704702000
}
```

---

### GetSingleOrder
Get single order details by order ID.

**gRPC**: `OrderQuery.GetSingleOrder`  
**REST**: `POST /v1/orders/single`

**Request**:
```json
{
  "order_id": "12345",
  "internal_user_id": "user_123"
}
```

**Response**: Full `OrderDetail` object

---

### GetOrderById
Get order by internal ID.

**gRPC**: `OrderQuery.GetOrderById`  
**REST**: `POST /v1/orders/by-id`

**Request**:
```json
{
  "order_id": "internal_id_789"
}
```

**Response**: Full `OrderDetail` object

---

### GetOrderLegacyById
Get order from legacy database.

**gRPC**: `OrderQuery.GetOrderLegacyById`  
**REST**: `POST /v1/orders/legacy`

**Request**:
```json
{
  "order_id": "12345",
  "internal_user_id": "user_123"
}
```

**Response**: Full `OrderDetail` object from legacy DB

---

### GetBulkOrders
Get multiple orders in a single request.

**gRPC**: `OrderQuery.GetBulkOrders`  
**REST**: `POST /v1/orders/bulk`

**Request**:
```json
{
  "order_ids": [
    "ORDER123",
    "ORDER456",
    "ORDER789"
  ]
}
```

**Response**:
```json
{
  "orders": [
    {
      "order_id": "ORDER123",
      "order_detail": { /* full order detail */ }
    },
    {
      "order_id": "ORDER456",
      "order_detail": { /* full order detail */ }
    }
  ]
}
```

---

### GetOrderDetailsByOrderId
Get comprehensive order details including fees breakdown.

**gRPC**: `OrderQuery.GetOrderDetailsByOrderId`  
**REST**: `POST /v1/orders/details`

**Request**:
```json
{
  "order_id": "ORDER123"
}
```

**Response**: Extended `OrderDetail` with fees breakdown

---

## Trip Management

### GetTripHistory
Get historical trips (completed orders).

**gRPC**: `OrderQuery.GetTripHistory`  
**REST**: `POST /v1/orders/history`

**Request**:
```json
{
  "limit": 20,
  "category": "ride",
  "cursor": "base64_cursor"
}
```

**Response**: Same format as `GetOrders`, filtered to completed trips

---

### GetTripScheduled
Get future scheduled trips.

**gRPC**: `OrderQuery.GetTripScheduled`  
**REST**: `POST /v1/orders/scheduled`

**Request**: Same as `GetTripHistory`

**Response**: Same format as `GetOrders`, filtered to scheduled trips

---

### GetLastTripPurpose
Get the purpose of user's last trip.

**gRPC**: `OrderQuery.GetLastTripPurpose`  
**REST**: `POST /v1/orders/last-trip-purpose`

**Request**:
```json
{
  "internal_user_id": "user_123"
}
```

**Response**:
```json
{
  "trip_purpose": "business"  // or "personal", "leisure", etc.
}
```

---

## Group Orders (Shuttle)

### GetListOrder
Get list of group orders for shuttle/Cititrans.

**gRPC**: `OrderQuery.GetListOrder`  
**REST**: `POST /v1/orders/list`

**Request**:
```json
{
  "internal_user_id": "user_123",
  "status": "active",  // "active", "schedule", "history"
  "category": "shuttle"
}
```

**Response**:
```json
{
  "records": [
    {
      "internal_user_id": "user_123",
      "category": "shuttle",
      "order_id": 12345,
      "price": 150000,
      "status": "active",
      "title": "{\"en\":\"Jakarta to Bandung\",\"id\":\"Jakarta ke Bandung\"}",
      "description": "{\"en\":\"Executive shuttle\",\"id\":\"Shuttle executive\"}",
      "order_date": "2025-01-10T08:00:00Z",
      "outstanding": 0,
      "customer_email": "user@example.com",
      "product_type": "cititrans_executive",
      "is_delete_price": false
    }
  ]
}
```

---

### UpsertGroupOrderDetail
Create or update group order details (batch operation).

**gRPC**: `OrderQuery.UpsertGroupOrderDetail`  
**REST**: `POST /v1/group-orders/upsert`

**Request**:
```json
{
  "group_order_details": [
    {
      "internal_user_id": "user_123",
      "category": "shuttle",
      "order_id": 12345,
      "price": 150000,
      "status": "active",
      "title": "{\"en\":\"Jakarta to Bandung\",\"id\":\"Jakarta ke Bandung\"}",
      "description": "{\"en\":\"Executive\",\"id\":\"Eksekutif\"}",
      "order_date": "2025-01-10T08:00:00Z",
      "product_type": "cititrans_executive"
    }
  ]
}
```

**Response**:
```json
{
  "success": true,
  "message": "Group order details upserted successfully"
}
```

---

### RescheduleGroupOrderDetail
Reschedule a group order item.

**gRPC**: `OrderQuery.RescheduleGroupOrderDetail`  
**REST**: `POST /v1/group-orders/reschedule`

**Request**:
```json
{
  "order_id": 12345,
  "internal_user_id": "user_123",
  "new_order_date": "2025-01-15T08:00:00Z"
}
```

**Response**:
```json
{
  "id": 67890  // new order ID
}
```

---

### DeleteGroupOrderDetail
Delete a group order item.

**gRPC**: `OrderQuery.DeleteGroupOrderDetail`  
**REST**: `POST /v1/group-orders/delete`

**Request**:
```json
{
  "order_id": 12345,
  "internal_user_id": "user_123"
}
```

**Response**:
```json
{
  "success": true,
  "deleted_count": 1
}
```

---

### CreateGroupOrderDetailMetadata
Create metadata for a group order.

**gRPC**: `OrderQuery.CreateGroupOrderDetailMetadata`  
**REST**: `POST /v1/group-orders/metadata`

**Request**:
```json
{
  "order_id": 12345,
  "metadata": {
    "custom_field_1": "value1",
    "custom_field_2": "value2"
  }
}
```

**Response**:
```json
{
  "id": 111  // metadata ID
}
```

---

### GetGroupOrderDetailMetadata
Get metadata for a group order.

**gRPC**: `OrderQuery.GetGroupOrderDetailMetadata`  
**REST**: `POST /v1/group-orders/metadata/get`

**Request**:
```json
{
  "order_id": 12345
}
```

**Response**:
```json
{
  "order_id": 12345,
  "metadata": {
    "custom_field_1": "value1",
    "custom_field_2": "value2"
  }
}
```

---

## Address & Location

### GetFrequentlyAddress
Get frequently used addresses from order history.

**gRPC**: `OrderQuery.GetFrequentlyAddress`  
**REST**: `POST /v1/addresses/frequent`

**Request**:
```json
{
  "internal_user_id": "user_123",
  "limit": 5
}
```

**Response**:
```json
{
  "addresses": [
    {
      "address": "Jl. Sudirman No. 123",
      "latitude": -6.2088,
      "longitude": 106.8456,
      "usage_count": 15
    },
    {
      "address": "Jl. Thamrin No. 456",
      "latitude": -6.1944,
      "longitude": 106.8229,
      "usage_count": 10
    }
  ]
}
```

---

### GetPickupInstructionSuggest
Get pickup instruction suggestions for a user.

**gRPC**: `OrderQuery.GetPickupInstructionSuggest`  
**REST**: `POST /v1/pickup-instructions/suggest`

**Request**:
```json
{
  "internal_user_id": "user_123"
}
```

**Response**:
```json
{
  "suggestions": [
    {
      "id": 1,
      "instruction": "Depan lobby utama",
      "usage_count": 20
    },
    {
      "id": 2,
      "instruction": "Parkir basement B1",
      "usage_count": 15
    }
  ]
}
```

---

### DeletePickupInstructionSuggest
Delete a pickup instruction suggestion.

**gRPC**: `OrderQuery.DeletePickupInstructionSuggest`  
**REST**: `POST /v1/pickup-instructions/delete`

**Request**:
```json
{
  "id": 1,
  "internal_user_id": "user_123"
}
```

**Response**:
```json
{}
```

---

## Billing & Payment

### GetBillingDetails
Get detailed billing breakdown for an order.

**gRPC**: `OrderQuery.GetBillingDetails`  
**REST**: `POST /v1/orders/billing`

**Request**:
```json
{
  "order_id": "12345",
  "internal_user_id": "user_123"
}
```

**Response**:
```json
{
  "admin_fee": 5000,
  "cancellation_fee": 0,
  "discount_value": 10000,
  "dropoff_at": "2025-01-07T11:30:00Z",
  "extra_time_fee": 0,
  "extra_value": 0,
  "final_fare": 145000,
  "handling_fee": 5000,
  "order_id": "12345",
  "outstanding": 0,
  "paid_amount": 145000,
  "payment_display_name": "Credit Card",
  "payment_method": "credit_card",
  "payment_status": "paid",
  "promotion_code": "PROMO10",
  "status": 7,  // completed
  "tips": 10000,
  "trip_fare": 140000,
  "trip_purpose": "business"
}
```

---

### GetCancellationFee
Calculate cancellation fee for an order.

**gRPC**: `OrderQuery.GetCancellationFee`  
**REST**: `POST /v1/orders/cancellation-fee`

**Request**:
```json
{
  "order_id": "12345"
}
```

**Response**:
```json
{
  "cancellation_fee": 25000,
  "reason": "Late cancellation (within 2 hours of pickup)"
}
```

---

### HasOrderWithOutstandingPayment
Check if user has any orders with outstanding payment.

**gRPC**: `OrderQuery.HasOrderWithOutstandingPayment`  
**REST**: `POST /v1/orders/has-outstanding`

**Request**:
```json
{}
```

**Response**:
```json
{
  "has_outstanding": true,
  "outstanding_amount": 50000,
  "order_ids": ["ORDER123", "ORDER456"]
}
```

---

### GetTipsDriverByOrderId
Get driver tips information for an order.

**gRPC**: `OrderQuery.GetTipsDriverByOrderId`  
**REST**: `POST /v1/orders/tips`

**Request**:
```json
{
  "order_id": "ORDER123"
}
```

**Response**:
```json
{
  "order_id": "ORDER123",
  "tips_amount": 10000,
  "tips_percentage": 10,
  "driver_id": "DRV789"
}
```

---

### RechargeWithOtherPaymentPostEvent
Handle post-event for recharge with alternative payment method.

**gRPC**: `OrderQuery.RechargeWithOtherPaymentPostEvent`  
**REST**: `POST /v1/payments/recharge-post-event`

**Request**:
```json
{
  "order_id": "ORDER123",
  "payment_method": "gopay",
  "amount": 50000
}
```

**Response**:
```json
{
  "success": true,
  "transaction_id": "TRX789"
}
```

---

## Chat

### GetChatRoomId
Get chat room ID for driver communication.

**gRPC**: `OrderQuery.GetChatRoomId`  
**REST**: `POST /v1/chat/room`

**Request**:
```json
{
  "order_id": "12345",
  "internal_user_id": "user_123"
}
```

**Response**:
```json
{
  "chat_room_id": "room_abc123",
  "driver_id": "DRV789",
  "order_status": 3  // in-progress
}
```

**Error Responses**:
```json
// Order already closed
{
  "error_code": "ODQR-4004",
  "message": {
    "english": "Cannot get room ID - order already closed",
    "indonesia": "Tidak dapat mendapatkan room ID - pesanan sudah ditutup"
  }
}
```

---

## Reschedule

### CheckEligibleReschedule
Check if an order is eligible for rescheduling.

**gRPC**: `OrderQuery.CheckEligibleReschedule`  
**REST**: `POST /v1/reschedule/check-eligible`

**Request**:
```json
{
  "order_id": "12345",
  "internal_user_id": "user_123"
}
```

**Response**:
```json
{
  "is_eligible": true,
  "reason": "",
  "min_reschedule_time": "2025-01-08T08:00:00Z",
  "max_reschedule_time": "2025-01-15T08:00:00Z"
}
```

**Not Eligible Response**:
```json
{
  "is_eligible": false,
  "reason": "Order is too close to pickup time (less than 4 hours)",
  "min_reschedule_time": null,
  "max_reschedule_time": null
}
```

---

### GetRescheduleConfirmation
Get reschedule confirmation with new fare calculation.

**gRPC**: `OrderQuery.GetRescheduleConfirmation`  
**REST**: `POST /v1/reschedule/confirmation`

**Request**:
```json
{
  "order_id": "12345",
  "internal_user_id": "user_123",
  "new_pickup_time": "2025-01-10T08:00:00Z"
}
```

**Response**:
```json
{
  "original_fare": 150000,
  "new_fare": 165000,
  "fare_difference": 15000,
  "reason": "Peak hour surcharge applied",
  "confirmation_required": true
}
```

---

## Status Checks

### HasActiveOrder
Check if user has any active (in-progress) orders.

**gRPC**: `OrderQuery.HasActiveOrder`  
**REST**: `POST /v1/status/has-active`

**Request**:
```json
{}
```

**Response**:
```json
{
  "has_active": true,
  "active_order_ids": ["ORDER123"],
  "active_order_count": 1
}
```

---

## Error Responses

### Error Format
```json
{
  "error_code": "ODQR-4001",
  "status_code": 400,
  "localized_message": {
    "english": "Missing required parameter: order_id",
    "indonesia": "Parameter wajib tidak ada: order_id"
  }
}
```

### Error Code Reference

| Code | HTTP | English Message | Indonesia Message |
|------|------|-----------------|-------------------|
| ODQR-4001 | 400 | Missing required parameter | Parameter wajib tidak ada |
| ODQR-4002 | 400 | Invalid parameter value | Nilai parameter tidak valid |
| ODQR-4003 | 400 | Not available for this product type | Tidak tersedia untuk tipe produk ini |
| ODQR-4004 | 400 | Cannot get room ID - order closed | Tidak dapat mendapatkan room ID - pesanan ditutup |
| ODQR-4005 | 400 | Product type not supported | Tipe produk tidak didukung |
| ODQR-4041 | 404 | Order not found | Pesanan tidak ditemukan |
| ODQR-5000 | 500 | Internal server error | Kesalahan server internal |
| ODQR-5001 | 500 | Delete pickup instruction failed | Gagal menghapus instruksi pickup |

---

## Pagination

### Cursor-Based Pagination
All list endpoints use cursor-based pagination:

```json
{
  "limit": 20,
  "cursor": "base64_encoded_cursor"
}
```

**Response includes next cursor**:
```json
{
  "metadata": {
    "count_data": 20,
    "next_cursor": "base64_next_cursor",
    "total_data": 150
  },
  "records": [...]
}
```

**Last page indicator**: `next_cursor` is empty string or null.

---

## Rate Limiting

- **Default**: 100 requests per minute per user
- **Burst**: 200 requests per minute
- **Headers**: Rate limit info in response headers

```
X-RateLimit-Limit: 100
X-RateLimit-Remaining: 95
X-RateLimit-Reset: 1704614460
```

---

## Related Documentation
- [[01-overview|Service Overview]]
- [[03-group-order-flows|Group Order Flows]]
- [[04-dependencies|Dependencies]]
- [Swagger UI](http://orderquery:8005/swagger)
- [Protobuf Definition](contract/order_query.proto)

---
**Last Updated**: 2025-01-07
**API Version**: v1
**Protocol**: gRPC + REST Gateway