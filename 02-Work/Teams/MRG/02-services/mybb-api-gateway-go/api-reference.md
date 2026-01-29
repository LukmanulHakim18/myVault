---
tags:
  - api
  - rest
  - grpc
  - api-gateway
  - mrg
  - reference
  - legacy
type: api-documentation
title: MyBB API Gateway - API Reference
parent: mybb-api-gateway-go
---
# MyBB API Gateway - API Reference

**Service**: [[README|MyBB API Gateway]]  
**Type**: API Documentation

---

## 📡 Server Configuration

| Protocol | Port | Description |
|----------|------|-------------|
| REST (Beego) | 3000 | Main HTTP API |
| gRPC (TLS) | 3443 | Streaming & events |
| pprof | 6060 | Debug profiling |
| Swagger | 3000/swagger | API documentation |

---

## 🔐 Authentication

### Token-Based Auth (Legacy)

Request header:
```
Token: <device_token>
Device-Id: <device_uuid>
App-Version: <app_version>
Accept-Language: en|id
```

### OTT (One-Time Token) Flow

1. Get server time: `GET /api/timesync`
2. Encrypt password: `POST /api/auth/encrypt-password`
3. Validate user: `POST /api/auth/validate-user`

---

## 🔧 REST API Endpoints

### Authentication (`/api/auth/*`)

| Method | Endpoint | Description |
|--------|----------|-------------|
| `GET` | `/api/timesync` | Get server epoch time |
| `POST` | `/api/auth/validate-user` | Validate user credentials |
| `POST` | `/api/auth/register` | Register new user |
| `POST` | `/api/auth/check-is-registered` | Check if phone is registered |
| `POST` | `/api/auth/encrypt-password` | Encrypt password with OTT |

#### TimeSync Response
```json
{
  "epoch_server_time": 1706500000
}
```

#### Validate User Request
```json
{
  "country_code": "+62",
  "login_id": "81234567890",
  "login_type": "phone"
}
```

---

### Session Management (`/api/v6/sessions/*`)

| Method | Endpoint | Description |
|--------|----------|-------------|
| `POST` | `/api/v6/sessions` | Sign in (login) |
| `POST` | `/api/v6/sessions/check` | Check session validity |
| `DELETE` | `/api/v6/me/sessions` | Sign out (logout) |

---

### User Management (`/api/v6/users/*`, `/api/v6/me/*`)

| Method | Endpoint | Description |
|--------|----------|-------------|
| `POST` | `/api/v6/users` | Sign up (register) |
| `GET` | `/api/v6/me` | Get current user profile |
| `PATCH` | `/api/v6/me` | Update user profile |
| `POST` | `/api/v6/users/forgot_password` | Request password reset |
| `POST` | `/api/v6/users/reset_password` | Reset password |
| `POST` | `/api/v6/users/change_password` | Change password |
| `POST` | `/api/v6/users/temporary_password` | Generate temporary password |
| `GET` | `/api/v6/users/feature` | Get user feature flags |
| `GET` | `/api/v6/users/reason_delete/:language` | Get deletion reasons |
| `POST` | `/api/v6/users/delete/verify` | Verify delete account |
| `POST` | `/api/v6/users/delete/account` | Delete account |

#### User Profile Response
```json
{
  "data": {
    "internal_id": "BB00123456",
    "email": "user@example.com",
    "phone": "+6281234567890",
    "name": "John Doe",
    "profile_image": "https://...",
    "is_activated": true,
    "is_legacy": false,
    "privileges": ["standard"],
    "enabled_features": {}
  }
}
```

---

### Verification (`/api/v6/verifications/*`)

| Method | Endpoint | Description |
|--------|----------|-------------|
| `POST` | `/api/v6/verifications` | Submit verification code |
| `PATCH` | `/api/v6/verifications/resend` | Resend verification code |

---

### Orders (`/api/v6/me/orders/*`)

| Method | Endpoint | Description |
|--------|----------|-------------|
| `POST` | `/api/v6/me/orders` | Create order |
| `POST` | `/api/v6/me/orders/init` | Initialize order |
| `POST` | `/api/v6/me/orders/retry` | Retry order |
| `GET` | `/api/v6/me/orders/ongoing` | Get ongoing orders |
| `GET` | `/api/v6/me/orders/ongoing/all` | Get all ongoing orders |
| `GET` | `/api/v6/me/orders/history/all` | Get order history |
| `GET` | `/api/v6/me/orders/info` | Get order info |
| `GET` | `/api/v6/me/orders/:product_type/:order_id` | Get order detail |
| `PATCH` | `/api/v6/me/orders/:product_type/:order_id` | Update order |
| `DELETE` | `/api/v6/me/orders/:product_type/:order_id` | Cancel order |
| `POST` | `/api/v6/me/orders/:product_type/:order_id/share_trip` | Share trip |
| `POST` | `/api/v6/me/orders/:product_type/:order_id/share_receipt` | Share receipt |
| `POST` | `/api/v6/me/orders/:product_type/:order_id/hide` | Hide order |
| `POST` | `/api/v6/me/orders/:product_type/:order_id/tips` | Tip driver |
| `POST` | `/api/v6/me/orders/:product_type/:order_id/save_rating` | Save rating |
| `POST` | `/api/v6/me/orders/:product_type/:order_id/recharge` | Recharge payment |
| `POST` | `/api/v6/me/orders/:product_type/:order_id/easy_ride` | Retry easy ride |
| `GET` | `/api/v6/me/orders/:product_type/:order_id/rent_info` | Get rent info |
| `GET` | `/api/v6/me/orders/:product_type/:order_id/check_eligible_reschedule` | Check reschedule |
| `POST` | `/api/v6/me/orders/:product_type/:order_id/reschedule/submit` | Submit reschedule |

#### Create Order Request
```json
{
  "product_type": "bluebird",
  "service_type_id": 1,
  "pickup_location": {
    "latitude": -6.175392,
    "longitude": 106.827153,
    "address": "Monas, Jakarta"
  },
  "destination_location": {
    "latitude": -6.200000,
    "longitude": 106.850000,
    "address": "Kuningan, Jakarta"
  },
  "payment_type": "cash",
  "promo_code": "PROMO123"
}
```

---

### Favorite Addresses (`/api/v6/me/favorite_addresses/*`)

| Method | Endpoint | Description |
|--------|----------|-------------|
| `GET` | `/api/v6/me/favorite_addresses` | List favorites |
| `POST` | `/api/v6/me/favorite_addresses` | Create favorite |
| `PATCH` | `/api/v6/me/favorite_addresses/:id` | Update favorite |
| `DELETE` | `/api/v6/me/favorite_addresses/:id` | Delete favorite |
| `GET` | `/api/v6/me/favorite_addresses/snap_nearest` | Snap to nearest |

---

### Payment Methods (`/api/v6/me/payment_methods/*`)

| Method | Endpoint | Description |
|--------|----------|-------------|
| `GET` | `/api/v6/me/payment_methods` | List payment methods |
| `GET` | `/api/v6/me/payment_methods/supported` | Get supported methods |
| `GET` | `/api/v6/me/payment_methods/:type/available` | Check availability |
| `POST` | `/api/v6/me/payment_methods/change_default` | Change default |
| `GET` | `/api/v6/me/payment_methods/payment_default` | Get default |

---

### Wallets (`/api/v6/me/wallets/*`)

| Method | Endpoint | Description |
|--------|----------|-------------|
| `GET` | `/api/v6/me/wallets/:provider` | Get wallet balance |
| `POST` | `/api/v6/me/wallets/:provider` | Login to wallet |
| `DELETE` | `/api/v6/me/wallets/:provider/:wallet_id` | Unlink wallet |
| `GET` | `/api/v6/me/wallets/:provider/transactions` | Transaction history |
| `POST` | `/api/v6/me/wallets/:provider/webview/login` | WebView login |
| `GET` | `/api/v6/me/wallets/:provider/webview/login/url` | Get login URL |
| `GET` | `/api/v6/me/wallets/:provider/webview/token` | Get token |

Supported providers: `paypro`, `tcash`, `dana`, `isaku`, `gopay`, `shopeepay`, `linkaja`

---

### Credit Cards (`/api/v6/me/credit_cards/*`)

| Method | Endpoint | Description |
|--------|----------|-------------|
| `GET` | `/api/v6/me/credit_cards` | List credit cards |
| `POST` | `/api/v6/me/credit_cards` | Register credit card |
| `DELETE` | `/api/v6/me/credit_cards/:epayment_token` | Remove credit card |

---

### Geolocation (`/api/v6/geolocation/*`)

| Method | Endpoint | Description |
|--------|----------|-------------|
| `GET` | `/api/v6/geolocation/geocode` | Geocode address |
| `GET` | `/api/v6/geolocation/geocode/:location_type` | Geocode with type |
| `GET` | `/api/v6/geolocation/autocomplete` | Autocomplete search |
| `GET` | `/api/v6/geolocation/autocomplete/v2` | Autocomplete v2 |
| `GET` | `/api/v6/geolocation/textsearch` | Text search |
| `GET` | `/api/v6/geolocation/directions` | Get directions |
| `GET` | `/api/v6/geolocation/distancematrix` | Distance matrix |
| `GET` | `/api/v6/geolocation/area` | Get area from geocode |
| `GET` | `/api/v6/geolocation/nearby_location/:type` | Nearby search |

#### Autocomplete Request
```
GET /api/v6/geolocation/autocomplete?input=monas&latitude=-6.175&longitude=106.827
```

#### Directions Request
```
GET /api/v6/geolocation/directions?origin=-6.175,106.827&destination=-6.200,106.850
```

---

### Landmarks (`/api/v6/landmarks/*`)

| Method | Endpoint | Description |
|--------|----------|-------------|
| `GET` | `/api/v6/landmarks/nearby` | Nearby landmarks |
| `GET` | `/api/v6/landmarks/nearby/:location_type` | Nearby with type |
| `GET` | `/api/v6/landmarks/suggestions` | Suggested landmarks |

---

### Service Types (`/api/v6/service_types/*`)

| Method | Endpoint | Description |
|--------|----------|-------------|
| `GET` | `/api/v6/service_types` | Get all service types |
| `GET` | `/api/v6/service_types/info` | Get service type info |
| `GET` | `/api/v6/service_types/goldenbird_detail` | Get GB detail |
| `GET` | `/api/v6/service_types/calculate_min_fare` | Calculate minimum fare |

---

### Golden Bird (`/api/v6/golden_bird/*`, `/api/v6/me/golden_bird/*`)

| Method | Endpoint | Description |
|--------|----------|-------------|
| `GET` | `/api/v6/golden_bird/area` | Get GB area |
| `GET` | `/api/v6/golden_bird/service_types` | GB service types |
| `GET` | `/api/v6/golden_bird/call_centers` | GB call centers |
| `POST` | `/api/v6/me/golden_bird/orders` | Create GB order |
| `GET` | `/api/v6/me/golden_bird/orders` | List GB orders |
| `GET` | `/api/v6/me/golden_bird/orders/:order_id` | Get GB order |
| `DELETE` | `/api/v6/me/golden_bird/orders/:order_id` | Cancel GB order |
| `POST` | `/api/v6/me/golden_bird/orders/:order_id/hide` | Hide GB order |
| `POST` | `/api/v6/me/golden_bird/orders/:order_id/rate` | Rate GB order |

---

### Marketing & Promotions (`/api/v6/me/*`)

| Method | Endpoint | Description |
|--------|----------|-------------|
| `POST` | `/api/v6/me/validate_promotion_code` | Validate promo |
| `POST` | `/api/v6/me/validate_promotion_discount` | Validate discount |
| `GET` | `/api/v6/me/promotion/areas/:promo_code` | Get promo areas |
| `POST` | `/api/v6/me/promotions/exploration/list/:id/:lang` | Exploration list |
| `POST` | `/api/v6/me/promotions/validate/auto-apply` | Auto-apply promo |
| `POST` | `/api/v6/me/promotions/validate/pre-validation` | Pre-validate promo |
| `GET` | `/api/v6/me/loyalty/customer` | Get loyalty customer |
| `GET` | `/api/v6/me/loyalty/customer/info` | Get loyalty info |
| `POST` | `/api/v6/me/loyalty/share/reward` | Share reward |
| `GET` | `/api/v6/me/loyalty/history/:order_id` | Get history points |
| `GET` | `/api/v6/me/referral` | Get referral code |
| `POST` | `/api/v6/me/referral/validate` | Validate referral |
| `GET` | `/api/v6/me/referral/list` | Get referral list |

---

### Subscriptions (`/api/subscriptions/*`)

| Method | Endpoint | Description |
|--------|----------|-------------|
| `GET` | `/api/subscriptions` | Get subscriptions by user |
| `POST` | `/api/subscriptions` | Get by param |
| `GET` | `/api/subscriptions/bulk` | Get bulk subscriptions |
| `GET` | `/api/subscriptions/detail` | Get subscription detail |
| `POST` | `/api/subscriptions/checkout` | Checkout subscription |
| `GET` | `/api/subscriptions/order/history` | Order history |
| `POST` | `/api/subscriptions/status` | Update status |

---

### Static Data

| Method | Endpoint | Description |
|--------|----------|-------------|
| `GET` | `/api/v6/area` | Get all areas |
| `GET` | `/api/v6/contact_centers` | Get contact centers |
| `GET` | `/api/v6/feedbacks` | Get feedback options |
| `GET` | `/api/v6/feedbacks/category/:lang` | Get feedback categories |
| `POST` | `/api/v6/feedbacks/submit` | Submit feedback |
| `GET` | `/api/v6/cancellation_reasons` | Get cancellation reasons |
| `GET` | `/api/v6/file_resources` | List file resources |
| `GET` | `/api/v6/file_resources/:id` | Get file resource |
| `GET` | `/api/v6/info_protocol/:lang` | Get info protocol |
| `GET` | `/api/v6/announcements` | Get announcements |
| `GET` | `/api/v6/static/app_version` | Get app version |
| `GET` | `/api/v6/eta_info` | Get ETA info |

---

### Tools (Internal) (`/api/v6/tools/*`)

| Method | Endpoint | Description |
|--------|----------|-------------|
| `POST` | `/api/v6/tools/send_sms_count` | Send SMS count |
| `POST` | `/api/v6/tools/send_email_count` | Send email count |
| `POST` | `/api/v6/tools/blast_notification` | Blast notification |
| `POST` | `/api/v6/tools/recharge_epay` | Recharge epay |
| `POST` | `/api/v6/tools/regenerate_trip_map/:order_id` | Regenerate map |
| `POST` | `/api/v6/tools/orders/finish` | Consult and finish |
| `DELETE` | `/api/v6/tools/cancel_by_system` | Cancel by system |
| `POST` | `/api/v6/tools/reset/limit/otp` | Reset OTP limit |
| `POST` | `/api/v6/tools/whitelist_phone_numbers` | Whitelist phones |
| `DELETE` | `/api/v6/tools/delete_token` | Delete token |

---

### Webhooks (`/webhook/*`)

| Method | Endpoint | Description |
|--------|----------|-------------|
| `POST` | `/webhook/midtrans/payment_notification` | Midtrans callback |
| `POST` | `/api/webhook` | OTP delivery status |

---

### SOAP (Legacy) (`/soap/*`)

| Method | Endpoint | Description |
|--------|----------|-------------|
| `GET` | `/soap/itop/wsdl` | Get WSDL |
| `POST` | `/soap/itop/service` | SOAP service |

---

## 🔧 gRPC Services

### LocationService

```protobuf
service LocationService {
  rpc GetCarLocation(CarLocationRequest) returns (stream CarLocationResponse);
  rpc GetNearbyCars(NearbyCarsRequest) returns (NearbyCarsResponse);
}
```

### OdrdLocationService

```protobuf
service OdrdLocationService {
  rpc CreateConsumerToken(CreateTokenRequest) returns (TokenResponse);
  rpc CreateDriverToken(CreateTokenRequest) returns (TokenResponse);
  rpc CreateTripId(TripIdRequest) returns (TripIdResponse);
}
```

### OrderService

```protobuf
service OrderService {
  rpc GetOrderChanges(OrderRequest) returns (stream OrderChangeEvent);
}
```

### SchoolBusService

```protobuf
service SchoolBusService {
  rpc GetBusLiveTrackingStatus(TrackingRequest) returns (TrackingResponse);
}
```

### EventService (Feature Flagged)

```protobuf
service EventService {
  rpc SubscribeEvents(EventRequest) returns (stream Event);
}
```

---

## 📝 Common Response Format

### Success Response
```json
{
  "data": { ... }
}
```

### Success with Boolean
```json
{
  "success": true
}
```

### Error Response
```json
{
  "error_code": "E001",
  "message": "Error message",
  "localized_message": {
    "english": "Error message",
    "indonesia": "Pesan error"
  }
}
```

---

## 🏷️ Tags

#api #rest #grpc #api-gateway #mrg #reference #legacy

---

*Last Updated*: 2026-01-29
