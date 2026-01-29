---
tags:
  - mrg
  - service
  - geoservice
  - geolocation
  - maps
  - reverse-geocode
  - grpc
  - documentation
team: MRG
type: service-documentation
title: Geo Service
status: production
created: '2025-01-26'
updated: '2025-01-26'
grpc_port: 6000
rest_port: 8000
repository: git.bluebird.id/mybb-ms/geoservice
tech_stack:
  - go
  - grpc
  - redis
  - postgresql
  - broker
---
# Geo Service

**Team**: MRG (Meta Reservation Gateway)  
**Status**: ✅ Production  
**Repository**: `git.bluebird.id/mybb-ms/geoservice`

---

## 📋 Overview

MyBB GeoService (GMO Service) adalah microservice yang bertanggung jawab untuk mengelola semua operasi geolokasi dalam ekosistem MyBluebird. Service ini menyediakan fitur geocoding, reverse geocoding, autocomplete lokasi, dan manajemen frequent locations untuk meningkatkan user experience dalam booking transportasi.

### Fungsi Utama

- **Reverse Geocoding** - Convert koordinat (lat/long) menjadi alamat lengkap dengan detail area
- **Autocomplete** - Suggestion lokasi berdasarkan keyword untuk pickup/dropoff
- **History Place** - Menampilkan lokasi yang sering dikunjungi user
- **Frequent Location Detection** - Auto-detect lokasi favorit berdasarkan trip history
- **Auto Snap** - Snap koordinat ke titik POI/landmark terdekat untuk akurasi
- **Area Management Integration** - Goldenbird area mapping untuk zona operasional
- **Order Event Listener** - Track trip locations untuk frequent location analysis

---

## 🛠️ Tech Stack

| Component | Technology |
|-----------|------------|
| Language | Go 1.24 |
| Protocol | gRPC + REST (gRPC-Gateway) + Message Broker |
| Database | PostgreSQL (Legacy DB) |
| Cache | Redis |
| Message Queue | Broker (AMQP/Kafka) |
| Map Provider | External Map Service (Google Maps/AMap) |
| Monitoring | Elastic APM |
| Container | Docker, Kubernetes |

---

## 🔑 Konsep Utama

### 1. Reverse Geocoding
- **V2**: Legacy version dengan subplace support
- **V3**: Latest version dengan timezone, goldenbird areas, dan category support

### 2. Location Types
- **Landmark**: POI dengan subplace (e.g., mall, airport)
- **Non-Landmark**: Alamat biasa tanpa subplace
- **Area**: Zona/area management untuk operational coverage

### 3. Favorite vs Frequent Location
- **Favorite**: User-saved locations dengan custom notes
- **Frequent**: Auto-detected berdasarkan trip patterns (min threshold trips dalam periode tertentu)

### 4. Auto Snap Feature
- Snap koordinat user ke landmark/POI terdekat
- Meningkatkan akurasi pickup/dropoff point
- Configurable timeout dan radius

### 5. Place Categories
- **pickup**: Location untuk penjemputan
- **dropoff**: Location untuk tujuan
- Support filtering berdasarkan kategori

---

## 🔌 Dependencies

### Internal Services

| Service | Purpose | Protocol |
|---------|---------|----------|
| **User Service** | Get user profile, favorite addresses | gRPC |
| **Area Management** | Goldenbird zone mapping, operational areas | gRPC |
| **Content Provider** | Popular places, POI data | gRPC |
| **Order Query** | Trip history untuk frequent location analysis | gRPC |

### External Services

| Service | Purpose | Protocol |
|---------|---------|----------|
| **Map Service** | Geocoding, reverse geocoding, autocomplete | HTTP REST |

### Client Libraries

| Library | Version | Purpose |
|---------|---------|---------|
| `userclient` | v0.0.23 | User service integration |
| `contentproviderclient` | v0.1.0 | Content provider integration |
| `aphrodite` | v1.9.47 | Common utilities & logger |
| `grpc-client` | v1.5.37 | gRPC client utilities |

### Infrastructure

| Component | Purpose |
|-----------|---------|
| **Redis** | Frequent location cache, session data |
| **PostgreSQL** | Legacy trip data storage |
| **Message Broker** | Order state changes event listener |

### Repository Structure

```go
type Repository struct {
    Redis           repoiface.IRedis
    MapService      repoiface.MapService
    UserService     repoiface.User
    LegacyDB        repoiface.LegacyDB
    AreaManagement  repoiface.AreaManagement
    ContentProvider repoiface.ContentProvider
    OrderQuery      repoiface.OrderQuery
}
```

---

## 📡 API Contracts

### gRPC Service

**Package**: `geo`  
**Service Name**: `GmoService`  
**Proto File**: `contract/geo.proto`  
**Ports**: gRPC `6000`, REST `8000`

### Methods Overview

#### Core Geolocation

| Method | Description | Interceptors |
|--------|-------------|--------------|
| `ReverseGeocodeV2` | Legacy reverse geocode dengan subplace | InputValidator, MetadataValidator |
| `ReverseGeocodeV3` | Latest reverse geocode dengan timezone & goldenbird areas | InputValidator |
| `AutoComplete` | Location suggestion berdasarkan keyword | InputValidator |
| `HistoryPlace` | Get user's frequently visited places | InputValidator |

#### Background Jobs

| Method | Description | Interceptors |
|--------|-------------|--------------|
| `OrderStateChangesListener` | Listen to order state changes untuk frequent location | - |
| `RemoveExpiredFrequentLocation` | Cleanup expired frequent locations | - |

#### Utility

| Method | Description | Interceptors |
|--------|-------------|--------------|
| `HealthCheck` | Health check | - |

---

## 🔄 Core Workflows

### Reverse Geocoding Flow (V3)
```
User taps map → ReverseGeocodeV3(lat, long, type, trip_id)
  ↓
Check Redis cache (by lat/long key)
  ↓ (cache miss)
Map Service: Get address from coordinates
  ↓
User Service: Get favorite addresses in radius
  ↓
Area Management: Get goldenbird areas
  ↓
Auto Snap (if enabled): Snap to nearest landmark
  ↓
Build response with:
  - Address details
  - Subplaces (if landmark)
  - Favorite address match
  - Goldenbird areas
  - Timezone
  ↓
Cache result in Redis → Return to client
```

### AutoComplete Flow
```
User types "Bandara" → AutoComplete(keyword, lat, long, trip_id)
  ↓
Parallel queries:
  1. Map Service: Search POI by keyword
  2. User Service: Match favorite addresses
  3. Content Provider: Get popular places (if enabled)
  ↓
Merge & deduplicate results
  ↓
Sort by:
  - Distance from current location
  - Popular place priority
  ↓
Enrich with:
  - Favorite address match
  - Area Management zones
  - Icons & categories
  ↓
Return sorted suggestions
```

### Frequent Location Detection Flow
```
Order completed → OrderStateChangesListener(order_data)
  ↓
Extract pickup & dropoff coordinates
  ↓
Store in PostgreSQL (legacy_trip_data)
  ↓
Background job (cron): RemoveExpiredFrequentLocation()
  ↓
Query trips in lookback period (e.g., 30 days)
  ↓
Group by location clusters (within radius)
  ↓
Filter: Count >= min_trip_threshold
  ↓
Generate frequent_location entries
  ↓
Cache in Redis for fast access
```

Lihat detail lengkap di: [[workflows|Detailed Workflows]]

---

## ⚙️ Configuration

### Environment Variables

```env
# Application
APP_NAME=geoservice
GRPC_PORT=6000
REST_PORT=8000
LOG_LEVEL=INFO

# Redis
REDIS_HOST=172.26.11.40
REDIS_PORT=6379
REDIS_PASSWORD=
REDIS_DATABASE=5

# Map Service (External)
MAP_SERVICE_HOST=https://maps.googleapis.com
MAP_SERVICE_TIMEOUT=5s
MAP_SERVICE_TOKEN=xxx

# User Service
USER_SERVICE_HOST=user-service
USER_SERVICE_PORT=6015

# Area Management
AREA_MANAGEMENT_HOST=area-management-service
AREA_MANAGEMENT_PORT=6020
AREA_MANAGEMENT_CHANNEL_ID=mybb-channel
AREA_MANAGEMENT_TIMEOUT=5s

# Content Provider
CONTENT_PROVIDER_HOST=content-provider-service
CONTENT_PROVIDER_PORT=6030

# Order Query
ORDERQUERY_HOST=orderquery-service
ORDERQUERY_PORT=6040

# Legacy Database
DB_LEGACY_HOST=172.26.11.50
DB_LEGACY_HOST_READ=172.26.11.51
DB_LEGACY_PORT=5432
DB_LEGACY_USERNAME=postgres
DB_LEGACY_PASSWORD=xxx
DB_LEGACY_NAME=legacy_mybb
DB_LEGACY_SSL_MODE=disable
DB_LEGACY_MAX_IDLE_CONNS=10
DB_LEGACY_MAX_OPEN_CONNS=100
DB_LEGACY_CONN_MAX_IDLE_TIME=10m
DB_LEGACY_CONN_MAX_LIFETIME=1h

# Radius Settings (meters)
RADIUS_NON_LANDMARK_FAVORITE_LOCATION=100
RADIUS_NON_LANDMARK_FREQUENT_LOCATION=100
RADIUS_LANDMARK_WITHOUT_SUBPLACE_FAVORITE_LOCATION=100
RADIUS_LANDMARK_WITHOUT_SUBPLACE_FREQUENT_LOCATION=50
RADIUS_LANDMARK_WITH_SUBPLACE_FAVORITE_LOCATION=100
RADIUS_LANDMARK_WITH_SUBPLACE_FREQUENT_LOCATION=50
RADIUS_EXCLUDE_FAVORITE_LOCATION_SETTER=100

# Frequent Location Settings
FREQUENT_LOCATION_MIN_TRIP_THRESHOLD=3
FREQUENT_LOCATION_LOOKBACK_PERIOD_DAYS=30d
FREQUENT_LOCATION_RADIUS_IN_METERS=100

# Auto Snap Feature
ENABLE_AUTO_SNAP_FEATURE=true
AUTO_SNAP_TIMEOUT=2s

# History Place
HISTORY_PLACE_LIMIT=2

# APM
ELASTIC_APM_SERVER_URL=
ELASTIC_APM_SERVICE_NAME=mybb-geo-service
ELASTIC_APM_ENVIRONMENT=PRODUCTION
```

---

## 📂 Project Structure

```
geoservice/
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
├── constant/                  # Constants
│   └── identifier.go
│
├── contract/                  # API contracts
│   ├── geo.proto
│   ├── geo.pb.go
│   ├── geo_grpc.pb.go
│   ├── geo.pb.gw.go
│   ├── geo.swagger.json
│   ├── geo.broker.yaml         # Message broker config
│   ├── geo.pb.broker.go
│   └── validator.go
│
├── model/                     # Domain models
│   ├── area_management_autocomplete.go
│   ├── area_management_filter.go
│   ├── area_management_reverse_geocode.go
│   ├── frequent_location.go
│   ├── map_service.go
│   ├── polygon_geometry.go
│   ├── popular_places.go
│   └── migrations/
│
├── domain/                    # Domain logic
│   ├── autosnap/
│   └── frequentlocation/
│       ├── getter/
│       └── setter/
│
├── repository/                # Data access layer
│   ├── base_repository.go
│   ├── repoiface/             # Interfaces
│   ├── redis/                 # Redis implementation
│   ├── mapservice/            # Map service client
│   ├── user/                  # User service client
│   ├── legacydb/              # PostgreSQL legacy DB
│   ├── areamanagement/        # Area management client
│   ├── content_provider/      # Content provider client
│   └── orderquery/            # Order query client
│
├── usecase/                   # Business logic (with tests)
│   ├── base_usecase.go
│   ├── reverse_geocode_v2.go
│   ├── reverse_geocode_v3.go
│   ├── auto_complete.go
│   ├── history_place.go
│   ├── order_state_changes_listener.go
│   ├── remove_expired_frequent_location.go
│   └── health_check.go
│
├── transport/                 # Transport layer
│   ├── base_transport.go
│   ├── reverse_geocode_v_2.go
│   ├── reverse_geocode_v_3.go
│   ├── auto_complete.go
│   ├── history_place.go
│   ├── order_state_changes_listener.go
│   ├── remove_expired_frequent_location.go
│   └── health_check.go
│
├── server/                    # Server setup
│   ├── grpc.go
│   ├── rest.go
│   ├── broker.go              # Message broker listener
│   └── metric.go
│
├── util/                      # Utilities
│   ├── interceptor/
│   ├── errors/
│   ├── constants/
│   ├── lib/
│   ├── servopt/
│   ├── parse_duration.go
│   ├── place_type_helper.go
│   ├── remove_plus_code.go
│   ├── response.go
│   ├── reverse_geocode_resp_helper.go
│   └── server.go
│
└── k8s/                       # Kubernetes manifests
    ├── deployment.yaml
    ├── service.yaml
    ├── gcp-application.yaml
    └── huawei-application.yaml
```

---

## 🎯 Key Features

### 1. Multi-Version Reverse Geocoding
- **V2**: Legacy support dengan backward compatibility
- **V3**: Enhanced dengan timezone, goldenbird areas, place category

### 2. Smart Location Suggestions
- Real-time autocomplete dari multiple sources
- Favorite & frequent location integration
- Distance-based sorting
- Popular place boosting

### 3. Frequent Location Intelligence
- Auto-detect favorite places dari trip patterns
- Configurable threshold dan lookback period
- Background job untuk cleanup expired entries

### 4. Auto Snap untuk Akurasi
- Snap ke nearest landmark/POI
- Reduce koordinat ambiguity
- Improve driver-customer meetup success rate

### 5. Area Management Integration
- Goldenbird zone mapping
- Operational area coverage check
- Zone-based routing support

---

## 🔒 Performance & Optimization

### Caching Strategy
- **Redis**: Reverse geocode results, frequent locations
- **TTL**: Configurable per data type
- **Cache key**: Coordinate-based dengan precision handling

### Rate Limiting
- Map service call optimization
- Batch processing untuk frequent location detection
- Timeout protection

### Database Optimization
- Read replica untuk legacy DB queries
- Connection pooling
- Index optimization pada trip location queries

---

## 🔗 Related Documentation

- [[api-reference|API Reference]]
- [[workflows|Detailed Workflows]]
- [[dependencies|Dependencies]]
- [[02-Work/Teams/MRG/00-overview/README|MRG Team Overview]]

---

## 🏷️ Tags

#mrg #service #geoservice #geolocation #maps #reverse-geocode #grpc #documentation

---

*Last Updated*: 2025-01-26  
*Generated from*: Repository analysis & code structure
