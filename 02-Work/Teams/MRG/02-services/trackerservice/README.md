---
tags:
  - mrg
  - service
  - trackerservice
  - tracking
  - location
  - firebase
  - grpc
  - documentation
team: MRG
type: service-documentation
title: Tracker Service
status: production
created: '2025-01-26'
updated: '2025-01-26'
grpc_port: 6013
rest_port: 8013
repository: git.bluebird.id/mybb-ms/trackerservice
tech_stack:
  - go
  - grpc
  - redis
  - firebase
  - pubsub
---
# Tracker Service

**Team**: MRG (Meta Reservation Gateway)  
**Status**: ✅ Production  
**Repository**: `git.bluebird.id/mybb-ms/trackerservice`

---

## 📋 Overview

MyBB TrackerService adalah microservice yang bertanggung jawab untuk mengelola real-time location tracking dalam ekosistem MyBluebird. Service ini menyediakan fitur untuk track driver location, find nearby available cars, dan share trip information dengan third parties.

### Fungsi Utama

- **Car Tracking** - Real-time tracking lokasi driver selama trip
- **Nearby Cars** - Mencari dan menampilkan mobil available di sekitar user
- **Share Trip** - Generate shareable URL untuk monitor trip secara real-time
- **Order State Listener** - Listen to order state changes untuk update tracking data
- **Firebase Integration** - Store real-time location data di Firebase Realtime Database

---

## 🛠️ Tech Stack

| Component | Technology |
|-----------|------------|
| Language | Go 1.22 |
| Protocol | gRPC + REST (gRPC-Gateway) |
| Real-time Database | Firebase Realtime Database |
| Cache | Redis |
| Message Queue | Google PubSub |
| Task Manager | Internal Background Worker |
| Monitoring | Elastic APM |
| Container | Docker, Kubernetes |

---

## 🔑 Konsep Utama

### 1. Real-Time Car Tracking
- Driver location di-update setiap interval (default: 20 detik)
- Data disimpan di Firebase untuk real-time sync ke client
- Redis sebagai caching layer untuk performance
- Tracking duration: 24 jam (configurable)

### 2. Nearby Cars Discovery
- Search available cars dalam radius tertentu
- Filter by service type (Regular, Premium, Executive)
- Integration dengan Goldenbird Gateway dan Taxi Partner
- Real-time availability check

### 3. Share Trip Functionality
- Generate unique URL untuk share trip
- External users bisa monitor trip tanpa login
- Background worker untuk parse dan serve trip data
- Expiration handling

### 4. Order State Integration
- Listen to order state changes via PubSub
- Update tracking status based on order state
- Trigger trip end process
- Payment status tracking

---

## 🔌 Dependencies

### Internal Services

| Service | Purpose | Protocol |
|---------|---------|----------|
| **Taxi Partner Gateway** | Get taxi availability, driver location | gRPC |
| **Payment Processor** | Payment status untuk trip end | gRPC |
| **Goldenbird Gateway** | Fleet management, car availability | gRPC |
| **Order Query** | Order details, trip information | gRPC |
| **Gorooster** | Event routing, webhook forwarding | HTTP |
| **Routine Manager** | Schedule periodic tasks | gRPC |

### External Services

| Service | Purpose |
|---------|---------|
| **Firebase Realtime DB** | Real-time location storage & sync |
| **Google PubSub** | Event messaging |

### Client Libraries

| Library | Version | Purpose |
|---------|---------|---------|
| `taxipartnergatewayclient` | v1.0.10 | Taxi partner integration |
| `paymentprocessorclient` | v1.1.15 | Payment processor integration |
| `gbgatewayclient` | v0.0.13 | Goldenbird gateway integration |
| `orderqueryclient` | v0.0.13 | Order query integration |
| `routinemanagerclient` | v0.0.2 | Routine manager integration |
| `gorooster-client` | v1.1.4 | Gorooster integration |
| `aphrodite` | v1.7.38 | Common utilities |
| `commonmessaging` | v0.1.15 | PubSub messaging |

### Infrastructure

| Component | Purpose |
|-----------|---------|
| **Firebase** | Real-time location database |
| **Redis** | Location cache, session data, nearby cars cache |
| **Google PubSub** | Order state changes events |

### Repository Structure

```go
type Repository struct {
    Redis            repoiface.RedisRepository
    Firebase         repoiface.Firebase
    Partner          repoiface.RepoPartnerGateway
    Gorooster        repoiface.Gorooster
    PaymentProcessor repoiface.PaymentProcessor
    GbGateway        repoiface.GbGateway
    OrderQuery       repoiface.OrderQuery
    RoutineManager   repoiface.RoutineManagerIface
}
```

---

## 📡 API Contracts

### gRPC Service

**Package**: `trackerservice`  
**Service Name**: `TrackerService`  
**Proto File**: `contract/tracker_service.proto`  
**Ports**: gRPC `6013`, REST `8013`

### Methods Overview

#### Car Tracking

| Method | Description | Interceptors |
|--------|-------------|--------------|
| `CarTracking` | Update car location during trip | - |
| `GetNearbyCars` | Get available cars nearby | InputValidator, MetadataValidator |

#### Trip Management

| Method | Description | Interceptors |
|--------|-------------|--------------|
| `ShareTrip` | Generate shareable trip URL | MetadataValidator |
| `OrderChanges` | Listen to order state changes | - |

#### Utility

| Method | Description | Interceptors |
|--------|-------------|--------------|
| `HealthCheck` | Health check | - |

---

## 🔄 Core Workflows

### Car Tracking Flow
```
Driver App sends location every 20s
    ↓
CarTracking(car_number, order_id, location)
    ↓
Validate order is active
    ↓
Store location in Firebase (path: /car_tracking/{order_id})
    ↓
Cache in Redis (TTL: 24h)
    ↓
Client apps receive real-time updates via Firebase listener
```

### Nearby Cars Flow
```
User opens app → GetNearbyCars(location, service_type)
    ↓
Check Redis cache (key: nearby_cars_{lat}_{lng}_{service_type})
    ↓ (cache miss)
Parallel queries:
  - Goldenbird Gateway: Get available GB cars
  - Taxi Partner Gateway: Get available partner cars
    ↓
Merge results
    ↓
Calculate distances from user location
    ↓
Filter by radius & service type
    ↓
Sort by distance (nearest first)
    ↓
Cache result (TTL: 30s) → Return to client
```

### Share Trip Flow
```
User taps Share Trip → ShareTrip(order_id, trip_id)
    ↓
Generate unique share URL
    ↓
Store share session in Redis
    ↓
Background Worker: Parse order details
    ↓
Generate public view page
    ↓
Return shareable URL to user
    ↓
External user accesses URL → See real-time tracking
```

Lihat detail lengkap di: [[workflows|Detailed Workflows]]

---

## ⚙️ Configuration

### Environment Variables

```env
# Application
APP_NAME=tracker-service
GRPC_PORT=6013
REST_PORT=8013
LOG_LEVEL=INFO

# Redis
REDIS_HOST=172.26.11.40
REDIS_PORT=6379
REDIS_PASSWORD=
REDIS_DB=0

# Firebase
FIREBASE_URL=https://mybluebird-production.firebaseio.com/
FIREBASE_AUTH=xxx
FIREBASE_AUTH_FILE=cert/firebase.json
FIREBASE_DB_URL=mybluebird-production.firebaseapp.com
FIREBASE_CREDENTIAL_FILE=cert/firebase-credentials.json

# Car Tracking Settings
CAR_LOCATION_DURATION=24h
CAR_LOCATION_FREQUENCY=20s
CAR_TRACKING_URL=http://gorooster-forwarder:8022/trackerservice/car-tracking

# Gorooster (Event Router)
GOROOSTER_URL=http://gorooster:1407
GOROOSTER_CLIENT_NAME=tracker-service
GOROOSTER_API_KEY=xxx

# Google PubSub
PUBSUB_CREDENTIAL=cert/pubsub-credentials.json
PUBSUB_PROJECT_ID=mybluebird

# Taxi Partner Gateway
TAXI_PARTNER_URL=taxi-partner-gateway
TAXI_PARTNER_PORT=6008

# Payment Processor
PAYMENT_PROCESSOR_HOST=payment-processor-service
PAYMENT_PROCESSOR_PORT=6010

# Goldenbird Gateway
GB_GATEWAY_HOST=gb-gateway-service
GB_GATEWAY_PORT=6011

# Order Query
ORDER_QUERY_HOST=order-query-service
ORDER_QUERY_PORT=6012
ORDER_QUERY_LEGACY_HOST=legacy-order-query
ORDER_QUERY_LEGACY_PORT=8080
ORDER_QUERY_LEGACY_TOKEN=xxx

# Routine Manager
ROUTINEMANAGER_HOST=routine-manager-service
ROUTINEMANAGER_PORT=6020

# Nearby Cars Settings
NEARBY_CAR_RADIUS_GB=200.0
NEARBY_CAR_LIMIT_GB=10
ODD_EVENT_TYPE=ALL

# Share Trip
ODRD_SHARE_TRIP_HOST=https://odrd.mybluebird.id
SHARE_TRIP_HOST=https://share.mybluebird.id

# Feature Flags
FULL_MICROSERVICE=true

# APM
ELASTIC_APM_SERVER_URL=
ELASTIC_APM_SERVICE_NAME=mybb-tracker-service
ELASTIC_APM_ENVIRONMENT=PRODUCTION
```

---

## 📂 Project Structure

```
trackerservice/
├── main.go                    # Entry point
├── go.mod                     # Dependencies
├── Dockerfile                 # Container build
├── Jenkinsfile                # CI/CD pipeline
│
├── config/                    # Configuration
│   ├── default.go
│   ├── logger/
│   └── repository/
│
├── contract/                  # API contracts
│   ├── tracker_service.proto
│   ├── tracker_service.pb.go
│   ├── tracker_service_grpc.pb.go
│   ├── tracker_service.pb.gw.go
│   ├── tracker_service.swagger.json
│   └── validator.go
│
├── model/                     # Domain models
│   ├── car_nearby.go
│   ├── car_tracking.go
│   ├── order_detail.go
│   └── white_list.go
│
├── repository/                # Data access layer
│   ├── base_repository.go
│   ├── repoiface/             # Interfaces
│   ├── repomock/              # Mocks
│   ├── reporedis/             # Redis implementation
│   ├── firebase/              # Firebase implementation
│   ├── gbgateway/             # GB Gateway client
│   ├── repotaxipartner/       # Taxi Partner client
│   ├── paymentprocessor/      # Payment Processor client
│   ├── orderquery/            # Order Query client
│   ├── orderquerylegacy/      # Legacy Order Query client
│   ├── repogorooster/         # Gorooster client
│   └── routinemanager/        # Routine Manager client
│
├── usecase/                   # Business logic (with tests)
│   ├── base_usecase.go
│   ├── car_tracking.go
│   ├── get_nearby.go
│   ├── share_trip.go
│   ├── order_changes.go
│   ├── end_trip_process.go
│   ├── access_control.go
│   └── health_check.go
│
├── transport/                 # Transport layer
│   ├── base_transport.go
│   ├── car_tracking.go
│   ├── get_nearby_cars.go
│   ├── share_trip.go
│   ├── order_changes.go
│   └── health_check.go
│
├── consumer/                  # Message queue consumers
│   ├── base_consumer.go
│   └── order_changes.go
│
├── task/                      # Background workers
│   ├── base_task.go
│   ├── share_trip_parser_worker.go
│   └── share_trip_parser_worker_test.go
│
├── server/                    # Server setup
│   ├── grpc.go
│   └── rest.go
│
├── util/                      # Utilities
│   ├── interceptor/
│   ├── errors/
│   ├── constants/
│   ├── lib/
│   ├── server.go
│   └── util.go
│
├── doc/                       # Documentation
│   ├── flow_sequence_diagram.plantuml
│   └── flow_sequence_diagram.png
│
└── k8s/                       # Kubernetes manifests
    ├── deployment.yaml
    ├── service.yaml
    ├── service-account.yaml
    ├── role.yaml
    └── role-binding.yaml
```

---

## 🎯 Key Features

### 1. Real-Time Location Tracking
- Driver location updates every 20 seconds
- Firebase Realtime Database untuk instant sync
- Redis caching untuk reduce Firebase load
- 24-hour tracking duration per trip

### 2. Nearby Cars Discovery
- Multi-source aggregation (GB + Partner cars)
- Distance-based filtering & sorting
- Service type filtering
- Cache optimization (30s TTL)

### 3. Share Trip
- Generate secure shareable URLs
- Public access tanpa authentication
- Background worker untuk data parsing
- Real-time tracking view untuk external users

### 4. Order State Integration
- PubSub event listener
- Auto-trigger trip end process
- Payment status synchronization
- State transition handling

### 5. Task Manager
- Background job processing
- Share trip URL parsing
- Periodic cleanup tasks
- Graceful shutdown support

---

## 🔒 Performance & Optimization

### Caching Strategy
- **Redis**: Car locations, nearby cars results, share trip data
- **Firebase**: Real-time location sync (read-optimized)
- **TTL**: 
  - Car location: 24h
  - Nearby cars: 30s
  - Share trip session: Until trip ends

### Rate Limiting
- Car tracking updates: Max 1 per 20s
- Nearby cars queries: Cached for 30s
- Firebase writes: Throttled per driver

### Firebase Optimization
- Write only on significant location changes (>10m)
- Use delta updates instead of full writes
- Index optimization for query performance

---

## 🔗 Related Documentation

- [[api-reference|API Reference]]
- [[workflows|Detailed Workflows]]
- [[dependencies|Dependencies]]
- [[02-Work/Teams/MRG/00-overview/README|MRG Team Overview]]

---

## 🏷️ Tags

#mrg #service #trackerservice #tracking #location #firebase #grpc #documentation

---

*Last Updated*: 2025-01-26  
*Generated from*: Repository analysis & code structure
