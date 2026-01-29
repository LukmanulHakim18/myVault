---
tags:
  - mrg
  - service
  - api-gateway
  - legacy
  - beego
  - grpc
  - documentation
team: MRG
type: service-documentation
title: MyBB API Gateway
status: legacy
created: '2026-01-29'
updated: '2026-01-29'
rest_port: 3000
grpc_port: 3443
repository: git.bluebird.id/mybb-legacy/mybb-app/src/mybb-api-gateway-go
tech_stack:
  - go
  - beego
  - grpc
  - postgresql
  - redis
  - kafka
  - firebase
---
# MyBB API Gateway

**Team**: MRG (Meta Reservation Gateway)  
**Status**: ⚠️ Legacy  
**Repository**: `git.bluebird.id/mybb-legacy/mybb-app/src/mybb-api-gateway-go`

---

## 📋 Overview

MyBB API Gateway adalah **legacy monolithic service** yang berfungsi sebagai entry point utama untuk semua request dari aplikasi mobile MyBluebird (Android/iOS). Service ini menangani routing, authentication, dan berbagai business logic yang seharusnya dipecah ke microservices.

### Fungsi Utama

- **API Gateway** - Entry point untuk semua REST API dari mobile apps
- **Authentication** - Token-based authentication (legacy, non-JWT)
- **Request Routing** - Route request ke berbagai backend services
- **User Management** - CRUD user, device management, profile updates
- **Order Proxy** - Proxy untuk order creation dan management
- **Geolocation** - Geocoding, autocomplete, directions (via Google Maps)
- **Payment Proxy** - Proxy untuk payment methods dan wallets
- **gRPC Server** - Location streaming, event changes, school bus tracking

### ⚠️ Legacy Notice

Service ini dalam status **legacy** dan sedang dalam proses migration ke microservices architecture. Beberapa fungsi sudah dimigrasikan ke:
- [[../authservice/README|Auth Service]] - Authentication & authorization
- [[../orderorchestrator/README|Order Orchestrator]] - Order management
- User Service - User CRUD operations
- Payment Processor - Payment handling

---

## 🛠️ Tech Stack

| Component | Technology |
|-----------|------------|
| Language | Go 1.23 |
| REST Framework | Beego v1.12.3 |
| Protocol | REST + gRPC (TLS) |
| Database | PostgreSQL |
| Cache | Redis (multiple instances) |
| Message Queue | Kafka |
| Realtime DB | Firebase |
| Cloud Storage | Google Cloud Storage |
| Monitoring | Elastic APM |
| Feature Flags | Unleash |
| Container | Docker, Kubernetes |

---

## 🔑 Konsep Utama

### 1. Legacy Token Authentication

Berbeda dengan Auth Service yang menggunakan JWT, API Gateway menggunakan sistem token legacy:

```go
// Token disimpan di Redis dengan format device object
type Device struct {
    Token           string
    Uuid            string
    User            *User
    AppVersion      string
    OperatingSystem string
}
```

- Token di-generate saat login dan disimpan di database `devices`
- Token di-cache ke Redis dengan TTL 30 hari
- Validasi dilakukan dengan lookup ke Redis/Database

### 2. Multi-Version API

Service ini memiliki **6 versi API** yang aktif:
- `/api/v1/*` - Oldest, deprecated
- `/api/v2/*` - User verification
- `/api/v3/*` - Third party auth, orders
- `/api/v4/*` - Golden Bird, tools, sessions
- `/api/v5/*` - Geolocation, landmarks, credit cards
- `/api/v6/*` - Latest, most features

### 3. Kafka Dispatcher

Request tertentu dikirim ke Kafka untuk async processing:
```
POST /api/v1/dispatcher/:topic
```

### 4. gRPC Services

Service ini juga menyediakan gRPC endpoints untuk:
- Location streaming (real-time car tracking)
- ODRD Location (On-Demand Ride Dispatch)
- Order events
- School bus tracking
- Failover OTP info

---

## 🔌 Dependencies

### Internal Services

| Service | Purpose | Protocol |
|---------|---------|----------|
| **Location Registry** | City/area lookup | HTTP |
| **Fare Service** | Fare calculation | HTTP |
| **Maps Service** | GMO directions | HTTP |
| **Dispatcher Service** | ETA calculation | HTTP |
| **Marketing Gateway** | Promo, loyalty, referral | HTTP |
| **Golden Bird** | GB order management | HTTP |
| **Corporate Portal** | Corporate voucher | HTTP |
| **Big Bird** | Big Bird reservations | HTTP |
| **School Bus** | School bus service | HTTP |
| **Promo Portal** | Promotion validation | HTTP |
| **ODRD** | On-Demand Ride Dispatch | HTTP |

### Infrastructure

| Component | Purpose |
|-----------|---------|
| **PostgreSQL** | User data, devices, favorite addresses |
| **Redis** | Token cache, session, rate limiting |
| **Redis (FDS)** | Fraud Detection System |
| **Kafka** | Message dispatching |
| **Firebase** | Realtime database, push notifications |
| **Google Cloud Storage** | Profile images |
| **Google Maps API** | Geocoding, directions, places |

Lihat detail lengkap di: [[dependencies|Dependencies]]

---

## 📡 API Overview

### REST Endpoints

| Namespace | Version | Purpose |
|-----------|---------|---------|
| `/api/auth/*` | - | Authentication (validate, register, encrypt) |
| `/api/v1/*` | v1 | Legacy: users, sessions, credit cards |
| `/api/v4/me/*` | v4 | User profile, orders, wallets |
| `/api/v5/me/*` | v5 | Places, orders, payment methods |
| `/api/v6/me/*` | v6 | Full feature: orders, wallets, loyalty |
| `/api/v6/geolocation/*` | v6 | Geocoding, autocomplete, directions |
| `/api/order/*` | - | Order tracking, billing details |
| `/api/subscriptions/*` | - | Marketing subscriptions |
| `/services/*` | - | Internal tools |
| `/webhook/*` | - | Payment webhooks |
| `/soap/itop/*` | - | Legacy SOAP integration |

### gRPC Services

| Service | Port | Purpose |
|---------|------|---------|
| LocationService | 3443 | Real-time location streaming |
| OdrdLocationService | 3443 | ODRD location updates |
| OrderService | 3443 | Order events |
| SchoolBusService | 3443 | School bus tracking |
| EventService | 3443 | Event changes (feature flagged) |

Lihat detail lengkap di: [[api-reference|API Reference]]

---

## ⚙️ Configuration

### Server Ports

| Protocol | Port | Environment Variable |
|----------|------|---------------------|
| REST | 3000 | - (hardcoded) |
| gRPC | 3443 | - (hardcoded) |
| pprof | 6060 | - (debug) |

### Key Environment Variables

```env
# Database
DB_HOST=database
DB_PORT=5432
DB_USERNAME=postgres
DB_PASSWORD=
DB_NAME=mybb_gateway_db

# Redis
REDIS_SERVER_IP=redis:6379
REDIS_SERVER_PASSWORD=
REDIS_SELECT_DB=2

# Google APIs
GOOGLE_API_KEY=
GOOGLE_PLACE_API_KEY=
GOOGLE_STATICMAP_SECRET=

# Service URLs
LOCATION_REGISTRY_URL=http://location_registry:3002
FARE_SERVICE_URL=http://fare_service:4000
MAPS_SERVICE_URL=https://mybb-mapsvc.bluebird.id
DISPATCHER_SERVICE_URL=https://apis.bluebird.id/v1/mock-mybb

# Authentication
JWT_SIGNATURE=
JWT_KEY_PATH=/var/cert/jwt_public_key.pem
OTT_TOLERANCE=30s
TOKEN_DURATION=720h

# TLS Certificates
CERT_FILE_PATH=/var/cert/localhost-cert.pem
KEY_FILE_PATH=/var/cert/localhost-key.pem

# APM
ELASTIC_APM_SERVER_URL=
ELASTIC_APM_SERVICE_NAME=mybb_app_api_gateway
ELASTIC_APM_ENVIRONMENT=staging
```

Lihat `.env.sample` untuk konfigurasi lengkap.

---

## 📂 Project Structure

```
mybb-api-gateway-go/
├── main.go                    # Entry point
├── api.go                     # Empty (legacy)
├── go.mod                     # Dependencies
├── Dockerfile                 # Container build
├── .gitlab-ci.yml             # CI/CD pipeline
│
├── appconfig/                 # Application config
│   └── app_config.go
│
├── cmd/                       # gRPC server setup
│   ├── grpc.go                # gRPC server initialization
│   └── grpc/                  # gRPC service implementations
│       ├── auth.go
│       ├── event.go
│       ├── location.go
│       ├── locationOdrd.go
│       ├── order.go
│       └── school_bus.go
│
├── conf/                      # Configuration files
│   ├── app.conf               # Beego config
│   └── serviceAccountKey.json # Firebase credentials
│
├── constants/                 # Constants
├── consumers/                 # Kafka consumers
│
├── controllers/               # HTTP handlers
│   ├── api/                   # REST API controllers
│   │   ├── base.go            # Base controller (1100+ lines!)
│   │   ├── base_ms.go         # Microservice base
│   │   ├── auth/              # Auth endpoints
│   │   ├── v1/                # API v1
│   │   ├── v2/                # API v2
│   │   ├── v3/                # API v3
│   │   ├── v4/                # API v4
│   │   ├── v5/                # API v5
│   │   ├── v6/                # API v6 (latest)
│   │   └── webhook/           # Webhook handlers
│   ├── soap/                  # SOAP controllers
│   └── web/                   # Web page controllers
│
├── database/                  # Database layer
│   ├── provider.go            # DB provider
│   ├── redis.go               # Redis setup
│   ├── redis/                 # Redis operations
│   ├── redisv8/               # Redis v8 client
│   └── migrations/            # DB migrations (50+ files)
│
├── dispatcher/                # Kafka dispatcher
├── email/                     # Email templates
├── endpoints/                 # gRPC endpoints
│
├── models/                    # Data models
│   ├── user.go
│   ├── device.go
│   ├── favorite_address.go
│   └── ... (40+ models)
│
├── repository/                # Data access layer
│   ├── user.go
│   ├── device.go
│   └── ... (30+ repositories)
│
├── routers/                   # URL routing
│   └── router.go              # Main router (1000+ lines!)
│
├── services/                  # Business logic
│   ├── user.go
│   ├── jwt.go
│   └── ... (40+ services)
│
├── middleware/                # HTTP middleware
│   ├── rate_limit.go
│   ├── app_version.go
│   └── abtest.go
│
├── parser/                    # Request/Response parsers
│   ├── request/
│   └── response/
│
├── lib/                       # External libraries
│   ├── firebase/
│   ├── google/
│   └── googlemap/
│
├── templates/                 # HTML/Email templates
├── assets/                    # Static assets
├── docs/                      # Swagger docs
└── k8s/                       # Kubernetes manifests
```

---

## 🚨 Technical Debt

### Critical Issues

| Issue | Description | Impact |
|-------|-------------|--------|
| **Monolithic Codebase** | Single service dengan 1000+ line files | Hard to maintain |
| **Multiple API Versions** | v1-v6 dengan duplicated logic | Code duplication |
| **Mixed Responsibilities** | Gateway + business logic | Tight coupling |
| **Legacy Auth** | Token-based, not JWT | Security concerns |
| **Direct DB Access** | Controllers access DB directly | No abstraction |
| **Giant Router** | `router.go` 1000+ lines | Hard to navigate |
| **Giant Base Controller** | `base.go` 1100+ lines | God object anti-pattern |

### Migration Status

| Component | Status | Target Service |
|-----------|--------|----------------|
| Authentication | 🔄 In Progress | Auth Service |
| User CRUD | 🔄 In Progress | User Service |
| Order Management | ✅ Migrated | Order Orchestrator |
| Payment | ✅ Migrated | Payment Processor |
| Token Management | 🔄 In Progress | Auth Service |

### Recommendations

1. **Deprecate v1-v4 APIs** - Focus on v5/v6 only
2. **Extract business logic** - Move to dedicated microservices
3. **Migrate to JWT** - Use Auth Service for authentication
4. **Split router** - Create separate router files per domain
5. **Remove direct DB access** - Use service clients
6. **Add API versioning strategy** - Proper deprecation policy

---

## 🔗 Related Documentation

- [[dependencies|Dependencies]]
- [[api-reference|API Reference]]
- [[legacy-flows|Legacy Flows & Tech Debt]]
- [[../authservice/README|Auth Service (replacement)]]
- [[../orderorchestrator/README|Order Orchestrator]]

---

## 🏷️ Tags

#mrg #service #api-gateway #legacy #beego #grpc #documentation

---

*Last Updated*: 2026-01-29  
*Assessment by*: Architecture Team
