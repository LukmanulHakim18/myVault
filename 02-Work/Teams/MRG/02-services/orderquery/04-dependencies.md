# External Dependencies

## Overview
OrderQuery service bergantung pada berbagai external services (via gRPC) dan libraries untuk fungsi query dan data aggregation.

## Core Dependencies

### Database

#### PostgreSQL (Primary)
- **Purpose**: Main database untuk order data (microservices architecture)
- **Library**: `git.bluebird.id/mybb/qb-posgresql v1.6.3`
- **ORM**: Custom QB PostgreSQL adapter
- **Connection Pool**: Configurable via env vars
- **Tables**:
  - `order_summary` - Aggregated order view
  - `taxi_orders` - Taxi order details
  - `group_order` - Group order headers (shuttle)
  - `group_order_detail` - Group order line items
  - `group_order_detail_metadata` - Additional metadata
  - `chat_rooms` - Driver chat room mapping
  - `order_fees` - Fee breakdown
  - `order_vehicles` - Vehicle assignments
  - `pickup_instruction_suggestions` - User pickup instructions
  - `tips_driver` - Driver tips

#### PostgreSQL (Legacy)
- **Purpose**: Legacy monolith database for backward compatibility
- **Access Pattern**: Read-only
- **Migration Status**: Gradual migration to main DB
- **Tables**: Legacy order tables from monolith

### External Services (gRPC Clients)

#### User Service
- **Purpose**: Get user favorite addresses
- **Library**: `git.bluebird.id/mybb-ms/lib/userclient v0.0.21`
- **Protocol**: gRPC
- **Endpoints Used**:
  - `GetFavoriteAddresses` - Retrieve user's saved addresses
- **Criticality**: HIGH
- **Fallback**: Return empty list if service unavailable

#### Config Service
- **Purpose**: Feature flag validation (handling fee feature access)
- **Library**: `git.bluebird.id/mybb-ms/lib/configseviceclient v0.0.9`
- **Protocol**: gRPC
- **Endpoints Used**:
  - `ValidateFeatureAccess` - Check if feature is enabled for user
- **Criticality**: MEDIUM
- **Fallback**: Default to disabled if service unavailable

#### Session Manager
- **Purpose**: User session management
- **Library**: `git.bluebird.id/mybb-ms/lib/sessionmanagerclient v0.0.32`
- **Protocol**: gRPC
- **Endpoints Used**:
  - `GetSession` - Retrieve user session
  - `ValidateSession` - Validate session token
- **Criticality**: HIGH
- **Fallback**: Return error if session invalid

#### Content Provider
- **Purpose**: Get file resources (icons, images)
- **Library**: `git.bluebird.id/mybb-ms/lib/contentproviderclient v0.1.0`
- **Protocol**: gRPC
- **Endpoints Used**:
  - `GetFileResource` - Retrieve file URL by key
- **Icons Retrieved**:
  - Product type icons (cititrans_executive, silverbird, etc.)
  - Status icons (active, schedule, history)
  - Category icons (ride, shuttle, rent, delivery)
- **Criticality**: LOW
- **Fallback**: Return empty icon URL (UI shows placeholder)

#### Promo Gateway
- **Purpose**: Get promo/voucher information
- **Library**: `git.bluebird.id/mybb-ms/lib/promogatewayclient v1.1.16`
- **Protocol**: gRPC
- **Endpoints Used**:
  - `GetPromoByCode` - Get promo details
  - `GetActivePromos` - List active promotions
- **Criticality**: MEDIUM
- **Fallback**: Show without promo info

#### Fare Service
- **Purpose**: Fare calculation and reschedule confirmation
- **Library**: Custom client
- **Protocol**: gRPC
- **Endpoints Used**:
  - `GetFareEstimation` - Calculate fare for route
  - `GetRescheduleConfirmation` - Get new fare for reschedule
- **Criticality**: HIGH (for reschedule feature)
- **Fallback**: Cannot provide reschedule confirmation

#### Location Registry
- **Purpose**: Location data and mapping
- **Library**: Custom client
- **Protocol**: gRPC
- **Endpoints Used**:
  - `GetLocationByID` - Get location details
  - `SearchLocations` - Search locations
- **Criticality**: MEDIUM
- **Fallback**: Return lat/lng only without detailed info

#### GBOrderProcessor (GoldenBird)
- **Purpose**: GoldenBird specific order processing
- **Library**: `git.bluebird.id/mybb-ms/lib/gbopclient v0.0.6`
- **Protocol**: gRPC
- **Endpoints Used**:
  - `GetGoldenBirdOrderStatus` - Get GB order status
  - `GetGoldenBirdOrderDetails` - Get GB order details
- **Criticality**: MEDIUM (only for GB orders)
- **Fallback**: Return basic order info without GB-specific data

### External Services (HTTP)

#### Autocomplete
- **Purpose**: Address autocomplete suggestions
- **Protocol**: HTTP/REST
- **Endpoints Used**:
  - `GET /autocomplete` - Get address suggestions
- **Criticality**: LOW
- **Fallback**: No autocomplete suggestions

#### COP Service (Cititrans Order Processor)
- **Purpose**: Cititrans/shuttle order processing
- **Protocol**: HTTP/REST
- **Endpoints Used**:
  - `POST /orders/validate` - Validate shuttle order
  - `GET /orders/{id}` - Get shuttle order details
- **Criticality**: HIGH (for shuttle orders)
- **Fallback**: Cannot process shuttle operations

### Internal Libraries

#### Aphrodite (Core Services)
- **Purpose**: Common utilities and middleware
- **Library**: `git.bluebird.id/mybb-ms/aphrodite v1.9.34`
- **Provides**:
  - Structured logging
  - gRPC middleware
  - HTTP middleware
  - Error handling utilities
  - Context utilities
  - Interceptors

### Observability

#### Elastic APM
- **Purpose**: Application Performance Monitoring
- **Library**: `go.elastic.co/apm/v2 v2.6.2`
- **Modules**:
  - `apmgrpc` - gRPC instrumentation
  - `apmhttp` - HTTP instrumentation
  - `apmsql` - SQL instrumentation
- **Configuration**:
  ```bash
  ELASTIC_APM_SERVICE_NAME=orderquery
  ELASTIC_APM_SERVER_URL=http://apm-server:8200
  ELASTIC_APM_ENVIRONMENT=production
  ```
- **Captured Data**:
  - Transaction traces
  - External service calls (gRPC, HTTP)
  - Database queries
  - Error tracking
  - Span annotations

#### Prometheus
- **Purpose**: Metrics collection
- **Library**: `github.com/prometheus/client_golang v1.21.0`
- **Metrics Exposed**:
  - `orderquery_requests_total` - Request counter
  - `orderquery_request_duration_seconds` - Latency histogram
  - `orderquery_errors_total` - Error counter by code
  - `orderquery_db_queries_total` - Database query counter
  - `orderquery_external_calls_total` - External service call counter
- **Endpoint**: `/metrics`
- **Scrape Interval**: 15s

#### Structured Logging
- **Library**: `go.uber.org/zap v1.21.0`
- **Integration**: `go.elastic.co/ecszap v1.0.1` (ECS format)
- **Log Levels**: DEBUG, INFO, WARN, ERROR, FATAL
- **Output**:
  - Stdout (JSON format)
  - Elasticsearch (via Filebeat)
  - File rotation (via lumberjack)

### Protocol Buffers & gRPC

#### gRPC Core
- **Library**: `google.golang.org/grpc v1.75.1`
- **Middleware**: `github.com/grpc-ecosystem/go-grpc-middleware/v2 v2.0.1`
- **Features**:
  - Connection pooling
  - Load balancing
  - Retry logic
  - Circuit breaker (via gobreaker)

#### gRPC Gateway
- **Library**: `github.com/grpc-ecosystem/grpc-gateway/v2 v2.27.3`
- **Purpose**: HTTP/JSON to gRPC translation
- **Swagger**: Auto-generated from proto
- **Endpoints**: Mapped 1:1 with gRPC methods

#### Protocol Buffers
- **Library**: `google.golang.org/protobuf v1.36.10`
- **Code Generation**:
  - `protoc` with Go plugin
  - gRPC plugin
  - Gateway plugin
  - Swagger plugin

### Utilities

#### Configuration
- **Library**: `github.com/joho/godotenv v1.5.1`
- **Format**: `.env` files
- **Priority**: Environment vars > .env file > defaults
- **Validation**: At startup via `config/default.go`

#### String Utilities
- **Library**: `github.com/adrg/strutil v0.3.1`
- **Purpose**: String similarity and fuzzy matching
- **Use Cases**:
  - Address matching
  - Pickup instruction suggestions

#### UUID Generation
- **Library**: `github.com/google/uuid v1.6.0`
- **Purpose**: Generate unique identifiers
- **Use Cases**:
  - Request IDs
  - Trace IDs

## Service Dependencies Matrix

| Service | Protocol | Purpose | Criticality | Fallback |
|---------|----------|---------|-------------|----------|
| **PostgreSQL Main** | TCP/5432 | Primary data store | **CRITICAL** | Service down |
| **PostgreSQL Legacy** | TCP/5432 | Legacy data access | **CRITICAL** | Return error |
| **User Service** | gRPC | Favorite addresses | HIGH | Empty list |
| **Config Service** | gRPC | Feature flags | MEDIUM | Default off |
| **Session Manager** | gRPC | Session validation | HIGH | Auth error |
| **Content Provider** | gRPC | File resources | LOW | Empty icons |
| **Promo Gateway** | gRPC | Promo info | MEDIUM | No promo |
| **Fare Service** | gRPC | Fare calculation | HIGH | No estimate |
| **Location Registry** | gRPC | Location data | MEDIUM | Basic info |
| **GBOrderProcessor** | gRPC | GB orders | MEDIUM | Basic info |
| **Autocomplete** | HTTP | Address suggest | LOW | No suggest |
| **COP Service** | HTTP | Shuttle orders | HIGH | Error |
| **Elastic APM** | HTTP/8200 | APM tracing | LOW | No tracing |
| **Prometheus** | HTTP/9090 | Metrics | LOW | No metrics |

## Configuration

### Environment Variables

```bash
# Database - Main
DATABASE_HOST=postgres-main.internal
DATABASE_PORT=5432
DATABASE_NAME=orderquery
DATABASE_USER=orderquery_user
DATABASE_PASSWORD=***
DATABASE_SSL_MODE=require
DATABASE_MAX_OPEN_CONNS=25
DATABASE_MAX_IDLE_CONNS=5

# Database - Legacy
LEGACY_DATABASE_HOST=postgres-legacy.internal
LEGACY_DATABASE_PORT=5432
LEGACY_DATABASE_NAME=bluebird_legacy
LEGACY_DATABASE_USER=legacy_reader
LEGACY_DATABASE_PASSWORD=***
LEGACY_DATABASE_SSL_MODE=require

# External Services - gRPC
USER_SERVICE_URL=user:50051
CONFIG_SERVICE_URL=configservice:50051
SESSION_MANAGER_URL=sessionmanager:50051
CONTENT_PROVIDER_URL=contentprovider:50051
PROMO_GATEWAY_URL=promogateway:50051
FARE_SERVICE_URL=fareservice:50051
LOCATION_REGISTRY_URL=locationregistry:50051
GB_ORDER_PROCESSOR_URL=gborderprocessor:50051

# External Services - HTTP
AUTOCOMPLETE_URL=https://autocomplete.internal.bluebird.id
COP_SERVICE_URL=https://cop.internal.bluebird.id

# Service Config
GRPC_PORT=6005
REST_PORT=8005
LOG_LEVEL=INFO

# Observability
ELASTIC_APM_SERVER_URL=http://apm:8200
ELASTIC_APM_SERVICE_NAME=orderquery
ELASTIC_APM_ENVIRONMENT=production
ELASTIC_APM_TRANSACTION_SAMPLE_RATE=0.1
PROMETHEUS_PORT=9090

# Feature Flags
ENABLE_LEGACY_DB=true
ENABLE_FAVORITE_ADDRESSES=true
ENABLE_HANDLING_FEE=true
```

## Health Checks

### Dependency Health Checks
Service performs health checks on startup dan periodic checks setiap 30s:

```go
// PostgreSQL Main
SELECT 1

// PostgreSQL Legacy
SELECT 1

// User Service
healthcheck.Ping()

// Config Service
healthcheck.Ping()

// Session Manager
healthcheck.Ping()

// Content Provider
healthcheck.Ping()
```

### Health Check Endpoint
`GET /health` or `HealthCheck` gRPC method

**Response format**:
```json
{
  "code": "200",
  "message": "OK",
  "dependencies": {
    "postgresql_main": "healthy",
    "postgresql_legacy": "healthy",
    "user_service": "healthy",
    "config_service": "degraded",
    "session_manager": "healthy",
    "content_provider": "healthy"
  },
  "timestamp": "2025-01-07T10:00:00Z"
}
```

## Circuit Breaker Configuration

### gRPC Clients
All gRPC clients menggunakan circuit breaker pattern:

```go
circuitBreaker := gobreaker.NewCircuitBreaker(gobreaker.Settings{
    Name:        serviceName,
    MaxRequests: 3,
    Interval:    10 * time.Second,
    Timeout:     30 * time.Second,
    ReadyToTrip: func(counts gobreaker.Counts) bool {
        failureRatio := float64(counts.TotalFailures) / float64(counts.Requests)
        return counts.Requests >= 3 && failureRatio >= 0.6
    },
})
```

**States**:
- **Closed**: Normal operation
- **Open**: Too many failures, block requests (30s)
- **Half-Open**: Test if service recovered

## Retry Strategy

### gRPC Retry Policy
```go
retryPolicy := grpc.WithRetryPolicy(grpc.RetryPolicy{
    MaxAttempts:          3,
    InitialBackoff:       100 * time.Millisecond,
    MaxBackoff:           1 * time.Second,
    BackoffMultiplier:    2.0,
    RetryableStatusCodes: []codes.Code{
        codes.Unavailable,
        codes.ResourceExhausted,
    },
})
```

### HTTP Retry Policy
```go
retryClient := &http.Client{
    Timeout: 5 * time.Second,
    Transport: &RetryTransport{
        MaxRetries:    3,
        RetryDelay:    100 * time.Millisecond,
        RetryStatuses: []int{502, 503, 504},
    },
}
```

## Troubleshooting

### Common Issues

#### User Service Unavailable
```bash
# Check service status
kubectl get pods -l app=userservice

# Check connectivity
curl http://userservice:50051/health

# View logs
kubectl logs -f deployment/userservice
```

**Symptoms**: Favorite addresses not loading
**Impact**: Non-critical, fallback to empty list

---

#### Session Manager Timeout
```bash
# Check service health
grpcurl -plaintext sessionmanager:50051 health.HealthCheck

# Check network
telnet sessionmanager 50051
```

**Symptoms**: Authentication failures
**Impact**: Critical, users cannot access orders

---

#### Database Connection Exhausted
```bash
# Check active connections
SELECT count(*) FROM pg_stat_activity WHERE datname = 'orderquery';

# Check connection pool
SELECT * FROM pg_stat_database WHERE datname = 'orderquery';
```

**Symptoms**: Slow queries, timeouts
**Impact**: Critical, service degraded

**Solution**:
```bash
# Increase pool size in config
DATABASE_MAX_OPEN_CONNS=50
DATABASE_MAX_IDLE_CONNS=10
```

---

#### Content Provider Icons Missing
```bash
# Test content provider
grpcurl -d '{"key":"cititrans_executive"}' \
  -plaintext contentprovider:50051 \
  contentprovider.ContentProvider/GetFileResource
```

**Symptoms**: Missing icons in UI
**Impact**: Low, UI shows placeholders

**Solution**: Icons cached at CDN, service recovery auto-heals

---

## Dependency Updates

### Version Control
```bash
# Update all dependencies
go get -u ./...

# Update specific dependency
go get git.bluebird.id/mybb-ms/lib/userclient@latest

# Vendor dependencies
go mod vendor

# Verify dependencies
go mod verify
```

### Security Scanning
```bash
# Check for known vulnerabilities
govulncheck ./...

# Or use Nancy
go list -json -m all | nancy sleuth
```

### Private Dependencies
```bash
# Configure Git for Bluebird repos
git config --global url."git@git.bluebird.id:".insteadOf "https://git.bluebird.id/"

# Configure GOPRIVATE
export GOPRIVATE=git.bluebird.id/*
```

## Performance Tuning

### Connection Pool Tuning
```bash
# Main DB
DATABASE_MAX_OPEN_CONNS=25      # Max concurrent connections
DATABASE_MAX_IDLE_CONNS=5       # Keep alive connections
DATABASE_CONN_MAX_LIFETIME=3600 # Max connection lifetime (seconds)

# Legacy DB
LEGACY_DATABASE_MAX_OPEN_CONNS=10  # Lower for read-only
LEGACY_DATABASE_MAX_IDLE_CONNS=2
```

### gRPC Client Tuning
```bash
# Connection pool per service
GRPC_MAX_CONNECTIONS=10
GRPC_KEEPALIVE_TIME=30s
GRPC_KEEPALIVE_TIMEOUT=10s
```

### Cache Configuration (if implemented)
```bash
# Redis for caching (optional)
REDIS_HOST=redis.internal
REDIS_PORT=6379
REDIS_DB=0
REDIS_TTL=300  # 5 minutes
```

## Related Documentation
- [[01-overview|Service Overview]]
- [[02-api-reference|API Reference]]
- [[03-group-order-flows|Group Order Flows]]
- [[k8s/huawei-application.yaml|Kubernetes Configuration]]

---
**Last Updated**: 2025-01-07
**Dependency Audit**: 2025-01-01
**Security Scan**: Pass