---
tags:
  - api
  - grpc
  - orderorchestrator
  - order
  - booking
  - mrg
  - reference
type: api-documentation
title: Order Orchestrator - API Reference
parent: orderorchestrator
---
# Order Orchestrator - API Reference

**Service**: [[README|Order Orchestrator]]  
**Type**: API Documentation

---

## 📡 Server Configuration

| Protocol | Port | Default |
|----------|------|---------|
| gRPC | `GRPC_PORT` | 6000 |
| REST (gRPC-Gateway) | `REST_PORT` | 8000 |

---

## 🔧 gRPC Methods

### Health Check

#### HealthCheck
```protobuf
rpc HealthCheck(EmptyRequest) returns (DefaultMessage);
```

---

### Cart Operations

#### CreateCart
Create a new shopping cart.
```protobuf
rpc CreateCart(EmptyRequest) returns (CreateCartResponse);
```
- **Output**:
  ```protobuf
  message CreateCartResponse {
    string cart_id = 1;
  }
  ```

#### AddCartItem
Add item to cart from session.
```protobuf
rpc AddCartItem(AddToCartRequest) returns (AddToCartResponse);
```
- **Input**:
  ```protobuf
  message AddToCartRequest {
    string session_id = 1;
    string fleet_id = 2;
    string category_type = 3;
    string business_type = 4;
    string cart_id = 5;
  }
  ```
- **Output**:
  ```protobuf
  message AddToCartResponse {
    string cart_item_id = 1;
  }
  ```

#### DeleteCartItem
Remove item from cart.
```protobuf
rpc DeleteCartItem(DeleteCartItemRequest) returns (DefaultMessage);
```
- **Input**: `{cart_id, cart_item_id}`

#### SetQuantity
Update item quantity in cart.
```protobuf
rpc SetQuantity(SetQuantityRequest) returns (DefaultMessage);
```
- **Input**: `{cart_id, cart_item_id, quantity}`

#### GetCart
Get cart with all items and calculated pricing.
```protobuf
rpc GetCart(GetCartRequest) returns (GetCartResponse);
```
- **Input**:
  ```protobuf
  message GetCartRequest {
    string cart_id = 1;
    string promo_code = 2;
  }
  ```
- **Output**:
  ```protobuf
  message GetCartResponse {
    string cart_id = 1;
    repeated Fleet fleet = 2;
    Fare total_cart_price = 3;
    CartData cart_data = 4;
  }
  
  message CartData {
    double total_price_amount = 1;
    double discount_amount = 2;
    double final_price_amount = 3;
  }
  ```

---

### Order Operations

#### CreateOrder
Create order from cart (MyBB App channel).
```protobuf
rpc CreateOrder(OrderRequest) returns (OrderResponse);
```
- **Input**:
  ```protobuf
  message OrderRequest {
    string cart_id = 1;
    CustomerInformation customer = 2;
    repeated PassengerInformation passenger = 3;
    string promo_code = 4;
    string session_id = 5;
  }
  
  message CustomerInformation {
    string salutation = 1;
    string name = 2;
    string phone_number = 3;
    string email = 4;
  }
  
  message PassengerInformation {
    string name = 1;
    string phone_number = 2;
    string email = 3;
    string flight_number = 4;
    string note_to_driver = 5;
  }
  ```
- **Output**:
  ```protobuf
  message OrderResponse {
    string external_booking_id = 1;
    repeated Fleet fleet = 2;
    Payment payment = 3;
    string customer_name = 4;
    string customer_phone_number = 5;
    string customer_email = 6;
    CartData data = 7;
    Fare total_cart_price = 8;
  }
  ```

#### CreateOrderWeb
Create order from web reservation channel.
```protobuf
rpc CreateOrderWeb(CreateOrderWebRequest) returns (CreateOrderWebResponse);
```
- **Input**:
  ```protobuf
  message CreateOrderWebRequest {
    string session_id = 1;
    string fleet_id = 2;
    string promo_code = 3;
    ContactInformationWeb contact_information = 4;
    PassengerInformationWeb passenger_information = 5;
  }
  ```
- **Output**:
  ```protobuf
  message CreateOrderWebResponse {
    string order_id = 1;
    string payment_url = 2;
  }
  ```

#### GetBookingDetail
Get booking details by external booking ID.
```protobuf
rpc GetBookingDetail(BookingDetailRequest) returns (BookingDetailResponse);
```
- **Input**: `{external_booking_id}`
- **Output**:
  ```protobuf
  message BookingDetailResponse {
    string external_booking_id = 1;
    Payment payment = 2;
    string customer_name = 3;
    string customer_phone_number = 4;
    string customer_email = 5;
    repeated Fleet fleet = 20;
    TransactionFare fare = 21;
    TransactionData data = 22;
  }
  ```

#### GetPayment
Get payment details.
```protobuf
rpc GetPayment(PaymentUrlRequest) returns (Payment);
```
- **Input**: `{external_payment_id}`
- **Output**:
  ```protobuf
  message Payment {
    string external_payment_id = 1;
    string expired_at = 3;
    string payment_method = 4;
    string identifier = 5;
    double amount = 6;
    string payment_at = 7;
    string status = 8;
    string payment_display_name = 9;
    string payment_method_icon = 10;
    string payment_url = 11;
    Promo promo = 20;
  }
  ```

#### GetOrderReceipt
Get receipt URL for completed order.
```protobuf
rpc GetOrderReceipt(GetOrderReceiptRequest) returns (GetOrderReceiptResponse);
```
- **Input**: `{order_id}`
- **Output**: `{receipt_url}`

---

### Order Item Details

#### GetRideDetail
Get taxi/ride order item details.
```protobuf
rpc GetRideDetail(OrderItemId) returns (RideOrderItemDetail);
```
- **Input**: `{external_order_item_id}`
- **Output**:
  ```protobuf
  message RideOrderItemDetail {
    OrderItem order_item = 1;
    RideDetail ride_detail = 2;
  }
  
  message RideDetail {
    string driver_name = 1;
    string driver_phone = 2;
    string vehicle_plate = 3;
    string vehicle_model = 4;
    string vehicle_brand = 5;
    string pickup_location = 6;      // JSON string
    string dropoff_location = 7;     // JSON string
    string pickup_time = 8;
    string dropoff_time = 9;
    double estimated_distance_km = 10;
    int32 estimated_duration_min = 11;
    double actual_distance_km = 12;
    int32 actual_duration_min = 13;
    string notes = 14;
    bool is_fixed_fare = 15;
    // ... package option fields
  }
  ```

#### GetRentDetail
Get rental order item details.
```protobuf
rpc GetRentDetail(OrderItemId) returns (RentOrderItemDetail);
```
- **Output**:
  ```protobuf
  message RentDetail {
    string driver_name = 1;
    string driver_phone = 2;
    string vehicle_plate = 3;
    string vehicle_model = 4;
    string vehicle_brand = 5;
    string pickup_location = 6;
    string dropoff_location = 7;
    string pickup_time = 8;
    string dropoff_time = 9;
    int32 rent_hours = 10;
    double extra_time_fee = 11;
    string package_code = 12;
    // ... package option fields
  }
  ```

#### GetDeliveryDetail
Get delivery order item details.
```protobuf
rpc GetDeliveryDetail(OrderItemId) returns (DeliveryOrderItemDetail);
```
- **Output**:
  ```protobuf
  message DeliveryDetail {
    string sender_name = 1;
    string sender_phone = 2;
    string recipient_name = 3;
    string recipient_phone = 4;
    string sender_photo_url1 = 5;
    string sender_photo_url2 = 6;
    string recipient_photo_url1 = 7;
    string recipient_photo_url2 = 8;
    string list_of_goods = 9;
    string pickup_location = 15;    // JSON string
    string dropoff_location = 16;   // JSON string
    string pickup_time = 17;
    string dropoff_time = 18;
    double estimated_distance_km = 19;
    int32 estimated_duration_min = 20;
  }
  ```

---

### Callbacks (Webhooks)

#### CallbackPaymentInformation
Handle payment status webhook from Payment Processor.
```protobuf
rpc CallbackPaymentInformation(CallbackPaymentRequest) returns (DefaultMessage);
```
- **Input**:
  ```protobuf
  message CallbackPaymentRequest {
    string transaction_id = 1;
    string partner_transaction_id = 2;
    string order_id = 3;
    string transaction_type = 4;
    double amount = 5;
    string status = 6;
    string payment_method = 7;
    string payment_method_name = 8;
    string bank_name = 9;
    string register_time = 10;
    string deadline_time = 11;
    string paid_time = 12;
    string webhook_url = 13;
    string reference_number = 14;
    string card_number = 15;
    string approval_code = 16;
    string payment_gateway = 17;
    MDRData mdr_data = 21;
  }
  ```

#### CallbackOrderItem
Handle order item status update callback.
```protobuf
rpc CallbackOrderItem(CallbackOrderItemRequest) returns (DefaultMessage);
```
- **Input**: Full order item data with status update

#### CallbackReceipt
Handle receipt generation callback.
```protobuf
rpc CallbackReceipt(CallbackReceiptRequest) returns (DefaultMessage);
```
- **Input**: `{booking_id, order_id, file_url, code}`

---

### Web Reservation

#### GetOrderDetailWeb
Get order details for web display.
```protobuf
rpc GetOrderDetailWeb(GetOrderDetailWebRequest) returns (GetOrderDetailWebResponse);
```
- **Input**: `{order_id}`
- **Output**:
  ```protobuf
  message GetOrderDetailWebResponse {
    OrderDetailInfoWeb order_detail = 1;
    PaymentDetailInfoWeb payment_detail = 2;
  }
  
  message OrderDetailInfoWeb {
    string order_id = 1;
    string city_name = 2;
    string service_name = 3;
    string service = 4;           // rent / airport_transfer / shuttle
    string rental_area = 5;
    FleetDetailWeb fleet_detail = 6;
    string pickup_address = 7;
    string destination_address = 8;
    string pickup_at = 9;         // RFC3339 timestamp
    DurationInfoWeb duration = 10;
  }
  
  message PaymentDetailInfoWeb {
    string promo_code = 1;
    double total = 2;
    repeated PaymentComponentWeb components = 3;
    PaymentMethodInfoWeb payment_method = 4;
    string status = 5;            // success / pending / failed
  }
  ```

#### PromoValidate
Validate promo code and calculate discount.
```protobuf
rpc PromoValidate(PromoValidateRequest) returns (PromoValidateResponse);
```
- **Input**: `{session_id, fleet_id, promo_code}`
- **Output**:
  ```protobuf
  message PromoValidateResponse {
    double fare_before_promo = 1;
    double fare_after_promo = 2;
    double promo_amount = 3;
    bool is_promo_valid = 4;
  }
  ```

---

### Internal Operations

#### GetInternalBookingDetail
Get complete booking details for internal services.
```protobuf
rpc GetInternalBookingDetail(GetInternalBookingDetailRequest) returns (GetInternalBookingDetailResponse);
```
- **Input**: `{id}`
- **Output**: Complete booking with all related entities (payments, order_items, passenger_info, ride_detail, rent_detail, vehicle_info, service_fees, fare_components)

---

## 📝 Common Types

### Fleet
```protobuf
message Fleet {
  string cart_id = 1;
  string fleet_id = 2;
  string title = 4;
  Fare Fare = 5;
  repeated Icon icons = 8;
  int32 quantity = 9;
  FleetData data = 10;
  repeated string order_item_id = 11;
}
```

### OrderItem
```protobuf
message OrderItem {
  string id = 1;
  string booking_id = 2;
  string fleet_id = 3;
  string external_order_item = 4;
  string quote_id = 5;
  string business_type = 6;
  int32 service_type_id = 7;
  string service_type_name_en = 8;
  string service_type_name_id = 9;
  string car_type = 10;
  double fare_fixed = 14;
  double fare_normal_price = 15;
  double fare_est_min = 16;
  double fare_est_max = 17;
}
```

### Promo
```protobuf
message Promo {
  string code = 1;
  double discount_amount = 2;
  string status = 3;
}
```

### DefaultMessage
```protobuf
message DefaultMessage {
  string code = 1;
  string message = 2;
}
```

---

## 📊 Business Types

| Code | Type | Description |
|------|------|-------------|
| `TAXI` | Ride | Point-to-point taxi service |
| `RENT` | Rental | Vehicle rental (hourly/daily) |
| `DELIVERY` | Delivery | Package delivery service |
| `SHUTTLE` | Shuttle | Fixed route transportation |

---

## 📊 Payment Status

| Status | Description |
|--------|-------------|
| `REQUEST_PAYMENT_LINK` | Requesting payment URL |
| `WAITING_FOR_PAYMENT` | Awaiting customer payment |
| `PAID` | Payment successful |
| `CANCELED` | Payment cancelled |
| `FAILED` | Payment failed |
| `REQUEST_REFUND` | Refund in progress |
| `REFUND_SUCCESS` | Refund completed |

---

## 📊 Booking Status

| Status | Description |
|--------|-------------|
| `CREATED` | Booking created, awaiting payment |
| `ACTIVE` | Payment success, services being booked |
| `COMPLETED` | All services completed |
| `FAILED` | Saga failed, compensated |
| `CANCELED` | Booking cancelled |

---

## 🏷️ Tags

#api #grpc #orderorchestrator #order #booking #mrg #reference

---

*Last Updated*: 2025-01-05
