# API Reference - Taxi Order Processor

## Overview
TOP service menyediakan comprehensive API untuk order lifecycle management dari creation hingga completion. Service menggunakan gRPC protocol dengan REST Gateway support.

## Base URLs

### gRPC
```
grpc://top:50051
```

### REST (via Gateway)
```
http://top:8051
```

## Authentication

### Required Headers
All endpoints (except HealthCheck) require authentication dan metadata headers:

| Header | Description | Example | Required |
|--------|-------------|---------|----------|
| `x-user-info` | User info from auth | Base64 encoded user data | Yes |
| `x-app-version` | App version | `2.5.0` | Yes |
| `x-device-type` | Device type | `android` or `ios` | Yes |
| `x-device-uuid` | Device UUID | `uuid-device-123` | Yes |
| `x-operating-system` | OS name | `Android` | Yes |
| `x-device-model` | Device model | `SM-G973F` | Yes |
| `x-manufacturer` | Manufacturer | `Samsung` | Yes |
| `x-token` | Auth token | JWT token | Yes |

---

## API Categories

- [[#Health & System|Health & System]]
- [[#Order Creation|Order Creation]]
- [[#Order Lifecycle|Order Lifecycle (Callbacks & Events)]]
- [[#Order Cancellation|Order Cancellation]]
- [[#Rating & Feedback|Rating & Feedback]]
- [[#Payment Operations|Payment Operations]]
- [[#Reschedule Operations|Reschedule Operations]]
- [[#Retry & Recovery|Retry & Recovery]]
- [[#Maintenance|Maintenance Checks]]

---

## Health & System

### HealthCheck
Check service health status.

**gRPC**: `TaxiOrderProcessor.HealthCheck`  
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

## Order Creation

### CreateOrder
Create new taxi order dengan comprehensive validation.

**gRPC**: `TaxiOrderProcessor.CreateOrder`  
**REST**: `POST /v1/orders`

**Pre-Creation Validations**:
- ✅ User soft ban check
- ✅ Payment method availability
- ✅ Service maintenance window
- ✅ Maximum concurrent bookings (5 future bookings)
- ✅ Area coverage validation
- ✅ Promo code validation
- ✅ Pre-authorization (credit card)

**Request**:
```json
{
  "booking_type": "now",  // "now" or "schedule"
  "car_type": "bluebird",
  "pickup": {
    "location": {
      "latitude": -6.2088,
      "longitude": 106.8456
    },
    "address": "Jl. Sudirman No. 123",
    "instruction": "Depan lobby utama",
    "city": "Jakarta"
  },
  "drop_off": {
    "location": {
      "latitude": -6.1944,
      "longitude": 106.8229
    },
    "address": "Jl. Thamrin No. 456",
    "city": "Jakarta"
  },
  "service_type": {
    "service_type_id": 1,  // 1=BlueBird, 2=SilverBird, 3=GoldenBird
    "package_id": 0
  },
  "payment_method": "credit_card",  // cash, gopay, dana, credit_card, etc
  "payment_identifier": "card_token_abc123",  // for non-cash
  "epayment": {
    "type": "credit_card",
    "identifier": "card_token_abc123",
    "display_name": "Visa *1234"
  },
  "promotion_code": "PROMO10",  // optional
  "trip_purpose": "business",
  "request_pickup_at": "2025-01-10T08:00:00Z",  // for schedule booking
  "note_to_driver": "Please call on arrival",
  "total_passengers": 2,
  "is_fixed_fare": false,
  "fare": {
    "est_min": 45000,
    "est_max": 65000,
    "fixed": 0,
    "locked_fare": 0
  },
  "vendor_chat": "Qiscus",  // chat provider
  "bbd_service_id": 1,
  "bbd_pkg_id": 0
}
```

**Response** (Success):
```json
{
  "order_id": 123456,
  "external_order_id": 789012,
  "display_order_id": "BBD-123456",
  "state": -2,  // ORDER_INITIAL
  "status": 0,
  "booking_type": "now",
  "car_type": "bluebird",
  "service_type": 1,
  "service_type_name": "BlueBird",
  "pickup": {
    "location": {
      "latitude": -6.2088,
      "longitude": 106.8456
    },
    "address": "Jl. Sudirman No. 123",
    "instruction": "Depan lobby utama"
  },
  "dropoff": {
    "location": {
      "latitude": -6.1944,
      "longitude": 106.8229
    },
    "address": "Jl. Thamrin No. 456"
  },
  "estimated_fare": {
    "min": 45000,
    "max": 65000
  },
  "payment_method": "credit_card",
  "payment_display_name": "Visa *1234",
  "payment_status": "pending",
  "charge_status": "not_charged",
  "trip_purpose": "business",
  "created_at": 1704614400,
  "pickup_time": "2025-01-10T08:00:00Z",
  "pass_code": "1234",  // for driver verification
  "trip_id": "trip_uuid_123"
}
```

**Error Responses**:
```json
// Soft ban error
{
  "code": "TOP-4033",
  "message": "You have been temporarily blocked from booking",
  "localized_message": {
    "en": "You have been temporarily blocked from booking due to multiple cancellations",
    "id": "Anda diblokir sementara karena terlalu banyak pembatalan"
  }
}

// Payment unavailable
{
  "code": "TOP-40010",
  "message": "Payment method unavailable"
}

// Maintenance window
{
  "code": "TOP-5000",
  "message": "Service under maintenance"
}

// Max bookings reached
{
  "code": "TOP-4004",
  "message": "Maximum future bookings reached (5)"
}
```

**State Transition**:
```
Initial → ORDER_INITIAL (-2) → LOOKING_FOR_DRIVER (0)
```

---

## Order Lifecycle

### CallbackOrder
Handle callbacks dari Taxi Partner Gateway (iTOP) untuk state updates.

**gRPC**: `TaxiOrderProcessor.CallbackOrder`  
**REST**: `POST /v1/orders/callback`

**Called By**: Taxi Partner Gateway (iTOP)

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
    "status": 1,  // EN_ROUTE
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
  "track": {
    "location": {
      "latitude": -6.2000,
      "longitude": 106.8300
    }
  },
  "state": 1  // EN_ROUTE
}
```

**Response**:
```json
{
  "code": "200",
  "message": "Order updated successfully"
}
```

**Processed States**:
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

### OrderEvent
Handle internal order events untuk state transitions.

**gRPC**: `TaxiOrderProcessor.OrderEvent`  
**REST**: `POST /v1/orders/events`

**Request**:
```json
{
  "meta_data": {
    "trace_id": "trace_123",
    "user_info": {
      "internal_id": "user_123",
      "name": "John Doe",
      "email": "john@example.com"
    }
  },
  "order_id": 123456,
  "event_state": 1,  // EN_ROUTE
  "bbd_service_id": 1,
  "bbd_pkg_id": 0
}
```

**Response**:
```json
{
  "code": "200",
  "message": "Event processed"
}
```

---

## Order Cancellation

### CancelOrder (by User)
User-initiated order cancellation.

**gRPC**: `TaxiOrderProcessor.CancelOrder`  
**REST**: `POST /v1/orders/cancel`

**Request**:
```json
{
  "order_id": "123456",
  "comment": "Driver taking too long"
}
```

**Response**:
```json
{
  "code": "200",
  "message": "Order cancelled successfully"
}
```

**Soft Ban Mechanism**:
```
IF user cancels ≥ 5 orders within 2 hours
THEN soft ban for 24 hours
```

**Allowed States for Cancellation**:
- LOOKING_FOR_DRIVER (0)
- EN_ROUTE (1)

---

### CancelOrderBySystem
System-initiated cancellation (e.g., timeout, no driver).

**gRPC**: `TaxiOrderProcessor.CancelOrderBySystem`  
**REST**: `POST /v1/orders/cancel-system`

**Request**:
```json
{
  "event": {
    "id": "evt_456",
    "type": "timeout",
    "time": "2025-01-07T10:15:00Z"
  },
  "order": {
    "order_id": 123456,
    "status": 2,  // CANCELED
    "request_id": "req_def"
  },
  "cancel": {
    "cancel_by": 2,  // 1=customer, 2=system, 3=driver
    "reason_type": 1,  // timeout
    "reason_text": "No driver available",
    "cancel_time": "2025-01-07T10:15:00Z"
  },
  "state": 2  // CANCELED
}
```

**Response**:
```json
{
  "code": "200",
  "message": "Order cancelled by system"
}
```

---

## Rating & Feedback

### RateOrder
Rate completed order dengan optional tips.

**gRPC**: `TaxiOrderProcessor.RateOrder`  
**REST**: `POST /v1/orders/rate`

**Request**:
```json
{
  "order_id": 123456,
  "rating": 5,  // 1-5 stars
  "tips": 10000,  // optional, minimum 1000 IDR
  "comment": "Great service!"  // optional
}
```

**Response**:
```json
{
  "order_id": 123456,
  "rating": 5,
  "tips": 10000,
  "comment": "Great service!",
  "is_rated": true
}
```

**Validations**:
- Rating: 1-5 (required)
- Tips: ≥ 1000 IDR (optional)
- Can only rate terminal states (DROP_OFF, ECV_DROP_OFF)

**Error Responses**:
```json
// Invalid rating
{
  "code": "TOP-9022",
  "message": "Rating must be between 1 and 5"
}

// Tips below minimum
{
  "code": "TOP-9020",
  "message": "Tips must be at least Rp 1.000"
}

// Tips exceeds maximum
{
  "code": "TOP-9021",
  "message": "Tips exceeds maximum allowed"
}

// Order not eligible for rating
{
  "code": "TOP-4005",
  "message": "Order cannot be rated in current state"
}
```

---

### HideTrip
Hide trip from history (soft delete from user view).

**gRPC**: `TaxiOrderProcessor.HideTrip`  
**REST**: `POST /v1/orders/hide`

**Request**:
```json
{
  "order_id": 123456
}
```

**Response**:
```json
{
  "code": "200",
  "message": "Trip hidden successfully"
}
```

**Note**: Order masih ada di database, hanya hidden dari user UI.

---

## Payment Operations

### AdjustChargeStatus
Adjust payment charge status (dari payment processor callback).

**gRPC**: `TaxiOrderProcessor.AdjustChargeStatus`  
**REST**: `POST /v1/payments/adjust-charge`

**Request**:
```json
{
  "user_id": "user_123",
  "payment_method": "gopay",
  "order_id": 123456,
  "charge_status": "success",  // "charging", "success", "failed"
  "failure_reason": "",  // if failed
  "outstanding": 0,
  "trip_fare": 55000,
  "final_fare": 55000
}
```

**Response**:
```json
{
  "code": "200",
  "message": "Charge status updated"
}
```

---

### UpdateChargeStatus
Update charge status dari payment processor.

**gRPC**: `TaxiOrderProcessor.UpdateChargeStatus`  
**REST**: `POST /v1/payments/update-charge`

**Request**:
```json
{
  "order_id": 123456,
  "charge_status": "success",
  "final_fare": 55000,
  "outstanding": 0
}
```

**Response**:
```json
{
  "code": "200",
  "message": "Charge status updated"
}
```

---

### PreAuthOrderCallback
Handle pre-authorization callback dari payment processor.

**gRPC**: `TaxiOrderProcessor.PreAuthOrderCallback`  
**REST**: `POST /v1/payments/preauth-callback`

**Request**:
```json
{
  "order_id": 123456,
  "pre_auth_status": "approved",  // "approved", "declined"
  "pre_auth_amount": 70000,
  "payment_identifier": "card_token_abc123",
  "failure_reason": ""  // if declined
}
```

**Response**:
```json
{
  "code": "200",
  "message": "Pre-auth status updated"
}
```

**Error Handling**:
```json
// Pre-auth declined
{
  "code": "TOP-4008",
  "message": "Payment pre-authorization declined"
}
```

---

## Reschedule Operations

### Reschedule
Reschedule future booking dengan validations.

**gRPC**: `TaxiOrderProcessor.Reschedule`  
**REST**: `POST /v1/orders/reschedule`

**Pre-Reschedule Validations**:
- ✅ Order belum pernah di-reschedule
- ✅ New schedule ≥ 30 minutes dari sekarang
- ✅ E-wallet balance sufficient (for e-wallet payment)
- ✅ ECV policy validation (for ECV voucher)

**Request**:
```json
{
  "order_id": 123456,
  "bbid": "user_123",
  "new_pickup_time": "2025-01-10T10:00:00Z"
}
```

**Process**:
```
1. Get reschedule session (from SessionManager)
2. Validate reschedule eligibility:
   - Check if already rescheduled
   - Validate new pickup time (≥ 30 min from now)
   - Check e-wallet balance (if applicable)
   - Validate ECV policy (if ECV voucher)
3. Call iTOP Reschedule API
4. Update order in database:
   - Update waited_car_time
   - Update trip_fare (if fixed fare changed)
   - Mark as rescheduled in order_info
5. Send push notification
```

**Response**:
```json
{
  "order_id": 123456,
  "pickup_time": "2025-01-10T10:00:00Z",
  "trip_fare": 60000,
  "message": "Order rescheduled successfully"
}
```

**Error Responses**:
```json
// Already rescheduled
{
  "code": "TOP-4005",
  "message": "Order has already been rescheduled",
  "localized_title": "Already Rescheduled",
  "localized_description": "This order has already been rescheduled once"
}

// Time exceeded
{
  "code": "TOP-4005",
  "message": "Reschedule time exceeded",
  "localized_title": "Sorry, reschedule time has passed",
  "localized_description": "You have exceeded the maximum reschedule period of 30 minutes before pick-up time",
  "reschedule_duration_en": "30 minutes",
  "reschedule_duration_id": "30 menit"
}

// Insufficient balance
{
  "code": "TOP-4093",
  "message": "Insufficient e-wallet balance",
  "localized_title": "Insufficient balance",
  "localized_description": "Your balance is insufficient by Rp 15.000. Please top up to continue",
  "underpayment_amount": 15000
}

// ECV not applicable
{
  "code": "TOP-40011",
  "message": "ECV voucher not applicable for new date",
  "localized_description": "Your voucher cannot be used on the selected date"
}
```

**Allowed States**:
- LOOKING_FOR_DRIVER (0)

**Reschedule Limits**:
- Can only reschedule once per order
- Minimum 30 minutes before pickup time
- Maximum reschedule window configurable (default 30 minutes)

---

## Retry & Recovery

### RetryOrder
Retry failed order (resubmit ke iTOP).

**gRPC**: `TaxiOrderProcessor.RetryOrder`  
**REST**: `POST /v1/orders/retry`

**Request**:
```json
{
  "order_id": 123456
}
```

**Process**:
```
1. Validate order state (must be terminal failure state)
2. Reset order state to LOOKING_FOR_DRIVER
3. Resubmit to iTOP
4. Update order in database
```

**Response**:
```json
{
  "order_id": 123456,
  "state": 0,  // LOOKING_FOR_DRIVER
  "message": "Order retry initiated"
}
```

**Allowed States**:
- ORDER_FAILED (-1)
- ORDER_TIMEOUT (-3)
- NO_TAXI (3)

---

### CompleteEndtripTimeout
Force complete trip yang timeout (tidak ada update dari driver).

**gRPC**: `TaxiOrderProcessor.CompleteEndtripTimeout`  
**REST**: `POST /v1/orders/complete-endtrip-timeout`

**Request**:
```json
{
  "external_user_id": "user_123",
  "product_type": "bluebird",
  "order_id": 123456
}
```

**Response**:
```json
{
  "code": "200",
  "message": "Order marked as completed due to timeout"
}
```

---

## Sync & Update

### SyncOrderSummary
Sync order data ke order_summary table.

**gRPC**: `TaxiOrderProcessor.SyncOrderSummary`  
**REST**: `POST /v1/orders/sync-summary`

**Request**:
```json
{
  "id": 123456
}
```

**Response**:
```json
{
  "code": "200",
  "message": "Order summary synced"
}
```

---

### UpdateOrderByOrderIdAndUserIdToFirebase
Update order ke Firebase untuk real-time updates.

**gRPC**: `TaxiOrderProcessor.UpdateOrderByOrderIdAndUserIdToFirebase`  
**REST**: `POST /v1/orders/update-firebase`

**Request**:
```json
{
  "order_id": 123456,
  "user_id": "user_123"
}
```

**Response**:
```json
{
  "code": "200",
  "message": "Firebase updated"
}
```

---

### SetMissingItopID
Set missing iTOP ID untuk order lama.

**gRPC**: `TaxiOrderProcessor.SetMissingItopID`  
**REST**: `POST /v1/orders/set-missing-itop-id`

**Request**:
```json
{
  "order_id": 123456,
  "external_order_id": 789012
}
```

**Response**:
```json
{
  "code": "200",
  "message": "iTOP ID set successfully"
}
```

---

## Maintenance

### CheckServiceMaintenance
Check if service is under maintenance.

**gRPC**: `TaxiOrderProcessor.CheckServiceMaintenance`  
**REST**: `POST /v1/maintenance/check`

**Request**:
```json
{
  "service_type_id": 1,  // 1=BlueBird, 2=SilverBird, 3=GoldenBird
  "check_time": "2025-01-07T10:00:00Z"
}
```

**Response**:
```json
{
  "code": "200",
  "message": "Service available",
  "is_maintenance": false
}
```

**Under Maintenance**:
```json
{
  "code": "TOP-5000",
  "message": "Service under maintenance",
  "is_maintenance": true,
  "maintenance_start": "2025-01-07T02:00:00Z",
  "maintenance_end": "2025-01-07T04:00:00Z"
}
```

---

## Search & History

### GetOrderForSearchLocationHistory
Get order untuk location history search.

**gRPC**: `TaxiOrderProcessor.GetOrderForSearchLocationHistory`  
**REST**: `POST /v1/orders/search-location-history`

**Request**:
```json
{
  "user_id": "user_123",
  "limit": 10
}
```

**Response**:
```json
{
  "orders": [
    {
      "order_id": 123456,
      "pickup_address": "Jl. Sudirman No. 123",
      "pickup_latitude": -6.2088,
      "pickup_longitude": 106.8456,
      "dropoff_address": "Jl. Thamrin No. 456",
      "dropoff_latitude": -6.1944,
      "dropoff_longitude": 106.8229,
      "created_at": 1704614400
    }
  ]
}
```

---

## State Transitions Summary

### Valid State Transitions

```
ORDER_INITIAL (-2)
  → LOOKING_FOR_DRIVER (0)
  → ORDER_FAILED (-1) [terminal]
  → ORDER_TIMEOUT (-3) [terminal]

LOOKING_FOR_DRIVER (0)
  → EN_ROUTE (1)
  → CANCELED (2) [terminal]
  → NO_TAXI (3) [terminal]
  → TAXI_NO_SHOW (5) [terminal]
  → CHAT_ROOM_CREATED (10)

EN_ROUTE (1)
  → ON_TRIP (4)
  → CANCELED (2) [terminal]
  → TAXI_NO_SHOW (5) [terminal]

ON_TRIP (4)
  → TRACKING (6)
  → CANCELED (2) [terminal]

TRACKING (6)
  → DROP_OFF (8) [terminal]
  → ECV_DROP_OFF (7) [terminal]
```

### Terminal States
Orders in terminal states tidak bisa berubah state lagi:
- ORDER_TIMEOUT (-3)
- ORDER_FAILED (-1)
- CANCELED (2)
- NO_TAXI (3)
- TAXI_NO_SHOW (5)
- ECV_DROP_OFF (7)
- DROP_OFF (8)
- OFFLINE_DISPATCH (9)

---

## Error Code Reference

| Code | HTTP | Description |
|------|------|-------------|
| TOP-4000 | 400 | Invalid value in request |
| TOP-4001 | 400 | Car number missing / User info not exist |
| TOP-4004 | 404 | Order not found / Limit future booking reached |
| TOP-4005 | 403 | Forbidden request |
| TOP-4008 | 400 | Payment CC rejected |
| TOP-4009 | 400 | Trip purpose empty |
| TOP-40010 | 400 | Payment unavailable |
| TOP-40011 | 400 | ECV not applicable for date |
| TOP-40012 | 400 | ECV not applicable for car type |
| TOP-40013 | 400 | Cash for easy ride not allowed |
| TOP-40014 | 400 | Trip voucher validation failed |
| TOP-4031 | 429 | CC concurrent limit reached |
| TOP-4032 | 409 | Ongoing e-wallet order exists |
| TOP-4033 | 409 | Multiple booking disallowed (soft ban) |
| TOP-4034 | 400 | Chat room empty |
| TOP-4041 | 404 | Card not found |
| TOP-4091 | 409 | Payment already success |
| TOP-4092 | 402 | Charging failed |
| TOP-4093 | 402 | Fare exceeds balance |
| TOP-4221 | 409 | Active/failed charge TCash order exists |
| TOP-5000 | 500 | Internal error |
| TOP-5001 | 500 | General UPG error |
| TOP-5002 | 500 | Failed retry order |
| TOP-5003 | 500 | Trip fare iTOP invalid |
| TOP-9020 | 400 | Tip below minimum (1000 IDR) |
| TOP-9021 | 400 | Tip exceeds maximum |
| TOP-9022 | 400 | Invalid rating (must be 1-5) |

---

## Related Documentation
- [[01-overview|Service Overview]]
- [[03-state-machine|State Machine Details]]
- [[04-payment-flows|Payment Flows]]
- [[05-dependencies|Dependencies]]

---
**Last Updated**: 2025-01-07
**API Version**: v1
**Protocol**: gRPC + REST Gateway