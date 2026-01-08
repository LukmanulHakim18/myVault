# HO Gateway - Dependencies

[[README|← Back to Overview]]

---

## Internal Service Dependencies

### 1. Config Service

**Purpose**: Operational city mapping  
**Repository**: `git.bluebird.id/mybb-ms/configservice`  
**Protocol**: gRPC  
**Client Library**: `configseviceclient`

**Usage**:
```go
operationalCity, err := repo.ConfigService.GetOperationalCityByItopID(ctx, itopID)
// Returns: OperationalCity{
//   ItopID: "JKT01",
//   AreaCode: "JKT",
//   CityName: "Jakarta",
// }
```

**Integration Points**:
- **Method**: `GetOperationalCityByItopID`
- **Called When**: Every e-wallet transaction processing
- **Failure Impact**: Transaction cannot be processed (returns error)

**Data Mapping**:
| iTOP ID | Area Code | City |
|---------|-----------|------|
| JKT01 | JKT | Jakarta |
| BDG01 | BDG | Bandung |
| SBY01 | SBY | Surabaya |
| DPS01 | DPS | Denpasar |

**Why Needed**:
- HO system uses area codes (JKT, BDG, etc.) not iTOP IDs
- Config Service maintains the mapping
- Enables multi-city operations

**Configuration**:
```env
CONFIGSERVICE_HOST=configservice.microservices.svc.cluster.local
CONFIGSERVICE_PORT=6000
```

---

### 2. HO Legacy Service

**Purpose**: Target system for transaction sync  
**Type**: External REST API (Legacy System)  
**Protocol**: HTTP REST

**Endpoint**:
```
POST ${HO_SERVICE_URL}/ho-ewallet
Content-Type: application/json
```

**Request Format**:
```json
{
  "TrxType": "RSV_DP",
  "TrxDate": "20250108",
  "TrxTime": "103000",
  "Area": "JKT",
  "OrderId": "ORD-123456",
  "Amount": 150000,
  "PayGateway": "MIDGPY",
  ...
}
```

**Response Format**:
```json
{
  "code": 200,
  "TrxID": "HO-TXN-123",
  "error": "",
  "Message": "Success"
}
```

**Integration Points**:
- **Method**: `HoEWalletTransaction`
- **Called When**: After creating transaction record
- **Failure Impact**: Retry with exponential backoff

**Retry Strategy**:
- Max attempts: Configurable (default 3)
- Retry delay: 1 second between attempts
- Republish to RabbitMQ on failure

**Configuration**:
```env
HO_SERVICE_URL=http://ho-service.legacy.svc
HO_SERVICE_TIMEOUT=30s
```

---

### 3. Payment Processor (UPG)

**Purpose**: Source of payment events  
**Repository**: `git.bluebird.id/mybb-ms/paymentprocessor`  
**Protocol**: RabbitMQ (Message Queue)

**Message Flow**:
```
Payment Processor → RabbitMQ → HO Gateway
```

**Topic**: `ho_e_wallet_transaction`

**Message Format**:
```json
{
  "payment_method": "gopay",
  "trx_date_time": "2025-01-08T10:30:00+07:00",
  "order_id": 789012,
  "amount": 150000,
  "transaction_id": "TXN-GOPAY-123",
  ...
}
```

**Integration Points**:
- **Event**: Payment webhook success
- **Publisher**: Payment Processor (UPG)
- **Consumer**: HO Gateway
- **Trigger**: Customer completes e-wallet payment

**Failure Handling**:
- Message stays in queue until acknowledged
- Retry mechanism built into HO Gateway
- Dead letter queue for max failures

---

## Infrastructure Dependencies

### 1. PostgreSQL

**Purpose**: Transaction records and attempt logs  
**Version**: 12+  
**Connection**: Via connection pool

**Tables**:
- `ho_transactions` - Main transaction records
- `ho_transaction_attempts` - Retry attempt logs

**Configuration**:
```env
DB_HOST=postgres.microservices.svc.cluster.local
DB_PORT=5432
DB_NAME=ho_gateway
DB_USERNAME=ho_gateway_user
DB_PASSWORD=<secret>
DB_SSL_MODE=disable
MAX_IDLE_CONNS=10
MAX_OPEN_CONNS=100
```

**Schema Management**:
- Migrations in: `model/migrations/`
- Auto-migration on startup: ❌ (manual deployment)

**Performance Considerations**:
- Connection pooling configured
- Indexes on `order_id` for fast lookup
- Index on `is_success` for reporting

---

### 2. RabbitMQ

**Purpose**: Message broker for async processing  
**Version**: 3.x  
**Protocol**: AMQP

**Topics**:
- `ho_e_wallet_transaction` - E-wallet transaction events

**Configuration**:
```env
RABBITMQ_URI=amqp://user:pass@rabbitmq:5672/
RABBITMQ_HO_E_WALLET_TOPIC=ho_e_wallet_transaction
```

**Consumer Settings**:
- Prefetch count: 1
- Auto-acknowledge: false (manual ack after success)
- Retry: Built into application logic

**Publisher Settings**:
- Delivery mode: Persistent
- Mandatory: true
- Immediate: false

**Failure Scenarios**:
1. Message consumed but HO call fails → Retry via republish
2. Database error → NACK, message returns to queue
3. Max retry reached → Publish to dead letter queue

---

### 3. Elastic APM

**Purpose**: Application Performance Monitoring  
**Version**: 7.x+  
**Protocol**: HTTP

**Configuration**:
```env
ELASTIC_APM_SERVICE_NAME=mybb-ho-gateway
ELASTIC_APM_SERVER_URL=http://apm-server:8200
ELASTIC_APM_ENVIRONMENT=production
ELASTIC_APM_SECRET_TOKEN=<secret>
```

**Traced Operations**:
- gRPC method calls
- HTTP client requests
- Database queries
- Message queue operations

**Span Hierarchy**:
```
HoEWalletTransaction (usecase)
├── GetOperationalCityByItopID (configservice)
├── CreateHoTransaction (database)
├── HoEWalletTransaction (ho_service)
└── UpdateHoTransactionAttempt (database)
```

**Metrics Collected**:
- Request duration
- Error rate
- Transaction throughput
- External service latency

---

### 4. Prometheus

**Purpose**: Metrics collection and alerting  
**Version**: 2.x  
**Protocol**: HTTP (scrape)

**Metrics Endpoint**: `http://<host>:8082/metrics`

**Custom Metrics**:
```
# Transaction attempts
ho_gateway_transaction_attempts_total{status="success|failed"}
ho_gateway_transaction_retry_total
ho_gateway_transaction_max_retry_total

# HO Service calls
ho_gateway_ho_service_duration_seconds
ho_gateway_ho_service_errors_total

# Message queue
ho_gateway_rabbitmq_messages_consumed_total
ho_gateway_rabbitmq_messages_published_total
```

**Standard Metrics**:
- Go runtime metrics
- gRPC server metrics
- HTTP server metrics

---

## External Libraries

### 1. Aphrodite (Internal Common Library)

**Repository**: `git.bluebird.id/mybb-ms/aphrodite`  
**Version**: v1.9.24

**Modules Used**:
- `logger` - Structured logging with context
- `lang` - Multi-language support
- `microservice` - Service utilities
- `grpc` - gRPC helpers
- `http` - HTTP client utilities
- `pubsub` - Message broker abstraction

**Key Features**:
```go
// Logging
cLogger.ConvertMapToFields(logData)
logger.GetLogger().InfoWithContext(ctx, "message", fields...)

// Language support
cLang.CustomTextByLanguage(ctx, "English", "Indonesian")
```

---

### 2. Bluebird Chassis

**Repository**: `git.bluebird.id/golang-core/bluebird-chassis`  
**Version**: v0.3.6

**Modules Used**:
- gRPC middleware
- Service discovery
- Health checks
- Metrics collection

---

### 3. gRPC Ecosystem

**Libraries**:
- `google.golang.org/grpc` v1.75.0 - Core gRPC
- `google.golang.org/protobuf` v1.36.8 - Protocol buffers
- `github.com/grpc-ecosystem/grpc-gateway/v2` v2.27.2 - REST gateway
- `github.com/grpc-ecosystem/go-grpc-middleware/v2` v2.3.2 - Middleware

**Purpose**:
- gRPC server and client
- REST API gateway
- Request logging and tracing
- Error handling

---

### 4. Other Key Dependencies

| Library | Version | Purpose |
|---------|---------|---------|
| `github.com/lib/pq` | v1.10.9 | PostgreSQL driver |
| `github.com/rabbitmq/amqp091-go` | v1.10.0 | RabbitMQ client |
| `go.elastic.co/apm/v2` | v2.7.1 | Elastic APM agent |
| `github.com/prometheus/client_golang` | v1.23.0 | Prometheus client |
| `github.com/joho/godotenv` | v1.5.1 | Environment loader |

---

## Dependency Graph

```mermaid
graph TD
    HOG[HO Gateway]
    
    HOG -->|gRPC| CS[Config Service]
    HOG -->|HTTP| HO[HO Legacy Service]
    HOG -->|Consume| RMQ[RabbitMQ]
    HOG -->|Publish| RMQ
    HOG -->|Store| DB[(PostgreSQL)]
    HOG -->|Trace| APM[Elastic APM]
    HOG -->|Metrics| PROM[Prometheus]
    
    UPG[Payment Processor] -->|Publish| RMQ
    
    CS -->|Read| CSDB[(Config DB)]
    
    style HOG fill:#ff6b6b
    style CS fill:#4ecdc4
    style HO fill:#ffe66d
    style RMQ fill:#a8e6cf
    style DB fill:#ff8b94
```

---

## Service Communication

### Synchronous (Request-Response)

1. **HO Gateway → Config Service**
   - Protocol: gRPC
   - Timeout: 5s
   - Retry: None (fail fast)

2. **HO Gateway → HO Legacy Service**
   - Protocol: HTTP REST
   - Timeout: 30s
   - Retry: 3 attempts with 1s delay

### Asynchronous (Message Queue)

1. **Payment Processor → HO Gateway**
   - Protocol: RabbitMQ AMQP
   - Topic: `ho_e_wallet_transaction`
   - Delivery: At-least-once

2. **HO Gateway → HO Gateway (Retry)**
   - Protocol: RabbitMQ AMQP
   - Topic: `ho_e_wallet_transaction`
   - Trigger: HO service failure

---

## Deployment Dependencies

### Kubernetes Resources

**Required**:
- `ConfigMap` - Application configuration
- `Secret` - Database credentials, service URLs
- `Service` - Internal service discovery
- `Deployment` - Pod management

**Optional**:
- `HorizontalPodAutoscaler` - Auto-scaling
- `PodDisruptionBudget` - High availability
- `ServiceMonitor` - Prometheus scraping

### Network Policies

**Ingress** (Allowed):
- From: Payment Processor (UPG)
- From: RabbitMQ
- From: Kubernetes API

**Egress** (Allowed):
- To: Config Service
- To: HO Legacy Service
- To: PostgreSQL
- To: RabbitMQ
- To: Elastic APM
- To: Kubernetes DNS

---

## Version Compatibility

### Minimum Versions

| Component | Minimum Version |
|-----------|----------------|
| Go | 1.25 |
| PostgreSQL | 12 |
| RabbitMQ | 3.8 |
| Kubernetes | 1.20 |

### Tested Versions

| Component | Version |
|-----------|---------|
| Go | 1.25.0 |
| PostgreSQL | 14.5 |
| RabbitMQ | 3.12 |
| Kubernetes | 1.27 |

---

## Related Documentation

- [[README|Service Overview]]
- [[api-reference|API Reference]]
- [[configservice]] - Config Service docs
- [[paymentprocessor]] - Payment Processor docs

---

*Last Updated*: 2025-01-08
