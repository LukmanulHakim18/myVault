---
tags:
  - api
  - grpc
  - taxipartnergateway
  - bbd
  - dispatch
  - mrg
  - reference
type: api-documentation
title: Taxi Partner Gateway - API Reference
parent: taxipartnergateway
---
# Taxi Partner Gateway - API Reference

**Service**: [[README|Taxi Partner Gateway]]  
**Type**: API Documentation

---

## 📡 Server Configuration

| Protocol | Port | Default |
|----------|------|---------|
| gRPC | `GRPC_PORT` | 6008 |
| REST (gRPC-Gateway) | `REST_PORT` | 8008 |

---

## 🔧 gRPC Methods

### Health Check

#### HealthCheck
```protobuf
rpc HealthCheck(Ping) returns (Ping);
```
- **Input**: `{ping: string}`
- **Output**: `{ping: string}`

---

### Order Operations

#### OrderTaxi
Create a new taxi order.
```protobuf
rpc OrderTaxi(OrderTaxiReq) returns (OrderTaxiResp);
```
- **Input**:
  ```protobuf
  message OrderTaxiReq {
    string request_id = 1;
    OrderService service = 2;         // {service_id, package_id}
    OrderSchedule schedule = 3;       // {type, pickup_time}
    repeated OrderCustomer customers = 4;
    Route route = 5;
    repeated OrderPayment payments = 6;
    repeated OrderPromo promos = 7;
    repeated OrderFare fares = 8;
    OrderPreference preference = 9;
    OrderNote note = 10;
    OrderPartner partner = 11;
    CustLocation ordering_location = 12;
    string quote_id = 14;
    repeated Settlement settlements = 15;
    int64 handling_fee = 16;
  }
  ```
- **Output**:
  ```protobuf
  message OrderTaxiResp {
    int64 order_id = 1;
    repeated OrderFare fares = 2;
    repeated OrderExtra extras = 3;
    repeated OrderTaxi orders = 4;
  }
  ```

#### OrderTaxiEzpay
Create order with EzPay for street hailing scenario.
```protobuf
rpc OrderTaxiEzpay(OrderEzpayReq) returns (OrderEzpayResp);
```
- **Input**:
  ```protobuf
  message OrderEzpayReq {
    string request_id = 1;
    string vehicle_no = 2;           // Vehicle plate number
    OrderService service = 3;
    repeated OrderCustomer customers = 4;
    CustLocation ordering_location = 6;
    repeated OrderPayment payments = 7;
    repeated OrderPromo promos = 8;
    Route route = 9;
    OrderNote note = 10;
    repeated AdditionalInfo additional_info = 11;
  }
  ```
- **Output**:
  ```protobuf
  message OrderEzpayResp {
    int64 order_id = 1;
    int32 state = 2;
    Location engage_location = 3;
    repeated OrderExtra extras = 4;
  }
  ```

#### CancelOrder
Cancel an existing order.
```protobuf
rpc CancelOrder(CancelOrderReq) returns (CancelOrderResponse);
```
- **Input**:
  ```protobuf
  message CancelOrderReq {
    string request_id = 1;
    int64 order_id = 2;
    int32 cancel_by = 3;       // Who cancelled (customer/system/driver)
    int32 reason_type = 4;
    string reason_text = 5;
    string remark = 6;
  }
  ```
- **Output**:
  ```protobuf
  message CancelOrderResponse {
    int64 job_id = 1;
    string message = 2;
  }
  ```

#### RetryFindingTaxi
Retry finding driver for an order.
```protobuf
rpc RetryFindingTaxi(OrderIDReqID) returns (DefaultMessage);
```
- **Input**: `{order_id, request_id}`
- **Output**: `{code, message}`

#### GetOrderDetail
Get order details by order ID.
```protobuf
rpc GetOrderDetail(OrderIDReqID) returns (Order);
```
- **Input**: `{order_id, request_id}`
- **Output**: `Order` (see Common Types)

#### GetOrderDetailByPartner
Get order details by partner order ID.
```protobuf
rpc GetOrderDetailByPartner(OrderPartner) returns (Order);
```
- **Input**:
  ```protobuf
  message OrderPartner {
    string customer_id = 1;
    string order_id = 2;      // partner_order_id
    string service_id = 3;
    string extra_info_1 = 4;
    string extra_info_2 = 5;
  }
  ```

---

### Quote & Fare Operations

#### CreateQuote
Create a fare quote for a trip.
```protobuf
rpc CreateQuote(CreateQuoteReq) returns (CreateQuoteRes);
```
- **Input**:
  ```protobuf
  message CreateQuoteReq {
    string request_id = 1;
    QuoteService service = 2;      // {service_id, package_id}
    QuoteSchedule schedule = 3;    // {type, pickup_time}
    QuoteRoute route = 4;          // {locations[]}
    repeated QuotePayment payments = 5;
  }
  ```
- **Output**:
  ```protobuf
  message CreateQuoteRes {
    string quote_id = 1;
    repeated QuoteFare fares = 2;
    QuoteRouteRes route = 3;
    string expired = 4;
  }
  ```

#### GetEstimateFare
Get fare estimation without creating quote.
```protobuf
rpc GetEstimateFare(EstimateFareReq) returns (EstimateFareResp);
```
- **Input**:
  ```protobuf
  message EstimateFareReq {
    int32 service_id = 1;
    int32 package_id = 2;
    string request_id = 3;
    int32 scheduler_type = 4;
    string pickup_time = 5;
    repeated int32 loc_types = 6;
    repeated double loc_latitudes = 7;
    repeated double loc_longitudes = 8;
    repeated string attributes = 9;
  }
  ```
- **Output**:
  ```protobuf
  message EstimateFareResp {
    repeated EstimateFare fares = 1;
    repeated AttributeResp attributes = 2;
  }
  ```

#### EditPartnerFare
Modify order fare/promo after creation.
```protobuf
rpc EditPartnerFare(EditPartnerFareRequest) returns (EditPartnerFareResponse);
```
- **Input**:
  ```protobuf
  message EditPartnerFareRequest {
    string request_id = 1;
    repeated PartnerPromo promos = 2;
    repeated PartnerFare fares = 3;
    string order_id = 4;
  }
  ```

---

### Vehicle & Location Operations

#### GetVacantVehicles
Get available vehicles in a specific area.
```protobuf
rpc GetVacantVehicles(VacantVehiclesReq) returns (VacantVehiclesResp);
```
- **Input**:
  ```protobuf
  message VacantVehiclesReq {
    string request_id = 1;
    int32 service_id = 2;
    int32 package_id = 3;
    double latitude = 4;
    double longitude = 5;
    int32 radius = 6;        // Search radius in meters
    int32 count = 7;         // Max vehicles to return
  }
  ```
- **Output**:
  ```protobuf
  message VacantVehiclesResp {
    int32 num_vehicles = 1;
    repeated Vehicle vehicles = 2;
  }
  
  message Vehicle {
    string vehicle_type = 1;
    string vehicle_model = 2;
    int32 passenger_seat = 3;
    string vehicle_no = 4;
    Location location = 5;
  }
  ```

#### GetVehicleLocation
Get current vehicle location for an order.
```protobuf
rpc GetVehicleLocation(OrderIDReqID) returns (VehicleLocationResp);
```
- **Input**: `{order_id, request_id}`
- **Output**:
  ```protobuf
  message VehicleLocationResp {
    OrderVehicleLocationResp order = 1;
  }
  
  message OrderVehicleLocationResp {
    int32 status = 1;
    Track track = 2;           // {location: {lat, lng}}
    repeated AttributeResp attributes = 3;
  }
  ```

#### GetVehicleByVehicleNo
Get vehicle information by plate number.
```protobuf
rpc GetVehicleByVehicleNo(GetVehicleByVehicleNoRequest) returns (GetVehicleByVehicleNoResponse);
```
- **Input**: `{request_id, vehicle_no}`
- **Output**:
  ```protobuf
  message GetVehicleByVehicleNoResponse {
    int32 base_package_id = 1;
    int32 fleet_service_id = 2;
    int32 service_type_id = 3;
  }
  ```

#### GetEta
Get estimated time of arrival.
```protobuf
rpc GetEta(GetEtaReq) returns (GetEtaResp);
```
- **Input**:
  ```protobuf
  message GetEtaReq {
    string request_id = 1;
    repeated ServiceRelation service_types = 2;
    Location location = 3;
    string trip_id = 8;
  }
  ```
- **Output**:
  ```protobuf
  message GetEtaResp {
    repeated Eta service_types = 1;
  }
  
  message Eta {
    int32 service_id = 1;
    string partner_service_id = 2;
    string service_name = 3;
    int32 num_vehicles = 4;
    double min_eta = 5;
    double max_eta = 6;
    string time_unit = 7;    // "min", "sec"
  }
  ```

---

### Order Modification

#### EditPartnerLocation
Update order pickup/dropoff locations.
```protobuf
rpc EditPartnerLocation(EditPartnerLocationRequest) returns (DefaultMessage);
```
- **Input**:
  ```protobuf
  message EditPartnerLocationRequest {
    string request_id = 1;
    repeated PartnerLocation locations = 2;
    string order_id = 3;
  }
  
  message PartnerLocation {
    double latitude = 1;
    double longitude = 2;
    string address = 3;
    string remark = 4;
    repeated int32 pickup_customers = 5;
    repeated int32 drop_customers = 6;
  }
  ```

#### EditPartnerPayment
Update order payment method.
```protobuf
rpc EditPartnerPayment(EditPartnerPaymentRequest) returns (EditPartnerPaymentResponse);
```
- **Input**:
  ```protobuf
  message EditPartnerPaymentRequest {
    string request_id = 1;
    repeated PartnerPayment payments = 2;
    string order_id = 3;
  }
  ```

---

### Rating & Review

#### SetOrderRating
Submit rating for completed order.
```protobuf
rpc SetOrderRating(SetOrderRatingRequest) returns (SetOrderRatingResponse);
```
- **Input**:
  ```protobuf
  message SetOrderRatingRequest {
    int64 order_id = 1;
    uint32 rating = 2;           // 1-5 stars
    string driver_id = 3;
    string vehicle_no = 4;
    repeated uint32 review_id = 5;  // Selected review options
  }
  ```

#### GetOrderRatingReviews
Get available rating review options.
```protobuf
rpc GetOrderRatingReviews(GetOrderRatingReviewsRequest) returns (GetOrderRatingReviewsResponse);
```
- **Input**: `{rating: string}`
- **Output**:
  ```protobuf
  message GetOrderRatingReviewsResponse {
    repeated RatingReviews rating_reviews = 1;
  }
  
  message RatingReviews {
    uint32 rating = 1;
    repeated Reviews reviews = 2;  // {review_id, review text}
  }
  ```

---

### Area Operations

#### GetAreaOperationalAndAirport
Check if location is in operational area and detect airport.
```protobuf
rpc GetAreaOperationalAndAirport(GetAreaOperationalAndAirportRequest) returns (GetAreaOperationalAndAirportResponse);
```
- **Input**:
  ```protobuf
  message GetAreaOperationalAndAirportRequest {
    Location pickup = 1;
    Location drop_off = 2;
  }
  ```
- **Output**:
  ```protobuf
  message GetAreaOperationalAndAirportResponse {
    bool is_airport = 1;
    string airport_code = 2;
    OperationalAndAirportArea operational_and_airport_area = 3;
    bool has_multiple_airports = 4;
  }
  ```

#### GetAreaById
Get area details by ID.
```protobuf
rpc GetAreaById(GetAreaByIdRequest) returns (Area);
```
- **Input**: `{id_lsb, id_msb, include_child}`
- **Output**: `Area` (see Common Types)

---

### System Operations

#### GetServices
Get available service types for a location.
```protobuf
rpc GetServices(GetServicesReq) returns (GetServicesResp);
```
- **Input**:
  ```protobuf
  message GetServicesReq {
    string request_id = 1;
    double loc_latitude = 2;
    double loc_longitude = 3;
  }
  ```
- **Output**:
  ```protobuf
  message GetServicesResp {
    repeated ServiceType services = 1;
  }
  
  message ServiceType {
    int32 service_id = 1;
    string name = 2;
    repeated BasePackage Packages = 3;
  }
  ```

#### SetSystemCallbackURL
Set callback URL for order event notifications.
```protobuf
rpc SetSystemCallbackURL(SetSystemCallBackUrlRequest) returns (DefaultMessage);
```
- **Input**:
  ```protobuf
  message SetSystemCallBackUrlRequest {
    string request_id = 1;
    SystemCallbackURL callback = 2;
  }
  
  message SystemCallbackURL {
    string api_name = 1;
    string api_version = 2;
    string url = 3;
    repeated SystemCallbackHeader headers = 4;
  }
  ```

#### NotifyOrderEvent
Handle order event notification from BBD.
```protobuf
rpc NotifyOrderEvent(NotifyEvent) returns (DefaultMessage);
```
- **Input**:
  ```protobuf
  message NotifyEvent {
    OrderEvent event = 1;
    Order order = 2;
    Track track = 3;
    repeated OrderPayment payments = 4;
    repeated OrderFare fares = 5;
    repeated OrderExtra extras = 6;
    OrderVehicle vehicle = 7;
    OrderDriver driver = 8;
    OrderCancel cancel = 9;
    OrderTrip trip = 10;
    uint64 partner_id_msb = 11;
    uint64 partner_id_lsb = 12;
  }
  ```

---

## 📝 Common Types

### Order
```protobuf
message Order {
  int64 order_id = 1;
  int32 status = 2;
  OrderService service = 3;
  repeated OrderCustomer customers = 4;
  Route route = 5;
  repeated OrderPayment payments = 6;
  repeated OrderPromo promos = 8;
  repeated OrderFare fares = 9;
  OrderPreference preference = 10;
  OrderNote note = 11;
  OrderPartner partner = 12;
  repeated OrderExtra extras = 13;
  OrderVehicle vehicle = 14;
  OrderDriver driver = 15;
  OrderRequest request = 16;
  OrderCancel cancel = 17;
  repeated OrderEvent events = 18;
}
```

### OrderCustomer
```protobuf
message OrderCustomer {
  int32 type = 1;
  string name = 2;
  string phone = 3;
  string callback_phone = 4;
  string email = 5;
  string partner_cust_id = 6;
  int32 is_callback = 7;
}
```

### OrderDriver
```protobuf
message OrderDriver {
  string nip = 1;
  string name = 2;
  string photo_url = 3;
  double rating = 4;
}
```

### OrderVehicle
```protobuf
message OrderVehicle {
  string vehicle_no = 1;
  string phone = 2;
  string model = 3;
}
```

### Area
```protobuf
message Area {
  string area_name = 1;
  int32 area_type = 2;
  string area_code = 3;
  string city_code = 7;
  double latitude = 8;
  double longitude = 9;
  int32 timezone = 13;
  string phone_code = 14;
  repeated Area child_areas = 15;
}
```

### DefaultMessage
```protobuf
message DefaultMessage {
  int32 code = 1;
  string message = 2;
}
```

---

## 🏷️ Tags

#api #grpc #taxipartnergateway #bbd #dispatch #mrg #reference

---

*Last Updated*: 2025-01-05
