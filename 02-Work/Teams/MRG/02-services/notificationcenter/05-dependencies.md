# External Dependencies

## Overview
Notification Center service bergantung pada berbagai external services dan libraries untuk menjalankan fungsi notifikasi multi-channel.

## Core Dependencies

### Database & Cache

#### PostgreSQL
- **Purpose**: Primary database untuk logging dan state management
- **Library**: `git.bluebird.id/mybb/qb-posgresql v1.6.2`
- **ORM**: GORM-based
- **Tables**:
  - `email_logs` - Email delivery tracking
  - `sms_logs` - SMS delivery tracking
  - `whatsapp_logs` - WhatsApp message tracking
  - `notification_logs` - Push notification tracking
- **Connection Pool**: Configurable via env vars
- **Migrations**: Manual SQL scripts in `sql/schema.sql`

#### Redis
- **Purpose**: 
  - Duplicate request prevention (10s TTL)
  - Rate limiting
  - Caching configuration
- **Library**: `github.com/go-redis/redis/v8 v8.11.5`
- **APM**: `go.elastic.co/apm/module/apmgoredisv8/v2 v2.4.7`
- **Key Patterns**:
  - `DUPLICATE_LOCK:notification-center:{function}:{hash}`
  - `RATE_LIMIT:{user_id}:{channel}`
  - `SMS_CONFIG:*`

### Messaging & Events

#### Google Cloud PubSub
- **Purpose**: Event-driven notifications
- **Library**: `cloud.google.com/go/pubsub v1.50.1`
- **Topics**:
  - `booking.events` - Booking lifecycle events
  - `order.item.events` - Order item updates
  - `payment.events` - Payment transactions
- **Subscription**: Pull-based with configurable concurrency
- **Dead Letter Queue**: Configured for failed messages

#### Kafka (via Watermill)
- **Purpose**: Alternative message broker
- **Library**: `github.com/ThreeDotsLabs/watermill-kafka/v3 v3.0.5`
- **Middleware**: Kafka AMQP adapter
- **Topics**: Similar to PubSub topics
- **Consumer Groups**: Configured per environment

#### RabbitMQ (Optional)
- **Library**: `github.com/ThreeDotsLabs/watermill-amqp/v3 v3.0.0`
- **Purpose**: Fallback message broker
- **Queues**: Same as Kafka topics

### Notification Providers

#### Firebase Cloud Messaging (FCM)
- **Purpose**: Push notifications to mobile apps
- **Library**: `firebase.google.com/go/v4 v4.14.0`
- **Authentication**: Service account JSON
- **Features**:
  - Android push notifications
  - iOS push notifications (APNs via FCM)
  - Topic-based messaging
  - Device group messaging
- **Configuration**:
  ```go
  FIREBASE_PROJECT_ID=mybb-production
  FIREBASE_CREDENTIALS_PATH=/path/to/service-account.json
  ```

#### BBOne Email Service
- **Purpose**: Primary email provider (Bluebird internal)
- **Library**: Internal Bluebird package
- **Endpoints**:
  - Send email with attachments
  - Bulk email sending
  - Template-based emails
- **Features**:
  - HTML email support
  - PDF attachments
  - Image embedding
  - CC/BCC support
- **Rate Limit**: 50 emails/hour per user

#### SendGrid (Backup)
- **Library**: `gopkg.in/gomail.v2 v2.0.0-20160411212932-81ebce5c23df`
- **Purpose**: Backup email provider when BBOne unavailable
- **API Key**: Stored in env vars
- **Features**: Similar to BBOne

#### DART SMS Gateway
- **Purpose**: Primary SMS provider
- **Library**: `git.bluebird.id/mybb-ms/lib/smsclient v0.1.5`
- **Authentication**: Username + Password (from env)
- **Configuration**:
  ```go
  DART_USER_ID=BB9871As9G
  DART_PASSWORD=KunAsPass786ShH
  DART_ORIGINAL=BLUEBIRD
  ```
- **Features**:
  - International SMS
  - Delivery reports
  - Prefix whitelist
- **Rate Limit**: 10 SMS/hour per user

#### WhatsApp Meta API
- **Purpose**: WhatsApp messaging (primarily OTP)
- **Authentication**: Access token
- **Endpoints**:
  - Send template message
  - Send text message
  - Media messages
- **Webhooks**: 
  - Delivery status callback
  - Read receipts
- **Template Management**: Pre-approved templates on Meta Business

#### Unicush (Legacy Push)
- **Purpose**: Legacy push notification system
- **Status**: Being migrated to FCM
- **Library**: Internal package `model/uniqush.go`

### Internal Services

#### Aphrodite (Core Services)
- **Purpose**: Common utilities and middleware
- **Library**: `git.bluebird.id/mybb-ms/aphrodite v1.9.41`
- **Provides**:
  - Structured logging
  - gRPC middleware
  - HTTP middleware
  - Error handling utilities
  - Context utilities

#### Bluebird Chassis
- **Purpose**: Service framework
- **Library**: `git.bluebird.id/golang-core/bluebird-chassis v0.3.2`
- **Features**:
  - Service discovery
  - Health checks
  - Metrics collection
  - Configuration management

### Observability

#### Elastic APM
- **Purpose**: Application Performance Monitoring
- **Library**: `go.elastic.co/apm/v2 v2.6.2`
- **Modules**:
  - `apmgrpc` - gRPC instrumentation
  - `apmhttp` - HTTP instrumentation
  - `apmgoredisv8` - Redis instrumentation
  - `apmsql` - SQL instrumentation
- **Configuration**:
  ```go
  ELASTIC_APM_SERVICE_NAME=notificationcenter
  ELASTIC_APM_SERVER_URL=http://apm-server:8200
  ELASTIC_APM_ENVIRONMENT=production
  ```
- **Captured Data**:
  - Transaction traces
  - Error tracking
  - Metrics (latency, throughput)
  - Span annotations

#### Prometheus
- **Purpose**: Metrics collection and alerting
- **Library**: `github.com/prometheus/client_golang v1.21.0`
- **Metrics Exposed**:
  - `notification_sent_total` - Counter per channel
  - `notification_failed_total` - Failure counter
  - `notification_duration_seconds` - Histogram
  - `notification_queue_length` - Gauge
- **Endpoint**: `/metrics`
- **Scrape Interval**: 15s

#### Structured Logging
- **Library**: `go.uber.org/zap v1.21.0`
- **Integration**: `go.elastic.co/ecszap v1.0.1` (ECS format)
- **Log Levels**: DEBUG, INFO, WARN, ERROR, FATAL
- **Output**: 
  - Stdout (JSON format)
  - Elasticsearch (via Filebeat)

### Protocol Buffers & gRPC

#### gRPC Core
- **Library**: `google.golang.org/grpc v1.75.1`
- **Middleware**: `github.com/grpc-ecosystem/go-grpc-middleware/v2 v2.0.1`
- **Features**:
  - Connection pooling
  - Load balancing
  - Retry logic
  - Circuit breaker

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

#### PDF Generation
- **Library**: `github.com/SebastiaanKlippert/go-wkhtmltopdf v1.9.3`
- **Purpose**: Generate PDF receipts/invoices
- **Requirements**: `wkhtmltopdf` binary installed
- **Features**:
  - HTML to PDF conversion
  - Custom page sizes
  - Header/footer support

#### Template Engine
- **Library**: Go's `html/template` (stdlib)
- **Location**: `util/template/`
- **Templates**:
  - Email templates (HTML)
  - SMS templates
  - Push notification templates
- **Parsing**: Pre-parsed at startup

#### Configuration
- **Library**: `github.com/joho/godotenv v1.5.1`
- **Format**: `.env` files
- **Priority**: Environment vars > .env file > defaults
- **Validation**: At startup via `config/default.go`

## Dependency Management

### Version Control
```bash
# Update all dependencies
go get -u ./...

# Update specific dependency
go get git.bluebird.id/mybb-ms/aphrodite@latest

# Vendor dependencies
go mod vendor

# Verify dependencies
go mod verify
```

### Security Scanning
```bash
# Check for known vulnerabilities
go list -json -m all | nancy sleuth

# Or use govulncheck
govulncheck ./...
```

### Private Dependencies
```bash
# Configure Git for Bluebird repos
git config --global url."git@git.bluebird.id:".insteadOf "https://git.bluebird.id/"

# Configure GOPRIVATE
export GOPRIVATE=git.bluebird.id/*
```

## Service Dependencies Matrix

| Service | Protocol | Purpose | Criticality |
|---------|----------|---------|-------------|
| PostgreSQL | TCP/5432 | Data persistence | **CRITICAL** |
| Redis | TCP/6379 | Caching & locking | **HIGH** |
| FCM | HTTPS | Push notifications | **HIGH** |
| BBOne Email | HTTPS | Email delivery | **HIGH** |
| DART SMS | HTTPS | SMS delivery | **MEDIUM** |
| WhatsApp Meta | HTTPS | WhatsApp messages | **MEDIUM** |
| PubSub/Kafka | TCP | Event streaming | **HIGH** |
| Elastic APM | HTTP/8200 | APM tracing | **LOW** |
| Prometheus | HTTP/9090 | Metrics | **LOW** |

## Fallback Strategies

### Email Fallback
```
BBOne (primary) 
  ↓ [timeout/error]
SendGrid (backup)
  ↓ [timeout/error]
Log failure + Retry queue
```

### SMS Fallback
```
DART Gateway (primary)
  ↓ [timeout/error]
Log failure + Retry queue
  ↓ [after 3 retries]
Alert on-call team
```

### Push Notification Fallback
```
FCM (primary)
  ↓ [timeout/error]
Unicush Legacy (backup, deprecated)
  ↓ [timeout/error]
Log failure + Retry queue
```

### Message Broker Fallback
```
PubSub (GCP, primary)
  ↓ [unavailable]
Kafka (self-hosted, backup)
  ↓ [unavailable]
RabbitMQ (last resort)
```

## Health Checks

### Dependency Health Checks
Service performs health checks pada startup dan periodic check setiap 30s:

```go
// PostgreSQL
SELECT 1

// Redis
PING

// FCM
Validate credentials

// BBOne Email
GET /health

// DART SMS
GET /status
```

Health check endpoint: `GET /health`

Response format:
```json
{
  "status": "healthy",
  "dependencies": {
    "postgresql": "healthy",
    "redis": "healthy",
    "fcm": "healthy",
    "bbone_email": "degraded",
    "dart_sms": "healthy"
  },
  "timestamp": "2025-01-07T10:00:00Z"
}
```

## Configuration

### Environment Variables

```bash
# Database
DATABASE_HOST=postgres.internal
DATABASE_PORT=5432
DATABASE_NAME=notificationcenter
DATABASE_USER=notif_user
DATABASE_PASSWORD=***
DATABASE_SSL_MODE=require

# Redis
REDIS_HOST=redis.internal
REDIS_PORT=6379
REDIS_PASSWORD=***
REDIS_DB=0

# Firebase
FIREBASE_PROJECT_ID=mybb-production
FIREBASE_CREDENTIALS_PATH=/etc/secrets/firebase.json

# BBOne Email
BBONE_EMAIL_URL=https://email.internal.bluebird.id
BBONE_EMAIL_API_KEY=***

# DART SMS
DART_USER_ID=***
DART_PASSWORD=***
DART_ORIGINAL=BLUEBIRD

# WhatsApp
WHATSAPP_META_TOKEN=***
WHATSAPP_META_PHONE_ID=***

# PubSub
PUBSUB_PROJECT_ID=mybb-production
PUBSUB_CREDENTIALS=/etc/secrets/pubsub.json
ENABLE_PUBSUB_NOTIFICATION=true

# Kafka (if enabled)
KAFKA_BROKERS=kafka1:9092,kafka2:9092
KAFKA_GROUP_ID=notificationcenter-consumer

# Observability
ELASTIC_APM_SERVER_URL=http://apm:8200
ELASTIC_APM_SERVICE_NAME=notificationcenter
ELASTIC_APM_ENVIRONMENT=production
PROMETHEUS_PORT=9090

# Service Config
GRPC_PORT=50051
REST_PORT=8080
LOG_LEVEL=INFO
```

## Troubleshooting

### Common Issues

#### FCM Authentication Failed
```bash
# Check credentials file
cat $FIREBASE_CREDENTIALS_PATH

# Verify project ID
gcloud projects list

# Re-download service account JSON
# From Firebase Console > Project Settings > Service Accounts
```

#### BBOne Email Timeout
```bash
# Test connectivity
curl https://email.internal.bluebird.id/health

# Check API key
echo $BBONE_EMAIL_API_KEY | base64 -d

# View logs
tail -f logs/bbone_email.log
```

#### Redis Connection Refused
```bash
# Test Redis connection
redis-cli -h $REDIS_HOST -p $REDIS_PORT -a $REDIS_PASSWORD ping

# Check network
telnet $REDIS_HOST $REDIS_PORT
```

#### PubSub Subscription Backlog
```bash
# Check subscription backlog
gcloud pubsub subscriptions describe notificationcenter-booking-events

# Increase concurrent consumers
kubectl scale deployment notificationcenter --replicas=5

# Purge old messages (if safe)
gcloud pubsub subscriptions seek notificationcenter-booking-events --time=$(date -u +%Y-%m-%dT%H:%M:%SZ)
```

## Related Documentation
- [[01-overview|Service Overview]]
- [[04-api-reference|API Reference]]
- [[k8s/huawei-application.yaml|Kubernetes Configuration]]

---
**Last Updated**: 2025-01-07
**Dependency Audit**: 2025-01-01
**Security Scan**: Pass