---
title: Service Info
type: service-documentation
team: MRG
status: production
repository: git.bluebird.id/mybb-ms/serviceinfo
tech_stack:
  - go
  - grpc
  - postgresql
  - redis
tags:
  - mrg
  - service
  - serviceinfo
  - fleet
  - rental
  - grpc
  - documentation
created: '2025-01-05'
updated: '2025-01-05'
---
# Service Info

**Team**: MRG (Meta Reservation Gateway)  
**Status**: ✅ Production  
**Repository**: `git.bluebird.id/mybb-ms/serviceinfo`

---

## 📋 Overview

Service Info adalah service mapper dan management asset untuk service di MRG ke partner. Service ini bertanggung jawab untuk:

- **Fleet List Management** - Menyediakan daftar fleet/armada untuk aplikasi MyBluebird
- **Service Type Mapping** - Mapping service type antara MyBB dan BBD
- **Rental Management** - Mengelola fitur rental (hourly/daily)
- **Cititrans Integration** - Integrasi dengan layanan shuttle Cititrans
- **Price Calculation** - Kalkulasi harga dan fare estimation

---

## 🛠️ Tech Stack

| Component | Technology |
|-----------|------------|
| Language | Go 1.24 |
| Protocol | gRPC + REST (gRPC-Gateway) |
| Database | PostgreSQL |
| Cache | Redis |
| Message Queue | - |
| Monitoring | Elastic APM, Prometheus |
| Container | Docker, Kubernetes (Huawei Cloud) |

---

## 🔌 Dependencies

### Internal Services (Upstream)

| Service | Purpose | Client Library |
|---------|---------|----------------|
| **GB Gateway** | Goldenbird gateway untuk booking | `gbgatewayclient` v0.0.20 |
| **Taxi Partner Gateway** | Gateway ke partner taxi | `taxipartnergatewayclient` v1.0.20 |
| **Session Manager** | Manage user session | `sessionmanagerclient` v0.0.52 |
| **Promo Gateway** | Promo & discount management | `promogatewayclient` v1.1.16 |
| **Price Manager** | Price management | `pricemanagerclient` v0.0.7 |
| **UPG Gateway** | Payment gateway | `upggatewayclient` v0.0.23 |
| **Cititrans** | Shuttle service integration | `cititransclient` v0.0.32 |
| **COP** | Central Operation Platform | `copclient` v0.0.7 |
| **Content Provider** | Content/asset management | `contentproviderclient` v0.1.0 |
| **Config Service** | Configuration management | `configseviceclient` v0.1.3 |
| **User Service** | User management | `userclient` v0.0.23 |
| **Geo Service** | Geocoding & location | `geoclient` v0.0.8 |
| **Payment Processor** | Payment processing | `paymentprocessorclient` v1.1.35 |
| **Location Registry** | Location/area registry | (internal) |
| **ODRD** | On-Demand Ride Dispatch | (internal) |
| **ETA ODRD** | ETA calculation from ODRD | (internal) |
| **Price Engine** | Dynamic pricing engine | (internal) |
| **Area Management** | Area/zone management | (internal) |
| **BB Location** | Bluebird location service | (internal) |
| **Fare Service** | Fare calculation | (internal) |

### Infrastructure

| Component | Purpose |
|-----------|---------|
| **PostgreSQL** | Primary database (service_type_mapping, call_center, car_details, package_options) |
| **PostgreSQL Legacy** | Legacy database untuk package_options |
| **Redis** | Caching |
| **Redis Legacy** | Legacy cache |

---

## 📡 API Contracts

### gRPC Services

**Package**: `serviceinfo`  
**Proto File**: `contract/service-info.proto`

#### Health & System

| Method | Description |
|--------|-------------|
| `HealthCheck` | Health check endpoint |
| `PopulateCacheServiceTypeMapping` | Populate cache untuk service type mapping |
| `UpdateServiceTypeMapping` | Update service type mapping |
| `DeleteServiceTypeMapping` | Delete service type mapping |

#### Fleet & Vehicle

| Method | Description | Interceptors |
|--------|-------------|--------------|
| `DynamicFleetListV2` | Get dynamic fleet list untuk rental | InputValidator |
| `ChooseFleet` | Pilih fleet untuk booking | InputValidator |
| `ChooseUpsellFleet` | Pilih upsell fleet | InputValidator |
| `UpsellFleet` | Get upsell fleet options | InputValidator |
| `VehicleInfo` | Get vehicle info by car number | - |

#### Rental Flow

| Method | Description | Interceptors |
|--------|-------------|--------------|
| `InitialInfo` | Get initial info untuk rental | InputValidator |
| `RentEstimateEndTime` | Estimate end time untuk rental | InputValidator |
| `RecalculateRentPrice` | Recalculate rental price | InputValidator |
| `RentOrderConfirmation` | Confirmation page untuk rental order | InputValidator |
| `RentOrderPaymentSelection` | Payment selection untuk rental | InputValidator |
| `RentChoosePlace` | Choose pickup/dropoff place | InputValidator |
| `UpdateDuration` | Update rental duration | InputValidator |
| `RentValidateAreaSimulator` | Validate area untuk simulator | InputValidator |
| `CheckSessionRentRevamp` | Check session untuk rent revamp | InputValidator |
| `SimulatorArea` | Get simulator area info | InputValidator |

#### Validation

| Method | Description | Interceptors |
|--------|-------------|--------------|
| `ValidatePrebook` | Validate prebook request | InputValidator |
| `ValidateCreateOrder` | Validate create order request | InputValidator |
| `ValidateEZPay` | Validate EZ Pay request | MetadataValidator, InputValidator |

#### Trip & Booking

| Method | Description | Interceptors |
|--------|-------------|--------------|
| `NewTripIdOdrd` | Generate new trip ID dari ODRD | MetadataValidator |
| `CreateConsumerToken` | Create consumer token | InputValidator, MetadataValidator |
| `GetCallCenterList` | Get call center list by area | - |

#### Cititrans (Shuttle)

| Method | Description | Interceptors |
|--------|-------------|--------------|
| `GetScheduleListCititrans` | Get Cititrans schedule | MetadataValidator, InputValidator |
| `GetVesselDetail` | Get vessel/bus detail | MetadataValidator, InputValidator |
| `GetCititransSession` | Get Cititrans session | MetadataValidator, InputValidator |
| `SetCititransSession` | Set Cititrans session | MetadataValidator, InputValidator |

---

## 🗄️ Database Schema

### Tables

| Table | Description |
|-------|-------------|
| `service_type_mapping` | Mapping service type BBD ↔ MyBB |
| `call_center` | Call center contacts per area |
| `car_details` | Car/vehicle details |
| `package_options` | Package options untuk rental |

### DDL Location
- `sql/service_type_mapping.ddl.sql`
- `sql/data.sql`

---

## ⚙️ Configuration

### Environment Variables

#### Application
```env
APP_NAME="Mybb Service Info"
GRPC_PORT=6011
REST_PORT=8011
LOG_LEVEL=INFO
```

#### Database
```env
DB_HOST=
DB_PORT=
DB_NAME=
DB_USERNAME=
DB_PASSWORD=
DB_SSL_MODE=
MAX_OPEN_CONNS=
MAX_IDLE_CONNS=
CONN_MAX_IDLE_TIME=
CONN_MAX_LIFETIME=
```

#### Redis
```env
REDIS_HOST=
REDIS_PORT=
REDIS_DB=
REDIS_PASSWORD=
```

#### Service Connections
```env
# Location & Area
LOCATION_REGISTRY_HOST=
LOCATION_REGISTRY_PORT=
BB_LOCATION_HOST=
BB_LOCATION_PORT=
GEO_SERVICE_HOST=
GEO_SERVICE_PORT=
AREA_MANAGEMENT_HOST=
AREA_MANAGEMENT_PORT=

# Booking & Trip
GB_GATEWAY_HOST=
GB_GATEWAY_PORT=
TAXI_PARTNER_GATEWAY_HOST=
TAXI_PARTNER_GATEWAY_PORT=
SESSION_MANAGER_HOST=
SESSION_MANAGER_PORT=

# Pricing & Payment
PROMO_GATEWAY_HOST=
PROMO_GATEWAY_PORT=
PRICE_MANAGER_HOST=
PRICE_MANAGER_PORT=
PRICE_ENGINE_HOST=
PRICE_ENGINE_PORT=
UPG_GATEWAY_HOST=
UPG_GATEWAY_PORT=
PAYMENT_PROCESSOR_HOST=
PAYMENT_PROCESSOR_PORT=

# Partner Integration
CITITRANS_HOST=
CITITRANS_PORT=

# Content & Config
CONTENT_PROVIDER_HOST=
CONTENT_PROVIDER_PORT=
CONFIG_SERVICE_HOST=
CONFIG_SERVICE_PORT=
USER_SERVICE_HOST=
USER_SERVICE_PORT=

# Auth (BBD)
BBDAUTH_HOST=
BBDAUTH_PORT=
BBDAUTH_CLIENT_ID=
BBDAUTH_USER_ID_MYBB=
BBDAUTH_USER_SECRET_MYBB=
```

---

## 📂 Project Structure

```
serviceinfo/
├── main.go                    # Entry point
├── go.mod                     # Dependencies
├── Dockerfile                 # Container build
├── Makefile                   # Build commands
├── Jenkinsfile                # CI/CD pipeline
│
├── config/                    # Configuration
│   ├── default.go
│   ├── logger/
│   └── repository/
│
├── constant/                  # Constants & enums
│   ├── service_type.go
│   ├── car_type.go
│   ├── payment_type.go
│   └── ...
│
├── contract/                  # API contracts
│   ├── service-info.proto     # gRPC definition
│   ├── service-info.pb.go     # Generated code
│   ├── service-info_grpc.pb.go
│   ├── service-info.pb.gw.go  # REST gateway
│   └── service-info.swagger.json
│
├── model/                     # Domain models
│   ├── fleet_data.go
│   ├── fare_service.go
│   ├── service_type.go
│   └── ...
│
├── repository/                # Data access layer
│   ├── base_repository.go     # Repository aggregator
│   ├── repoiface/             # Repository interfaces
│   ├── repomock/              # Mock implementations
│   ├── db/                    # PostgreSQL repositories
│   ├── redis/                 # Redis repositories
│   └── {service}/             # External service clients
│
├── usecase/                   # Business logic
│   ├── base_usecase.go
│   ├── dynamic_fleet_list_v2.go
│   ├── choose_fleet.go
│   └── ...
│
├── transport/                 # Transport layer (handlers)
│   ├── base_transport.go
│   ├── dynamic_fleet_list_v2.go
│   └── ...
│
├── server/                    # Server setup
│   ├── grpc.go
│   ├── rest.go
│   └── metric.go
│
├── pkg/                       # Internal packages
│   ├── initialinfo/           # Initial info strategies
│   └── rent/                  # Rental utilities
│
├── util/                      # Utilities
│   ├── interceptor/           # gRPC interceptors
│   ├── errors/
│   └── ...
│
├── validator/                 # Input validators
│
├── sql/                       # Database migrations
│
└── k8s/                       # Kubernetes manifests
    └── huawei-application.yaml
```

---

## 🔗 Related Documentation

- [[02-Work/Teams/MRG/00-overview/README|MRG Team Overview]]
- [[02-Work/Architecture-Office/initiatives/gcp-to-hwc-migration/INDEX|GCP to HWC Migration]]

---

## 🏷️ Tags

#mrg #service #serviceinfo #fleet #rental #grpc #documentation

---

*Last Updated*: 2025-01-05  
*Generated from*: Repository analysis
