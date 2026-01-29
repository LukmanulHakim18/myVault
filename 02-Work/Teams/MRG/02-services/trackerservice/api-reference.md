---
tags:
  - api
  - grpc
  - trackerservice
  - mrg
  - reference
type: api-documentation
title: Tracker Service - API Reference
parent: trackerservice
---
# Tracker Service - API Reference

**Service**: [[README|Tracker Service]]  
**Type**: API Documentation

---

## 📡 Server Configuration

| Protocol | Port | Default |
|----------|------|---------|
| gRPC | `GRPC_PORT` | 6013 |
| REST (gRPC-Gateway) | `REST_PORT` | 8013 |

---

## 🔧 gRPC Methods

### Health & Utility

#### HealthCheck
```protobuf
rpc HealthCheck(EmptyRequest) returns (DefaultResponse);
```
- **Description**: Service health check
- **Output**: `{code: int32, message: string}`

---

### Car Tracking

#### CarTracking
```protobuf
rpc CarTracking(CarLocationRequest) returns (DefaultResponse);
```
- **Description**: Update driver's current location during active trip
- **Frequency**: Called every 20 seconds by driver app
- **Input**:
  ```protobuf
  message CarLocationRequest {
    string car_number = 1;       // Vehicle registration number
    int64 order_id = 2;          // Internal order ID
    int64 external_order_id = 3; // External system order ID
    MetaData meta_data = 4;      // Request metadata
    string user_id = 5;          // Driver user ID
  }
  
  message MetaData {
    string app_version = 1;
    string device_type = 2;      // "ios" or "android"
    string device_uuid = 3;
    string accept_language = 4;
    string request_id = 5;
  }
  ```
- **Output**:
  ```protobuf
  message DefaultResponse {
    int32 code = 1;     // 200 = success, 400 = error
    string message = 2;
  }
  ```

**Business Logic**:
1. Validate order is active
2. Get driver's current location from device
3. Store location in Firebase: `/car_tracking/{order_id}`
4. Cache in Redis with 24h TTL
5. Return success response

**Firebase Data Structure**:
```json
{
  "car_tracking": {
    "ORDER_12345": {
      "car_number": "B1234XYZ",
      "latitude": -6.2088,
      "longitude": 106.8456,
      "timestamp": 1706246400,
      "driver_id": "DRV123"
    }
  }
}
```

#### GetNearbyCars
```protobuf
rpc GetNearbyCars(NearbyCarsRequest) returns (NearbyCarsResponse);
```
- **Interceptors**: InputValidator, MetadataValidator
- **Description**: Find available cars near user's location
- **Input**:
  ```protobuf
  message NearbyCarsRequest {
    Location location = 1;
    int64 service_type_id = 2;  // 1=Regular, 2=Premium, 3=Executive
  }
  
  message Location {
    double latitude = 1;
    double longitude = 2;
  }
  ```
- **Output**:
  ```protobuf
  message NearbyCarsResponse {
    repeated Car cars = 1;
    string error = 2;
  }
  
  message Car {
    string taxi_category = 1;   // "goldenbird" or "partner"
    string number = 2;           // Car registration number
    Location location = 3;       // Car current location
  }
  ```

**Business Logic**:
1. Check Redis cache (key: `nearby_cars:{lat}:{lng}:{service_type}`)
2. If cache miss, query both sources in parallel:
   - Goldenbird Gateway: Get GB fleet cars
   - Taxi Partner Gateway: Get partner cars
3. Merge results
4. Calculate distance from user location
5. Filter by radius (default: 200m configurable)
6. Sort by distance (nearest first)
7. Apply limit (default: 10 cars)
8. Cache result for 30 seconds
9. Return cars list

**Response Example**:
```json
{
  "cars": [
    {
      "taxi_category": "goldenbird",
      "number": "B1234XYZ",
      "location": {
        "latitude": -6.2088,
        "longitude": 106.8456
      }
    },
    {
      "taxi_category": "partner",
      "number": "B5678ABC",
      "location": {
        "latitude": -6.2090,
        "longitude": 106.8460
      }
    }
  ]
}
```

---

### Trip Management

#### ShareTrip
```protobuf
rpc ShareTrip(ShareTripRequest) returns (ShareTripResponse);
```
- **Interceptors**: MetadataValidator
- **Description**: Generate shareable URL for trip tracking
- **Input**:
  ```protobuf
  message ShareTripRequest {
    string order_id = 1;  // Order ID to share
    string trip_id = 2;   // Trip ID for tracking
  }
  ```
- **Output**:
  ```protobuf
  message ShareTripResponse {
    string url = 1;  // Shareable URL
  }
  ```

**Business Logic**:
1. Validate order exists and is active
2. Generate unique share token (UUID)
3. Store share session in Redis:
   ```
   Key: share_trip:{token}
   Value: {order_id, trip_id, created_at, expires_at}
   TTL: Until trip ends (max 24h)
   ```
4. Queue background job untuk parse order details
5. Generate shareable URL: `https://share.mybluebird.id/trip/{token}`
6. Return URL to user

**Background Worker**:
- Parse order details from Order Query
- Generate public view page with:
  - Driver info (name, photo, car details)
  - Pickup & dropoff locations
  - ETA
  - Real-time location updates
- Store parsed data in Redis for fast serving

**Share Trip URL Response Example**:
```json
{
  "url": "https://share.mybluebird.id/trip/550e8400-e29b-41d4-a716-446655440000"
}
```

**Public View Features**:
- Real-time driver location on map
- Trip route visualization
- ETA updates
- Driver & car information
- Pickup & dropoff details
- No authentication required

#### OrderChanges
```protobuf
rpc OrderChanges(OrderChangesReq) returns (DefaultResponse);
```
- **Description**: Handle order state changes from PubSub events
- **Trigger**: Automatic via PubSub subscription
- **Input**:
  ```protobuf
  message OrderChangesReq {
    int64 order_state = 1;         // Order state ID
    bool trip_ended = 2;           // True if trip completed
    string final_epay = 3;         // Final e-payment amount
    string epay_flag = 4;          // E-payment status flag
    string passcode_fail = 5;      // Payment failure reason
    string payment_method = 6;     // "cash", "epay", "cc"
    string error = 7;              // Error message if any
    string passcode = 8;           // Payment passcode
    bool insufficient_balance = 9; // E-wallet balance check
    int64 service_type = 10;       // Service type ID
    int64 order_id = 11;           // Order ID
    int64 external_order_id = 12;  // External order ID
    string user_id = 13;           // Customer user ID
  }
  ```
- **Output**:
  ```protobuf
  message DefaultResponse {
    int32 code = 1;
    string message = 2;
  }
  ```

**Business Logic by Order State**:

| State | Action |
|-------|--------|
| **1 - Order Created** | Initialize tracking session |
| **2 - Driver Assigned** | Start car tracking |
| **3 - Driver Arrived** | Notify customer |
| **4 - Trip Started** | Begin real-time tracking |
| **5 - Trip Ended** | Stop tracking, trigger end process |
| **6 - Payment Processing** | Update payment status |
| **7 - Order Completed** | Cleanup tracking data |
| **8 - Order Cancelled** | Stop tracking, cleanup |

**End Trip Process** (when `trip_ended = true`):
1. Stop car tracking updates
2. Store final location
3. Calculate trip statistics
4. Sync payment status with Payment Processor
5. Update Firebase tracking status
6. Clear Redis tracking cache
7. Expire share trip URL
8. Notify relevant services

---

## 📝 Redis Key Patterns

### Enums for Redis Keys
```protobuf
enum KEY {
  NEARBY_CARS    = 0;  // Prefix: nearby_cars:
  CAR_LOCATION   = 1;  // Prefix: car_location:
  ORDER_CHANGE   = 2;  // Prefix: order_change:
}

enum FIELD {
  NEARBY_CAR_REQUEST      = 0;  // Request metadata
  NEARBY_CAR_RESPONSE     = 1;  // Cached nearby cars
  CAR_LOCATION_RESPONSE   = 2;  // Current car location
  CAR_LOCATION_IS_SHARE   = 3;  // Share trip active flag
  ORDER_CHANGE_RESPONSE   = 4;  // Last order state
}
```

### Redis Key Structure

**Nearby Cars Cache**:
```
Key Pattern: nearby_cars:{latitude}:{longitude}:{service_type_id}
Value: JSON array of Car objects
TTL: 30 seconds
```

**Car Location Cache**:
```
Key Pattern: car_location:{order_id}
Value: Hash map with fields:
  - NEARBY_CAR_REQUEST: Request metadata
  - CAR_LOCATION_RESPONSE: Current location
  - CAR_LOCATION_IS_SHARE: Boolean flag
TTL: 24 hours
```

**Order Change Cache**:
```
Key Pattern: order_change:{order_id}
Value: Hash map with fields:
  - ORDER_CHANGE_RESPONSE: Last state data
TTL: 24 hours
```

**Share Trip Session**:
```
Key Pattern: share_trip:{token}
Value: JSON object with order_id, trip_id, metadata
TTL: Until trip ends (max 24h)
```

---

## 🔀 Method Flow Diagrams

### CarTracking Flow
```
Driver App (every 20s)
    ↓
CarTracking(car_number, order_id, location)
    ↓
Validate order is active
    ↓
Firebase.Set(/car_tracking/{order_id}, location)
    ↓
Redis.HSet(car_location:{order_id}, RESPONSE, location)
    ↓
Return success
    ↓
Client apps receive Firebase update (real-time)
```

### GetNearbyCars Flow
```
User opens app
    ↓
GetNearbyCars(location, service_type)
    ↓
Check Redis cache
    ↓ (cache miss)
Parallel queries:
  - GbGateway.GetAvailableCars()
  - TaxiPartner.GetAvailableCars()
    ↓
Merge results
    ↓
Calculate distances
    ↓
Filter by radius (200m)
    ↓
Sort by distance
    ↓
Limit to 10 cars
    ↓
Cache in Redis (30s TTL)
    ↓
Return cars to client
```

### ShareTrip Flow
```
User taps "Share Trip"
    ↓
ShareTrip(order_id, trip_id)
    ↓
Validate order active
    ↓
Generate unique token (UUID)
    ↓
Redis.Set(share_trip:{token}, session_data)
    ↓
Queue background job
    ↓
Generate URL: https://share.mybluebird.id/trip/{token}
    ↓
Return URL to user
    ↓
Background Worker:
  - Parse order details
  - Generate public view
  - Cache parsed data
    ↓
External user accesses URL → See real-time tracking
```

### OrderChanges Flow
```
PubSub Event (Order State Change)
    ↓
OrderChanges(state, order_data)
    ↓
Parse order state
    ↓
Switch (state):
  case TRIP_STARTED:
    - Initialize tracking
  case TRIP_ENDED:
    - Stop tracking
    - Trigger end process
  case CANCELLED:
    - Cleanup tracking data
    ↓
Update Redis cache
    ↓
Update Firebase status
    ↓
Return acknowledgment
```

---

## 🎯 Request/Response Examples

### Example 1: CarTracking Update
**Request**:
```json
{
  "car_number": "B1234XYZ",
  "order_id": 12345,
  "external_order_id": 67890,
  "user_id": "DRV123",
  "meta_data": {
    "app_version": "3.5.0",
    "device_type": "android",
    "device_uuid": "abc-123-def",
    "accept_language": "id",
    "request_id": "req-12345"
  }
}
```

**Response**:
```json
{
  "code": 200,
  "message": "Location updated successfully"
}
```

### Example 2: GetNearbyCars
**Request**:
```json
{
  "location": {
    "latitude": -6.2088,
    "longitude": 106.8456
  },
  "service_type_id": 1
}
```

**Response**:
```json
{
  "cars": [
    {
      "taxi_category": "goldenbird",
      "number": "B1234XYZ",
      "location": {
        "latitude": -6.2089,
        "longitude": 106.8457
      }
    },
    {
      "taxi_category": "goldenbird",
      "number": "B5678ABC",
      "location": {
        "latitude": -6.2090,
        "longitude": 106.8458
      }
    }
  ],
  "error": ""
}
```

### Example 3: ShareTrip
**Request**:
```json
{
  "order_id": "ORDER123",
  "trip_id": "TRIP456"
}
```

**Response**:
```json
{
  "url": "https://share.mybluebird.id/trip/550e8400-e29b-41d4-a716-446655440000"
}
```

### Example 4: OrderChanges (Trip Ended)
**Request**:
```json
{
  "order_state": 5,
  "trip_ended": true,
  "final_epay": "150000",
  "payment_method": "epay",
  "service_type": 1,
  "order_id": 12345,
  "external_order_id": 67890,
  "user_id": "CUST123"
}
```

**Response**:
```json
{
  "code": 200,
  "message": "Trip ended successfully"
}
```

---

## 🏷️ Tags

#api #grpc #trackerservice #mrg #reference

---

*Last Updated*: 2025-01-26
