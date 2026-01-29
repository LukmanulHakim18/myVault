---
tags:
  - mrg
  - service
  - legacy
  - order-processing
  - kafka
  - monolith
  - documentation
  - needs-decomposition
team: MRG
type: service-documentation
title: Order Processing Service (Legacy)
status: legacy
created: '2026-01-29'
updated: '2026-01-29'
kafka_topics: 91
rest_port: REST_API_PORT
repository: git.bluebird.id/mybb-legacy/mybb-app/src/mybb-order-processing-go
tech_stack:
  - go
  - kafka
  - postgresql
  - mongodb
  - redis
  - gcs
legacy: true
decomposition_candidate: true
---
# Order Processing Service (Legacy)

**Team**: MRG (Meta Reservation Gateway)  
**Status**: ⚠️ Production Legacy  
**Repository**: `git.bluebird.id/mybb-legacy/mybb-app/src/mybb-order-processing-go`

---

## 📋 Overview

MyBB Order Processing adalah **legacy monolithic service** yang bertanggung jawab untuk mengelola seluruh lifecycle order dalam ekosistem MyBluebird. Service ini merupakan salah satu core service tertua yang menangani 91 Kafka topics untuk berbagai operasi order.

### Fungsi Utama

- **Order Lifecycle Management** - Create, update, cancel booking
- **Trip Management** - Start trip, end trip, trip history
- **Payment Integration** - Wallet, credit card, e-money, pre-auth
- **Rating & Feedback** - Driver rating, order rating, feedback
- **Trip Receipt** - Generate receipt, share receipt via email
- **Open API B2B** - Merchant order management
- **Golden Bird Integration** - GB Airport, Rental, Limo
- **BBD Integration** - Dispatch system (ITOP)

### ⚠️ Legacy Warning

Service ini adalah **kandidat untuk dekomposisi** karena:
- Monolithic architecture handling 91 Kafka topics
- Mixed database strategy (PostgreSQL + MongoDB)
- Dual Redis versions (v4 + v8)
- Tight coupling dengan mybb-common package
- 8 tahun schema evolution (77 migrations, 2016-2024)

---

## 🛠️ Tech Stack

| Component | Technology |
|-----------|------------|
| Language | Go 1.23 |
| Transport | Kafka (primary), REST HTTP (secondary) |
| Database | PostgreSQL (GORM) |
| Document Store | MongoDB |
| Cache | Redis v4 + Redis v8 |
| Message Queue | Kafka, RabbitMQ |
| Cloud Storage | Google Cloud Storage |
| External Integration | ITOP (XML/SOAP) |
| Monitoring | Elastic APM |
| Feature Flags | Unleash |

---

## 🔑 Konsep Utama

### 1. Kafka-First Architecture
- 91 Kafka topics sebagai primary transport
- REST HTTP hanya untuk internal/debugging
- Consumer group strategy untuk reliability

### 2. Order State Machine
```
INITIAL → LOOKING_FOR_DRIVER → DRIVER_FOUND → ON_TRIP → TRIP_ENDED → COMPLETED
                ↓                                           ↓
           CANCELLED                                   RATING_PENDING
```

### 3. Payment Methods
| Code | Payment Type |
|------|--------------|
| `cash` | Cash |
| `ecv` | E-Corporate Voucher |
| `cc` | Credit Card |
| `tcash` | LinkAja (legacy) |
| `trip_voucher` | Trip Voucher |
| `dana` | DANA |
| `isaku` | iSaku |
| `gopay` | GoPay |
| `shopeepay` | ShopeePay |

### 4. Service Types
| ID | Service |
|----|---------|
| 0 | BlueBird Regular |
| 1 | SilverBird Regular |
| 2 | Golden Bird Airport |
| 3 | Golden Bird Rental |
| 9 | Easy Ride |
| 98 | Fixed Fare |
| 99 | BirdKirim |

### 5. Consumer Group Strategy
- **Event consumers** (no group, lose data on restart): UPDATE_CURRENT_BOOKING, UPDATE_CHARGE_STATUS
- **Simple consumers** (no group): ORDER_CHANGES stream
- **Group consumers** (default): Semua topic lainnya dengan guaranteed delivery

---

## 🔌 Dependencies

### Upstream Services (yang memanggil service ini)

| Service | Protocol | Topics/Endpoints |
|---------|----------|------------------|
| **API Gateway** | Kafka | REQUEST_BOOKING, CANCEL_BOOKING, END_TRIP, dll |
| **BBD Dispatch** | Kafka | ORDER_CALLBACK, UPDATE_CURRENT_BOOKING |
| **Payment Processor** | Kafka | UPDATE_CHARGE_STATUS, UPDATE_REFUND_STATUS |
| **Open API** | Kafka | OPEN_API_* topics |

### Downstream Services (yang dipanggil service ini)

| Service | Protocol | Purpose |
|---------|----------|---------|
| **ITOP** | HTTP/SOAP | Dispatch system integration |
| **Payment Processor** | HTTP REST | Payment operations, wallet, CC |
| **Meta Payment Gateway** | HTTP REST | ECV, trip voucher, payment methods |
| **Location Registry** | HTTP REST | Area lookup, service type, airport |
| **Price Manager** | HTTP REST | Carbon saving calculation |
| **API Gateway** | HTTP REST | User feature flags |
| **Corporate Portal** | HTTP REST | ECV validation |
| **Golden Bird** | HTTP REST | GB order processing |
| **GB Limo** | HTTP REST | GB Limo service |
| **PubSub** | HTTP REST | Event publishing to MKT |
| **Maps Service (GMO)** | HTTP REST | Geocoding, directions, static maps |

### Infrastructure Dependencies

| Component | Purpose |
|-----------|---------|
| **PostgreSQL** | Primary database (orders, fare, surge prices) |
| **MongoDB** | Order info, order state (extended data) |
| **Redis v4** | Legacy cache operations |
| **Redis v8** | gRPC cache, stream, rate limiting |
| **Kafka** | Message broker (91 topics) |
| **RabbitMQ** | Secondary message queue |
| **GCS** | Trip images storage |

Lihat detail di: [[dependencies|Dependencies Detail]]

---

## 📡 API Contracts

### Kafka Topics (91 Topics)

#### Core Order Operations (8 topics)
| Topic | Consumer | Description |
|-------|----------|-------------|
| `TOPIC_REQUEST_BOOKING` | RequestBookingConsumer | Create new booking |
| `TOPIC_UPDATE_CURRENT_BOOKING` | UpdateCurrentBookingConsumer | Update ongoing booking |
| `TOPIC_CANCEL_BOOKING` | CancelBookingConsumer | Cancel by user |
| `TOPIC_CANCEL_BOOKING_BY_SYSTEM` | CancelBookingBySystemConsumer | System cancellation |
| `TOPIC_END_TRIP` | EndTripConsumer | End trip (from BBD) |
| `TOPIC_UPDATE_ORDER` | UpdateOrderConsumer | Generic update |
| `TOPIC_ORDER_CHANGES` | OrderChangesConsumer | Stream changes |
| `TOPIC_ORDER_CALLBACK` | OrderCallbackConsumer | BBD callback |

#### Payment & Billing (8 topics)
| Topic | Consumer | Description |
|-------|----------|-------------|
| `TOPIC_RECHARGE_BOOKING` | RechargeBookingConsumer | Recharge order |
| `TOPIC_UPDATE_CHARGE_STATUS` | UpdateChargeStatusConsumer | Payment status update |
| `TOPIC_UPDATE_REFUND_STATUS` | UpdateRefundStatusConsumer | Refund status update |
| `TOPIC_CANCELLATION_FEE` | CancellationFeeConsumer | Calculate cancel fee |
| `TOPIC_TIPS_DRIVER` | TipsDriverConsumer | Driver tipping |

Lihat semua 91 topics di: [[kafka-topics|Kafka Topics Reference]]

### REST API Endpoints

| Method | Endpoint | Handler | Description |
|--------|----------|---------|-------------|
| GET | `/ping` | - | Health check |
| POST | `/orders` | orderhttp.Create | Create order |
| GET | `/active_orders` | orderhttp.GetActiveOrders | Get active orders |
| GET | `/last_order` | orderhttp.GetLastOrder | Get last order |
| GET | `/cancellation_fee/{order_id}` | orderhttp.CancellationFee | Get cancellation fee |
| GET | `/order_info/all` | orderhttp.GetOrderInfoAll | Get all order info |
| POST | `/order_info` | orderhttp.UpdateOrderInfo | Update order info |

Lihat detail di: [[api-reference|API Reference]]

---

## 🔄 Order Flows

### Request Booking Flow
```
API Gateway → REQUEST_BOOKING
    ↓
RequestBookingConsumer
    ↓
Validate User (soft ban, hard ban)
    ↓
Get Area & Service Type (Location Registry)
    ↓
Validate Payment Method
    ↓
[If CC Pre-auth] → Payment Processor → Wait Callback
    ↓
[If Wallet] → Reserve Balance
    ↓
Create Order (PostgreSQL)
    ↓
Create Order Info (MongoDB)
    ↓
[If GB] → Golden Bird Service
    ↓
[If Regular] → BBD/ITOP Dispatch
    ↓
NOTIFY_BOOKING_RESULT → API Gateway
```

### End Trip Flow
```
BBD/ITOP → END_TRIP
    ↓
EndTripConsumer
    ↓
Validate Order State
    ↓
Calculate Final Fare
    ↓
[If Wallet] → Charge Wallet
    ↓
Update Order State
    ↓
Generate Trip Image (GCS)
    ↓
Send Receipt Email
    ↓
Notify Order Changes
```

Lihat detail di: [[order-flows|Order Flows Detail]]

---

## ⚙️ Configuration

### Database Configuration
```env
DB_USERNAME=postgres
DB_NAME=mybb_order_processing_db
DB_PASSWORD=xxx
DB_PORT=5432
DB_HOST=database
DB_SSL_MODE=disable
DB_MAX_OPEN_CONNS=1024
DB_MAX_IDLE_CONNS=10
```

### ITOP Configuration
```env
ITOP_SERVER=http://192.168.0.206:8111/mobile/mobileService?wsdl
ITOP_MAX_IDLE_CONNS=400
ITOP_CONN_TIMEOUT=30s
```

### Kafka Configuration
```env
KAFKA_BROKER_LIST=kafka:9092
KAFKA_PARTITION_LIST=0
```

### Payment Configuration
```env
PAYMENT_PROCESSOR_SERVER=http://payment_processor:9090
PAYMENT_GATEWAY_SERVER_V2=http://meta_payment_gateway_v2_go:3009
WALLET_PAYMENT_METHODS=paypro,tcash,dana,isaku
MAX_LINKAJA_CONCURRENT_BOOKING=2
MAX_DANA_CONCURRENT_BOOKING=2
```

### Feature Flags
```env
PRE_AUTH_FEATURE_FLAG=false
PRE_AUTH_CC_SUPPORTED_APP_VERSION=6.2.0
USE_NEW_PROMO_ENGINE=false
EVENT_CHANGE_FEATURE_FLAG=false
```

### GCS Configuration
```env
IMAGE_STORAGE=cloud
GCS_BUCKET_IMAGE_TRIP=mybb-images-dev-00
GOOGLE_APPLICATION_CREDENTIALS=/var/conf/storage.json
```

---

## 📂 Project Structure

```
mybb-order-processing-go/
├── main.go                      # Entry point (Kafka + REST)
├── go.mod                       # Go 1.23
├── .env.sample                  # Environment template
│
├── cmd/                         # CLI commands
│   └── kafka.go
│
├── router/                      # Kafka router
│   └── router.go                # 91 topics registration
│
├── consumers/                   # ~160 consumer files
│   ├── base.go                  # Base consumer (840 lines)
│   ├── factory.go               # Topic → Consumer mapping
│   ├── request_booking.go       # 1144 lines
│   ├── end_trip.go              # 435 lines
│   ├── cancel_booking.go
│   ├── bbd/                     # BBD-specific consumers
│   ├── goldenbird/              # GB consumers
│   ├── fare/                    # Fare-related consumers
│   ├── auto_retry/              # Retry consumers
│   └── order_info/              # Order info consumers
│
├── controllers/                 # REST controllers
│   ├── active_orders.go
│   ├── cancellation_fee.go
│   ├── order_info.go
│   └── order_upsell.go
│
├── services/                    # Business logic
│   ├── create-order/            # Order creation service
│   ├── goldenbird/              # GB service
│   ├── order-info/              # Order info service
│   └── rent-multidays/          # Rental service
│
├── repository/                  # Data access
│   ├── repository.go            # Base repository
│   ├── order.go                 # 1601 lines, OrderRepository
│   ├── mongo/                   # MongoDB repositories
│   └── querier/                 # Custom queries
│
├── models/                      # Domain models
│   ├── order.go                 # 878 lines, 60+ fields
│   ├── delivery_information.go
│   └── order_fares.go
│
├── database/                    # DB setup
│   ├── database.go
│   ├── redis.go
│   └── migrations/              # 77 migrations (2016-2024)
│
├── itop/                        # ITOP integration
│   ├── models/
│   ├── templates/               # XML templates
│   └── webservice/
│
├── payment-processor/           # Payment processor client
├── meta-payment-gateway/        # MPG client
├── location-registry/           # Location registry client
├── price-manager/               # Price manager client
├── goldenbird/                  # GB webservice
├── pubsub/                      # PubSub client
│
├── email/                       # Email templates
│   └── templates/               # HTML templates
│
├── lib/                         # Libraries
│   ├── google/                  # Google Maps
│   ├── gmo/                     # Maps service
│   ├── imagestorage/            # GCS storage
│   └── calculator/              # Fare calculator
│
├── open_api/                    # B2B Open API
│   ├── http/
│   └── service/
│
├── constants/                   # Local constants
├── messages/                    # Kafka messages
└── utils/                       # Utilities
```

---

## 🔒 Security Measures

### Order Creation Security
- **Redis Lock**: Prevent duplicate order creation per user
- **Soft Ban**: Users dengan banyak cancel di-restrict
- **Hard Ban**: Users yang di-blacklist tidak bisa order

### Payment Security
- **Pre-auth CC**: Hold amount sebelum trip
- **Wallet Reserve**: Reserve balance untuk e-wallet
- **Max Concurrent Booking**: Limit per payment method

### Rate Limiting
- **Order Creation**: 10s delay between orders
- **OTP**: Max retry dengan cooldown

---

## 🔴 Technical Debt & Recommendations

### High Priority
1. **Service Decomposition** - Split 91 topics ke domain services
2. **Database Consolidation** - Evaluate MongoDB necessity

### Medium Priority
3. **Redis Migration** - Remove v4, migrate to v8
4. **Consumer Organization** - Group by domain in subdirectories
5. **Test Coverage** - Add comprehensive tests

### Low Priority
6. **Documentation** - Add inline documentation
7. **Migration Cleanup** - Review 77 migrations

---

## 🔗 Related Documentation

- [[dependencies|Dependencies Detail]]
- [[order-flows|Order Flows]]
- [[kafka-topics|Kafka Topics Reference]]
- [[api-reference|API Reference]]
- [[02-Work/Teams/MRG/00-overview/README|MRG Team Overview]]

---

## 🏷️ Tags

#mrg #service #legacy #order-processing #kafka #monolith #documentation

---

*Last Updated*: 2026-01-29  
*Generated from*: Repository analysis at `D:\code\go\mybb-ms\mybb-order-processing-go`
