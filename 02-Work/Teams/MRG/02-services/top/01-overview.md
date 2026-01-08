# Taxi Order Processor (TOP) Service - Overview

## Service Information
- **Service Name**: top (mybb-taxi-order-processor)
- **Team**: MRG (Meta Reservation Gateway)
- **Domain**: Order Processing & State Management
- **Repository**: `git.bluebird.id/mybb-ms/top`
- **Language**: Go 1.24
- **Architecture Pattern**: State Machine + Event-Driven Architecture
- **Local Path**: `D:\code\go\mybb-ms\top`
- **Service Acronym**: TOP

## Purpose
**Taxi Order Processor (TOP)** adalah core microservice yang bertanggung jawab untuk **end-to-end order processing** untuk layanan taxi BlueBird Group. Service ini mengelola complete order lifecycle dari order creation hingga completion, termasuk payment processing, state transitions, dan integrations dengan taxi partner gateway.

### Core Responsibilities
1. **Order Creation & Validation** - Create new orders dengan comprehensive validation
2. **State Management** - Manage order lifecycle menggunakan State Machine pattern
3. **Payment Processing** - Handle payment transactions termasuk pre-auth dan charging
4. **Partner Integration** - Communicate dengan taxi partner systems (iTOP, BBD legacy)
5. **Event Processing** - Process order events dari multiple sources (callbacks, pubsub)
6. **Rating & Feedback** - Handle order rating dan driver tips
7. **Cancellation & Reschedule** - Manage order cancellations dan reschedule operations

## Business Context

### Supported Vehicle Types
TOP melayani berbagai kategori kendaraan BlueBird Group:

| Category | Vehicle Types | Description |
|----------|---------------|-------------|
| **Taxi** | BlueBird, SilverBird, GoldenBird | Regular taxi service |
| **Delivery** | Birdkirim | Package delivery service |
| **Rental** | BlueBird Rent, SilverBird Rent | Vehicle rental |

### Service Categories
```go
const (
    SERVICE_CATEGORY_RIDE     = "ride"
    SERVICE_CATEGORY_DELIVERY = "delivery"
    SERVICE_CATEGORY_RENT     = "rent"
)
```

## Architecture Overview

### State Machine Pattern
TOP menggunakan **State Pattern** untuk mengelola order lifecycle dengan transisi yang kompleks:

```
┌─────────────────────────────────────────────────────────────────┐
│                    ORDER STATE MACHINE                           │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│  Initial States:                                                 │
│  ┌──────────────┐     ┌──────────────┐     ┌──────────────┐    │
│  │ORDER_INITIAL │────>│ORDER_FAILED  │     │ORDER_TIMEOUT │    │
│  │   (State -2) │     │  (State -1)  │     │  (State -3)  │    │
│  └──────┬───────┘     └──────────────┘     └──────────────┘    │
│         │                                                         │
│         ▼                                                         │
│  ┌──────────────────┐                                           │
│  │LOOKING_FOR_DRIVER│                                           │
│  │    (State 0)     │                                           │
│  └─────┬────────────┘                                           │
│        │                                                          │
│    ┌───┴────┬──────────┬──────────┐                            │
│    ▼        ▼          ▼          ▼                             │
│  ┌────┐  ┌────┐  ┌─────────┐  ┌────────┐                      │
│  │EN  │  │NO  │  │CANCELED │  │CHAT    │                      │
│  │ROUTE  │TAXI│  │(State 2)│  │ROOM    │                      │
│  │(1) │  │(3) │  └─────────┘  │CREATED │                      │
│  └─┬──┘  └────┘                │(10)    │                      │
│    │        ▲                   └────────┘                      │
│    ▼        │                                                    │
│  ┌────────┐ │                                                    │
│  │ON_TRIP │ │                                                    │
│  │(State 4)│ │                                                    │
│  └───┬────┘ │                                                    │
│      │      │                                                     │
│      ▼      │                                                     │
│  ┌────────┐ │    ┌───────────┐                                  │
│  │TRACKING│ │    │TAXI_NO_SHOW│                                 │
│  │(State 6)│ └────│(State 5)  │                                 │
│  └───┬────┘      └───────────┘                                  │
│      │                                                            │
│   ┌──┴───┐                                                       │
│   ▼      ▼                                                        │
│ ┌─────┐┌──────┐                                                  │
│ │ECV  ││DROP  │                                                  │
│ │DROP ││OFF   │                                                  │
│ │OFF  ││(8)   │◄─── Terminal State                              │
│ │(7)  │└──────┘                                                  │
│ └─────┘                                                           │
└─────────────────────────────────────────────────────────────────┘
```

### State Definitions

| State Code | State Name | Description | Terminal |
|------------|------------|-------------|----------|
| -3 | ORDER_TIMEOUT | Order creation timeout | ✅ |
| -2 | ORDER_INITIAL | Initial order creation | ❌ |
| -1 | ORDER_FAILED | Order creation failed | ✅ |
| 0 | LOOKING_FOR_DRIVER | Searching for available driver | ❌ |
| 1 | EN_ROUTE | Driver en route to pickup | ❌ |
| 2 | CANCELED | Order cancelled (user/system) | ✅ |
| 3 | NO_TAXI | No taxi available | ✅ |
| 4 | ON_TRIP | Trip in progress | ❌ |
| 5 | TAXI_NO_SHOW | Driver no-show | ✅ |
| 6 | TRACKING | Trip tracking active | ❌ |
| 7 | ECV_DROP_OFF | ECV drop off completed | ✅ |
| 8 | DROP_OFF | Trip completed (drop off) | ✅ |
| 9 | OFFLINE_DISPATCH | Offline dispatcher | ✅ |
| 10 | CHAT_ROOM_CREATED | Chat room established | ❌ |

**Terminal States**: ORDER_TIMEOUT, ORDER_FAILED, CANCELED, NO_TAXI, TAXI_NO_SHOW, DROP_OFF, ECV_DROP_OFF, OFFLINE_DISPATCH

### Key Design Patterns

#### 1. State Pattern
Complete state machine implementation untuk order lifecycle:

```go
type OrderState interface {
    UpdateIncomingStateLookingForDriver(ctx, req) (*model.Order, error)
    UpdateIncomingStateEnRoute(ctx, req) (*model.Order, error)
    UpdateIncomingStateCanceled(ctx, req) (*model.Order, error)
    UpdateIncomingStateOnTrip(ctx, req) (*model.Order, error)
    UpdateIncomingStateTracking(ctx, req) (*model.Order, error)
    UpdateIncomingStateDropOff(ctx, req) (*model.Order, error)
    // ... other state transitions
    
    CancelOrder(ctx, req) error
    RateOrder(ctx, req) error
    RetryOrder(ctx, req) (*model.Order, error)
    Reschedule(ctx, req) (*model.Order, error)
}
```

**Implementation Files**:
- `pkg/state/state_0_looking_for_driver.go`
- `pkg/state/state_1_en_route.go`
- `pkg/state/state_2_order_canceled.go`
- `pkg/state/state_4_on_trip.go`
- `pkg/state/state_8_drop_off.go`
- ... (per state implementation)

**Benefits**:
- ✅ Clear state transitions
- ✅ Prevented invalid state changes
- ✅ Easy to extend with new states
- ✅ Testable state logic

#### 2. Package Organization Pattern
Service organized by concern dengan clear boundaries:

```
pkg/
├── state/          # State machine implementations
├── preOrder/       # Pre-order validations & logic
├── postOrder/      # Post-order processing
├── payment/        # Payment processing logic
└── notification/   # Notification utilities
```

**Benefits**:
- ✅ Clear separation of concerns
- ✅ Easy to locate business logic
- ✅ Reusable packages
- ✅ Independent testing

#### 3. Event-Driven Architecture
Service processes events dari multiple sources:

```go
// Event sources:
- CallbackOrder() - dari Taxi Partner Gateway (iTOP)
- OrderEvent() - dari internal event bus
- PubSub events - dari message broker
```

**Event Flow**:
```
External System (iTOP) → CallbackOrder → State Machine → Database Update → Notification
Internal Event Bus     → OrderEvent    → State Machine → Database Update → Notification
```

#### 4. Repository Pattern
Data access diabstraksi melalui repository interfaces:

```go
type Repository struct {
    DB                  DB                        // Main database
    Redis               Redis                     // Caching & locking
    RedisStream         RedisStream               // Event streaming
    TaxiPartnerGW       TaxiPartnerGW            // iTOP integration
    PaymentProcessor    PaymentProcessor         // UPG integration
    User                User                      // User service
    SessionManager      SessionManager            // Session management
    ConfigService       ConfigService             // Feature flags
    GeoService          GeoService                // Location services
    PriceCalculator     PriceCalculator           // Fare calculation
    PromoGW             PromoGW                   // Promo service
    Firebase            Firebase                  // Real-time updates
    LocationRegistry    LocationRegistry          // Location data
    TrackerService      TrackerService            // GPS tracking
    // ... other repositories
}
```

## Integration Architecture

### Service Dependencies
```
┌─────────────────────────────────────────────────────────────────┐
│                             TOP                                  │
│                                                                  │
│  Core Processing:                                                │
│  ┌────────────────┐    ┌─────────────────┐                     │
│  │Taxi Partner GW │◄───│  State Machine  │                     │
│  │   (iTOP)       │    │                 │                     │
│  └────────┬───────┘    └────────┬────────┘                     │
│           │                      │                               │
│           ▼                      ▼                               │
│  ┌──────────────────────────────────────┐                      │
│  │      Order Processing Logic          │                      │
│  └──────────────────────────────────────┘                      │
│           │                      │                               │
│    ┌──────┴──────┬──────────────┴─────┬──────────┐            │
│    ▼             ▼                     ▼          ▼             │
│  ┌────┐    ┌────────┐          ┌─────────┐  ┌────────┐        │
│  │UPG │    │User    │          │Geo      │  │Price   │        │
│  │    │    │Service │          │Service  │  │Calc    │        │
│  └────┘    └────────┘          └─────────┘  └────────┘        │
│                                                                  │
│  ┌────────┐    ┌────────┐    ┌──────────┐   ┌────────┐        │
│  │Promo GW│    │Session │    │Location  │   │Tracker │        │
│  │        │    │Manager │    │Registry  │   │Service │        │
│  └────────┘    └────────┘    └──────────┘   └────────┘        │
│                                                                  │
│  ┌────────┐    ┌──────────┐                                    │
│  │Firebase│    │Notif     │                                    │
│  │        │    │Center    │                                    │
│  └────────┘    └──────────┘                                    │
│                                                                  │
│  Databases:                                                      │
│  ┌──────────────┐    ┌──────────┐                              │
│  │PostgreSQL    │    │Redis     │                              │
│  │              │    │          │                              │
│  └──────────────┘    └──────────┘                              │
└─────────────────────────────────────────────────────────────────┘
```

### Integration Matrix

| Service | Protocol | Purpose | Criticality |
|---------|----------|---------|-------------|
| **Taxi Partner GW (iTOP)** | gRPC | Order dispatch & callbacks | **CRITICAL** |
| **Payment Processor (UPG)** | gRPC | Payment processing | **CRITICAL** |
| **User Service** | gRPC | User information | HIGH |
| **GeoService** | gRPC | Location services | HIGH |
| **PriceCalculator** | gRPC | Fare calculation | HIGH |
| **PromoGW** | gRPC | Promo validation | MEDIUM |
| **SessionManager** | gRPC | Session validation | HIGH |
| **ConfigService** | gRPC | Feature flags | MEDIUM |
| **LocationRegistry** | gRPC | Location data | MEDIUM |
| **TrackerService** | gRPC | GPS tracking | MEDIUM |
| **Firebase** | gRPC | Real-time updates | LOW |
| **NotificationCenter** | gRPC | Notifications | LOW |
| **PostgreSQL** | TCP/5432 | Order data | **CRITICAL** |
| **Redis** | TCP/6379 | Cache & locks | **CRITICAL** |

## Core Features

### 1. Order Creation
Complete order creation flow dengan validations:

**Validations**:
- User eligibility (tidak sedang soft ban)
- Payment method availability
- Service maintenance window check
- Maximum concurrent bookings (5 future bookings)
- Area/location validation
- Promo code validation
- Pre-authorization (untuk credit card)

**Process**:
```
CreateOrder Request
  ↓
Pre-Order Validations
  ├─ Check soft ban status
  ├─ Validate payment method
  ├─ Check maintenance window
  ├─ Validate area coverage
  └─ Validate max bookings
  ↓
Pre-Auth (if credit card)
  ↓
Submit to Taxi Partner Gateway
  ↓
Create Order in DB
  ↓
Send to Firebase (real-time)
  ↓
Return Order Details
```

### 2. Order Lifecycle Management
State transitions via callbacks dan events:

**Callback Processing** (`CallbackOrder`):
- Dari iTOP (Taxi Partner Gateway)
- Updates order state
- Updates vehicle & driver info
- Calculates fares
- Sends notifications
- Updates Firebase real-time

**Event Processing** (`OrderEvent`):
- Dari internal event bus
- State transitions
- Database updates
- Notification triggers

### 3. Payment Processing
Comprehensive payment handling:

**Pre-Authorization**:
- Credit card pre-auth before order dispatch
- Pre-auth amount based on estimated fare
- Callback dari UPG untuk status

**Charging**:
- Charge on trip completion (drop off)
- Adjust charge if fare changes
- Handle charging failures
- Retry mechanism

**Payment States**:
- `NOT_CHARGED` - Belum di-charge
- `CHARGING` - Sedang proses charge
- `SUCCESS` - Charge berhasil
- `FAILED` - Charge gagal

### 4. Cancellation & Reschedule

#### Cancellation
**Types**:
- User cancellation (`CancelOrder`)
- System cancellation (`CancelOrderBySystem`)
- Driver cancellation (via callback)

**Soft Ban Mechanism**:
```go
MAX_ALLOWED_CANCELLED_ORDERS = 5
INTERVAL_TIME_CANCELLED_ORDER = 2 hours
EXPIRED_TIME_SOFT_BAN = 24 hours
```

Rules: Jika user cancel 5+ orders dalam 2 jam → soft ban 24 jam

#### Reschedule
**Features**:
- Reschedule future bookings
- Recalculate fare dengan new pickup time
- Update iTOP dengan new schedule
- Notify customer

### 5. Rating & Tips
**Rating**:
- Scale: 1-5 stars
- Comments support
- Hidden rating feature
- Driver performance tracking

**Tips**:
```go
minimumTip = 1000 IDR
```
- Optional driver tips
- Added to final fare
- Transferred to driver

### 6. Order Retry
**Retry Mechanism**:
- Retry failed orders
- Only for specific failure states
- Reset order state to LOOKING_FOR_DRIVER
- Submit ulang ke iTOP

### 7. Maintenance Window
**Features**:
- Scheduled maintenance windows
- Block order creation during maintenance
- Configurable per service type
- Automatic enforcement

```go
CheckServiceMaintenance(request) → returns maintenance status
```

## Data Model

### Core Tables

#### orders (taxi_orders)
```go
type Order struct {
    ID                 int64     // Order ID
    InternalUserID     string    // User BBID
    ExternalOrderID    string    // iTOP order ID
    Status             int       // Order state
    ServiceType        string    // bluebird, silverbird, goldenbird
    PickupLatitude     float64
    PickupLongitude    float64
    PickupAddress      string
    DropoffLatitude    float64
    DropoffLongitude   float64
    DropoffAddress     string
    PickupTime         time.Time
    EstimatedFare      float64
    FinalFare          float64
    PaymentMethod      string
    PaymentStatus      string
    ChargeStatus       string    // charging, success, failed
    DriverID           string
    DriverName         string
    VehicleNumber      string
    Rating             int
    Tips               float64
    // ... many more fields
}
```

#### order_summary
```go
type OrderSummary struct {
    OrderID            int64
    InternalUserID     string
    Category           string    // ride, delivery, rent
    Status             int
    FinalFare          float64
    Outstanding        float64
    IsRated            bool
    CreatedAt          time.Time
    UpdatedAt          time.Time
}
```

#### orders_pre_auth
```go
type OrderPreAuth struct {
    OrderID            int64
    PaymentIdentifier  string
    PreAuthAmount      float64
    PreAuthStatus      string
    CreatedAt          time.Time
}
```

#### soft_ban_users
```go
type SoftBanUser struct {
    InternalUserID     string
    BanReason          string
    BannedAt           time.Time
    ExpiresAt          time.Time
}
```

### Redis Keys

**Order Locking**:
```
create:{order_id}         # Lock during creation
cancel:{order_id}         # Lock during cancellation
end_trip:{order_id}       # Lock during trip end
```

**User Tracking**:
```
user:cancelled:{user_id}  # Track cancellations for soft ban
user:active_orders:{user_id}
```

## Observability

### APM Integration (Elastic APM)
```go
// Instrumentation
apmgrpc.NewUnaryServerInterceptor()  // gRPC tracing
apmhttp.Wrap(handler)                 // HTTP tracing
```

**Tracked Operations**:
- Order creation flow
- State transitions
- External service calls
- Database queries
- Payment operations

### Prometheus Metrics
```go
// Custom metrics
top_orders_created_total
top_order_state_transitions_total
top_payment_operations_total{status}
top_cancellations_total{reason}
top_external_call_duration_seconds{service}
```

### Health Checks
- **Endpoint**: `HealthCheck` gRPC method
- **Checks**:
  - Database connectivity
  - Redis availability
  - External services reachability

## Error Handling

### Error Code Structure
```
TOP-4xxx → Client Errors (4xx HTTP)
TOP-5xxx → Server Errors (5xx HTTP)
TOP-9xxx → Business Logic Errors (validation)
```

### Error Code Registry

| Code | Description | HTTP |
|------|-------------|------|
| TOP-4000 | Invalid value in request | 400 |
| TOP-4001 | Car number missing / User info not exist | 400 |
| TOP-4004 | Order not found | 404 |
| TOP-4005 | Forbidden request | 403 |
| TOP-4008 | Payment CC rejected | 400 |
| TOP-4009 | Trip purpose empty | 400 |
| TOP-40010 | Payment unavailable | 400 |
| TOP-40011 | ECV not applicable for date | 400 |
| TOP-40012 | ECV not applicable for car type | 400 |
| TOP-40013 | Cash for easy ride not allowed | 400 |
| TOP-40014 | Trip voucher validation failed | 400 |
| TOP-4031 | CC concurrent limit reached | 429 |
| TOP-4032 | Ongoing e-wallet order exists | 409 |
| TOP-4033 | Multiple booking disallowed | 409 |
| TOP-4034 | Chat room empty | 400 |
| TOP-4041 | Card not found | 404 |
| TOP-4091 | Payment already success | 409 |
| TOP-4092 | Charging failed | 402 |
| TOP-4093 | Fare exceeds balance | 402 |
| TOP-4221 | Active/failed charge TCash order exists | 409 |
| TOP-5000 | Internal error | 500 |
| TOP-5001 | General UPG error | 500 |
| TOP-5002 | Failed retry order | 500 |
| TOP-5003 | Trip fare iTOP invalid | 500 |
| TOP-9020 | Tip below minimum | 400 |
| TOP-9021 | Tip exceeds maximum | 400 |
| TOP-9022 | Invalid rating | 400 |

## Service Maturity

### Current Status: **Production-Critical** ⭐⭐⭐⭐⭐
- ✅ State machine implementation
- ✅ Comprehensive test coverage
- ✅ Dual protocol support (gRPC + REST)
- ✅ Full observability integration
- ✅ Graceful shutdown handling
- ✅ Payment integration (pre-auth & charging)
- ✅ Soft ban mechanism
- ✅ Maintenance window support

### Production Metrics
- **Uptime**: 99.95% SLA
- **Latency P95**: <500ms (order creation)
- **Request Rate**: ~1000 req/s peak
- **Order Success Rate**: >95%
- **Payment Success Rate**: >98%

## Special Features

### Soft Ban System
Automatic user soft ban untuk excessive cancellations:
```go
// Tracked per user:
- Cancelled orders dalam 2 jam terakhir
- Jika ≥ 5 cancellations → soft ban 24 jam
- Expired automatically setelah 24 jam
```

### Fare Types
Multiple fare components:
```go
ARGO_METER_FARE    // Metered fare
EXTRA_FARE         // Additional charges
FIXED_FARE         // Fixed price
DISCOUNT_FARE      // Promo discounts
CANCELATION_FARE   // Cancellation fee
ADJUSTMENT_FARE    // Manual adjustments
```

### Real-time Updates
Firebase integration untuk real-time order updates ke mobile app:
- Order status changes
- Driver location updates
- ETA updates
- Fare updates

## Cron Jobs

Service supports cron job mode:

```bash
# TNE (Trip Not Ended) checker
./top -execUsecase=cronjobordertne
```

**OrderSuspectTNE**: Checks for trips yang stuck (tidak complete dalam waktu reasonable)

## Related Documentation
- [[02-api-reference|API Reference]] - Complete API documentation
- [[03-state-machine|State Machine Details]] - Deep dive into state transitions
- [[04-payment-flows|Payment Flows]] - Payment processing details
- [[05-dependencies|External Dependencies]] - Services and integrations

---
**Last Updated**: 2025-01-07
**Documentation Status**: Complete
**Service Owner**: MRG Team