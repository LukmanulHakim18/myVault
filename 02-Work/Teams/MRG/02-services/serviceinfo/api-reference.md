---
title: Service Info - API Reference
type: api-documentation
parent: serviceinfo
tags:
  - api
  - grpc
  - serviceinfo
  - mrg
  - reference
---
# Service Info - API Reference

**Service**: [[README|Service Info]]  
**Type**: API Documentation

---

## 📡 Server Configuration

| Protocol | Port | Default |
|----------|------|---------|
| gRPC | `GRPC_PORT` | 6011 |
| REST (gRPC-Gateway) | `REST_PORT` | 8011 |

---

## 🔧 gRPC Methods

### Health & System

#### HealthCheck
```protobuf
rpc HealthCheck(EmptyRequest) returns (DefaultResponse);
```
- **Description**: Health check endpoint
- **Input**: Empty
- **Output**: `{code, message}`

#### PopulateCacheServiceTypeMapping
```protobuf
rpc PopulateCacheServiceTypeMapping(EmptyMessage) returns (EmptyMessage);
```
- **Description**: Populate cache untuk service type mapping dari database

#### UpdateServiceTypeMapping
```protobuf
rpc UpdateServiceTypeMapping(UpdateServiceTypeMappingRequest) returns (EmptyMessage);
```
- **Input**:
  - `service_id_bbd`: int32 - BBD service ID
  - `package_id_bbd`: int32 - BBD package ID
  - `service_type_id_mybb`: int32 - MyBB service type ID

#### DeleteServiceTypeMapping
```protobuf
rpc DeleteServiceTypeMapping(DeleteServiceTypeMappingRequest) returns (EmptyMessage);
```
- **Input**:
  - `service_id_bbd`: int32
  - `package_id_bbd`: int32

---

### Fleet & Vehicle

#### DynamicFleetListV2
```protobuf
rpc DynamicFleetListV2(DynamicFleetListV2Request) returns (DynamicFleetListV2Response);
```
- **Interceptors**: InputValidator
- **Description**: Get dynamic fleet list untuk rental
- **Input**:
  ```
  pickup_latitude: double
  pickup_longitude: double
  area_code: string
  date: string
  time: string
  duration: string
  service_type: string
  dropoff_latitude: double
  dropoff_longitude: double
  pickup_is_airport: bool
  dropoff_is_airport: bool
  pickup_address: string
  dropoff_address: string
  ```
- **Output**:
  ```
  title: string
  description: string
  information: {icon_url, text}
  session_id: string
  fleet: [DynamicFleetV2]
  area_info: {area_id, city, city_code}
  ```

#### ChooseFleet
```protobuf
rpc ChooseFleet(ChooseFleetRequest) returns (ChooseFleetResponse);
```
- **Interceptors**: InputValidator
- **Input**:
  - `session_id`: string
  - `fleet_id`: string
  - `service_type`: string

#### VehicleInfo
```protobuf
rpc VehicleInfo(VehicleInfoRequest) returns (VehicleInfoResponse);
```
- **Input**: `car_number`: string
- **Output**: `service_type_id`: int32

---

### Rental Flow

#### InitialInfo
```protobuf
rpc InitialInfo(InitialInfoRequest) returns (InitialInfoResponse);
```
- **Interceptors**: InputValidator
- **Description**: Get initial info untuk memulai rental flow
- **Input**:
  - `latitude`: string
  - `longitude`: string
  - `service`: string (hourly/daily)
- **Output**:
  ```
  trip_journey: {id, trip_id}
  autofill: {contact, area, date, time, duration, estimate_end_time_text}
  config: {start_date, end_date, available_time, available_duration}
  decoration: {greeting, operational_area, action_button, faq}
  ```

#### RentEstimateEndTime
```protobuf
rpc RentEstimateEndTime(RentEstimateEndTimeRequest) returns (RentEstimateEndTimeResponse);
```
- **Input**:
  - `area_code`: string
  - `date`: string
  - `time`: string
  - `duration`: string
- **Output**: `estimate_end_time_text`: string

#### RecalculateRentPrice
```protobuf
rpc RecalculateRentPrice(RecalculateRentPriceRequest) returns (RecalculateRentPriceResponse);
```
- **Input**:
  ```
  session_id: string
  pickup: LocationDetail
  dropoff: LocationDetail
  all_in_packages: bool
  selected_package: [SelectedPackage]
  ```

#### RentOrderConfirmation
```protobuf
rpc RentOrderConfirmation(RentOrderConfirmationRequest) returns (RentOrderConfirmationResponse);
```
- **Input**:
  ```
  session_id: string
  payment: {type, identifier}
  time: string
  marketing: {promo: {code}, subscription: {id}}
  ```

#### RentOrderPaymentSelection
```protobuf
rpc RentOrderPaymentSelection(RentOrderPaymentSelectionRequest) returns (RentOrderPaymentSelectionResponse);
```
- **Input**: `session_id`: string
- **Output**: List of payment options grouped

---

### Validation

#### ValidatePrebook
```protobuf
rpc ValidatePrebook(ValidatePrebookRequest) returns (ValidatePrebookResponse);
```
- **Input**:
  ```
  session_id: string
  fleet_id: string
  date: string
  time: string
  duration: {id, type, display}
  pickup: LocationDetail
  dropoff: LocationDetail
  driver_notes: string
  ```
- **Output**:
  ```
  data: {
    is_upsell: bool
    upsell: UpsellInfo (if applicable)
  }
  ```

#### ValidateCreateOrder
```protobuf
rpc ValidateCreateOrder(ValidateCreateOrderRequest) returns (ValidateCreateOrderResponse);
```
- **Input**:
  ```
  session_id: string
  driver_notes: string
  trip_purpose: string
  service_type: string
  ```

#### ValidateEZPay
```protobuf
rpc ValidateEZPay(ValidateEZPayRequest) returns (ValidateEZPayResponse);
```
- **Interceptors**: MetadataValidator, InputValidator
- **Input**:
  ```
  car_number: string
  pickup: Location
  dropoff: Location
  payment_provider: string
  payment_identifier: string
  trip_id: string
  promo_code: string
  subscription_order_id: int32
  ```
- **Output**:
  ```
  service_type_id: int32
  service_type_name: string
  ```

---

### Trip & Booking

#### NewTripIdOdrd
```protobuf
rpc NewTripIdOdrd(Location) returns (NewTripOdrdResponse);
```
- **Interceptors**: MetadataValidator
- **Input**: `{latitude, longitude, place_id}`
- **Output**: `{trip_id, name}`

#### CreateConsumerToken
```protobuf
rpc CreateConsumerToken(AuthorizationMap) returns (ConsumerTokenResponse);
```
- **Interceptors**: InputValidator, MetadataValidator
- **Input**: `{trip_id}`
- **Output**:
  ```
  creation_timestamp: string
  expiration_timestamp: string
  audience: string
  token: string
  token_type: string
  authorization_claims: AuthorizationClaims
  ```

#### GetCallCenterList
```protobuf
rpc GetCallCenterList(CallCentersRequest) returns (CallCentersResponse);
```
- **Input**:
  - `product_type`: string
  - `city`: string
- **Output**: List of call centers per area

---

### Cititrans (Shuttle)

#### GetScheduleListCititrans
```protobuf
rpc GetScheduleListCititrans(GetScheduleListCititransRequest) returns (GetScheduleListCititransResponse);
```
- **Interceptors**: MetadataValidator, InputValidator
- **Input**:
  ```
  date: string
  shelter_departure_id: string
  shelter_arrival_id: string
  total_passenger: int32
  service_type_id: string
  sort_by: string
  departure_time: string
  page: string
  limit: string
  session_id: string
  order_id: int64
  ```

#### GetVesselDetail
```protobuf
rpc GetVesselDetail(GetVesselDetailRequest) returns (GetVesselDetailResponse);
```
- **Input**: `{id, token}`
- **Output**: Vessel/bus details with seats, facilities, images

#### GetCititransSession / SetCititransSession
```protobuf
rpc GetCititransSession(GetCititransSessionRequest) returns (GetCititransSessionResponse);
rpc SetCititransSession(SetCititransSessionRequest) returns (SetCititransSessionResponse);
```
- Manage Cititrans booking session

---

## 📝 Common Types

### Location
```protobuf
message Location {
  double latitude = 1;
  double longitude = 2;
  string place_id = 3;
}
```

### LocationDetail
```protobuf
message LocationDetail {
  string id = 1;
  string external_id = 2;
  repeated GoldenbirdArea goldenbird_areas = 3;
  string name = 4;
  Coordinate coordinate = 5;
  string address = 6;
  string place_category = 7;
  repeated Subplace subplaces = 8;
  repeated PolygonPoint polygon = 9;
}
```

### Duration
```protobuf
message Duration {
  string id = 1;
  string type = 2;    // "hourly" or "daily"
  string display = 3;
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
  string business_code = 11;
  string product_category_name = 12;
  string service_line = 13;
  string price_code = 14;
}
```

---

## 🏷️ Tags

#api #grpc #serviceinfo #mrg #reference

---

*Last Updated*: 2025-01-05
