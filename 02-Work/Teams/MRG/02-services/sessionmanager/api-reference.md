---
tags:
  - api
  - grpc
  - sessionmanager
  - mrg
  - reference
type: api-documentation
title: Session Manager - API Reference
parent: sessionmanager
---
# Session Manager - API Reference

**Service**: [[README|Session Manager]]  
**Type**: API Documentation

---

## 📡 Server Configuration

| Protocol | Port | Default |
|----------|------|---------|
| gRPC | `GRPC_PORT` | 6027 |
| REST (gRPC-Gateway) | `REST_PORT` | 8027 |

---

## 🔧 gRPC Methods

### Health Check

#### HealthCheck
```protobuf
rpc HealthCheck(EmptyRequest) returns (DefaultResponse);
```
- **Description**: Health check endpoint
- **Output**: `{code, message}`

---

### Pre-Order Session

#### SetSession
```protobuf
rpc SetSession(SessionData) returns (DefaultResponse);
```
- **Interceptors**: ValidateInput
- **Description**: Store pre-order session data
- **Input**:
  ```protobuf
  message SessionData {
    string session_key = 1;
    FleetListRequest fleet_list_request = 2;
    FleetListResponse fleet_list_response = 3;
  }
  ```

#### GetSession
```protobuf
rpc GetSession(GetSessionRequest) returns (SessionData);
```
- **Interceptors**: ValidateInput
- **Input**: `{session_key: string}`
- **Output**: `SessionData`

#### DeleteSession
```protobuf
rpc DeleteSession(GetSessionRequest) returns (DefaultResponse);
```
- **Interceptors**: ValidateInput
- **Input**: `{session_key: string}`

---

### EZ Pay Session

#### SetSessionEZPay
```protobuf
rpc SetSessionEZPay(SessionEZPayData) returns (DefaultResponse);
```
- **Interceptors**: ValidateInput
- **Input**:
  ```protobuf
  message SessionEZPayData {
    string session_key = 1;
    ValidateEZPayRequest request = 2;
    ValidateEZPayResponse response = 3;
    BbdGetVehicleByVehicleNoResponse vehicle_info_bbd_response = 4;
  }
  ```

#### GetSessionEZPay
```protobuf
rpc GetSessionEZPay(GetSessionRequest) returns (SessionEZPayData);
```

#### DeleteSessionEZPay
```protobuf
rpc DeleteSessionEZPay(GetSessionRequest) returns (DefaultResponse);
```

---

### Cititrans Session

#### SetSessionCititrans
```protobuf
rpc SetSessionCititrans(SessionDataCititrans) returns (DefaultResponse);
```
- **Interceptors**: ValidateInput
- **Input**:
  ```protobuf
  message SessionDataCititrans {
    string session_key = 1;
    CititransScheduleListRequest request = 2;
    CititransScheduleListResponse response = 3;
    CititransSessionData session_data = 4;
  }
  ```

#### GetSessionCititrans
```protobuf
rpc GetSessionCititrans(GetSessionRequest) returns (SessionDataCititrans);
```

---

### Reschedule Session

#### SetSessionReschedule
```protobuf
rpc SetSessionReschedule(SessionReschedule) returns (DefaultResponse);
```
- **Input**:
  ```protobuf
  message SessionReschedule {
    bool delete_after_get = 1;
    string session_key = 2;
    string pickup_time = 3;
    bool is_price_changed = 4;
    double fixed = 5;
    double estimated_high = 6;
    double estimated_low = 7;
    bool is_promo_valid = 8;
    string promo_code = 9;
    LocalizedMessage localized_message = 10;
    string product_type = 11;
    string order_id = 12;
    double fixed_base = 13;
  }
  ```

#### GetSessionReschedule
```protobuf
rpc GetSessionReschedule(GetSessionRequest) returns (SessionReschedule);
```

---

### Rent Revamp Session

#### SetSessionRentRevamp
```protobuf
rpc SetSessionRentRevamp(SessionRentRevamp) returns (DefaultResponse);
```
- **Interceptors**: ValidateInput
- **Description**: Store complete rental booking session
- **Input**:
  ```protobuf
  message SessionRentRevamp {
    string session_key = 1;
    DynamicFleetListRequest dynamic_fleet_list_request = 2;
    DynamicFleetListResponse dynamic_fleet_list_response = 3;
    string fleet_id = 4;
    DynamicFleetV2 selected_fleet = 5;
    RecalculateRentPriceRequest recalculate_rent_price_request = 6;
    RecalculateRentPriceResponse recalculate_rent_price_response = 7;
    UpsellFleetResponse upsell_fleet_response = 8;
    ValidatePrebookRequest validate_prebook_request = 9;
    ValidatePrebookResponse validate_prebook_response = 10;
    RentOrderConfirmationRequest rent_order_confirmation_request = 11;
    RentOrderConfirmationResponse rent_order_confirmation_response = 12;
    ContactInformation contact_information = 13;
    PassengerInformation passenger_information = 14;
    LocationDetail selected_pickup = 15;
    LocationDetail selected_dropoff = 16;
  }
  ```

#### GetSessionRentRevamp
```protobuf
rpc GetSessionRentRevamp(GetSessionRequest) returns (SessionRentRevamp);
```

#### DeleteSessionRentRevamp
```protobuf
rpc DeleteSessionRentRevamp(GetSessionRequest) returns (DefaultResponse);
```

---

## 📝 Common Types

### GetSessionRequest
```protobuf
message GetSessionRequest {
  string session_key = 1;
}
```

### DefaultResponse
```protobuf
message DefaultResponse {
  string code = 1;
  string message = 2;
}
```

### Location
```protobuf
message Location {
  double latitude = 1;
  double longitude = 2;
}
```

### Fleet
```protobuf
message Fleet {
  string name = 1;
  string car_type = 2;
  ServiceType service_type = 3;
  string icon = 4;
  bool is_fixed_fare = 5;
  bool is_new = 6;
  int32 position = 7;
  InstructionField instruction_field = 8;
  int32 total_passengers = 9;
  Eta eta = 10;
  Fare fare = 11;
  PackageOption package_option = 12;
  string fleet_id = 13;
  string quote_id = 14;
  string icon_v6 = 15;
}
```

### DynamicFleetV2
```protobuf
message DynamicFleetV2 {
  string id = 1;
  string icon_url = 2;
  string name = 3;
  Passenger passenger = 4;
  Luggage luggage = 5;
  FareV2 fare = 6;
  FleetDescription description = 7;
  bool available = 8;
  string unavailable_reason = 9;
  string item_id = 10;
}
```

### ContactInformation
```protobuf
message ContactInformation {
  string salutation = 1;
  string name = 2;
  string phone_number = 3;
  string email_address = 4;
}
```

### PassengerInformation
```protobuf
message PassengerInformation {
  string salutation = 1;
  string name = 2;
  string phone_number = 3;
  string email_address = 4;
  string flight_number = 5;
  string note = 6;
}
```

---

## 🏷️ Tags

#api #grpc #sessionmanager #mrg #reference

---

*Last Updated*: 2025-01-05
