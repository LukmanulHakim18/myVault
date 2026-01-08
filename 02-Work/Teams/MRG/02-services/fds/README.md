---
tags:
  - mrg
  - service
  - fds
  - fraud-detection
  - security
  - grpc
  - documentation
team: MRG
type: service-documentation
title: FDS Service
status: production
created: '2025-01-05'
updated: '2025-01-05'
grpc_port: 6001
rest_port: 8001
repository: git.bluebird.id/mybb-ms/fds
tech_stack:
  - go
  - grpc
  - postgresql
  - redis
---
# FDS Service (Fraud Detection System)

**Team**: MRG (Meta Reservation Gateway)  
**Status**: ✅ Production  
**Repository**: `git.bluebird.id/mybb-ms/fds`

---

## 📋 Overview

MyBB Fraud Detection System (FDS) adalah microservice yang bertanggung jawab untuk mendeteksi dan mencegah aktivitas fraud dalam ekosistem MyBluebird. Service ini menyediakan mekanisme keamanan untuk melindungi sistem dari penyalahgunaan seperti spam OTP, cancel order berlebihan, dan brute force password.

### Fungsi Utama

- **Soft Ban Management** - Ban sementara otomatis berdasarkan threshold (cancel order, OTP spam, wrong password)
- **Hard Ban Management** - Ban permanen oleh admin untuk user fraud
- **Phone Number Scanning** - Deteksi nomor telepon yang sudah di-banned
- **Whitelist Management** - Kelola nomor telepon yang dikecualikan dari banned
- **Counter Tracking** - Track attempt counter untuk berbagai aktivitas

---

## 🛠️ Tech Stack

| Component | Technology |
|-----------|------------|
| Language | Go 1.23+ |
| Protocol | gRPC + REST (gRPC-Gateway) |
| Database | PostgreSQL |
| Cache | Redis |
| Monitoring | Elastic APM, Prometheus |
| Container | Docker, Kubernetes |

---

## 🔑 Konsep Utama

### 1. Soft Banned (Temporary Ban)

Ban otomatis oleh sistem ketika user mencapai threshold tertentu. Ban ini memiliki durasi terbatas dan akan otomatis release.

| Banned Type | Threshold | Duration Window | Ban Duration | Used In |
|-------------|-----------|-----------------|--------------|---------|
| `CANCEL_ORDER` | 5 attempts | 2 hours | 24 hours | `create_order` |
| `CANCEL_ORDER_AFTER_DISPATCH` | 5 attempts | 2 hours | 24 hours | After driver dispatch |
| `OTP_ATTEMPT` | 3 attempts | 24 hours | 24 hours | `send_otp` |
| `WRONG_PASSWORD` | 5 attempts | 2 minutes | 5 minutes | `login` |

**Flow:**
```
User Action → Increment Counter → Check Threshold → If Exceeded → Apply Soft Ban
```

### 2. Hard Banned (Permanent Ban)

Ban permanen yang dilakukan oleh admin untuk user yang terbukti melakukan fraud. Hard ban memerlukan revoke manual.

**Identifier Types:**
- `BBID` - BlueiBird ID
- `PHONE_NUMBER` - Nomor telepon
- `DEVICE_ID` - Device identifier

**Features:**
- Login token tracking untuk force logout
- Bilingual reason (ID/EN)
- Soft delete support

### 3. Phone Number Scanning

Deteksi fraud berdasarkan prefix digit nomor telepon. Digunakan untuk validasi saat registrasi atau login.

### 4. Whitelist

Daftar nomor telepon yang dikecualikan dari semua jenis banned. Biasanya untuk internal testing atau VIP users.

---

## 🔌 Dependencies

### Infrastructure

| Component | Purpose |
|-----------|---------|
| **PostgreSQL** | Hard banned storage, soft banned config |
| **PostgreSQL Legacy** | Legacy gateway database |
| **Redis** | Counter storage, soft banned cache |

### Repository Structure

```go
type Repository struct {
    FDS    repoiface.FraudDetectionSystem
    Redis  repoiface.Redis
    DB     DB
    Legacy repoiface.Legacy
}

type DB struct {
    HardBanned       repoiface.HardBanned
    SoftBannedConfig repoiface.SoftBannedConfig
}
```

---

## 📡 API Contracts

### gRPC Service

**Package**: `fds`  
**Proto File**: `contract/fds.proto`  
**Ports**: gRPC `6001`/`6101`, REST `8001`/`8101`

### Methods Overview

#### Health Check

| Method | Description | Interceptors |
|--------|-------------|--------------|
| `HealthCheck` | Service health check | - |

#### Soft Ban Operations

| Method | Description | Interceptors |
|--------|-------------|--------------|
| `GetSoftBanned` | Get all soft bans for identifier | ValidateInput |
| `GetSoftBannedByType` | Get specific soft ban by type | ValidateInput |
| `SoftBannedCounter` | Increment attempt counter | ValidateInput |
| `RevokeSoftBanned` | Remove soft ban | ValidateInput |

#### Hard Ban Operations

| Method | Description | Interceptors |
|--------|-------------|--------------|
| `SetHardBanned` | Create hard ban | ValidateInput |
| `CheckHardBanned` | Check if identifier is hard banned | ValidateInput |
| `GetHardBanned` | Get hard ban details by ID | ValidateInput |
| `RevokeHardBanned` | Remove hard ban | ValidateInput |

#### Phone Number Operations

| Method | Description | Interceptors |
|--------|-------------|--------------|
| `ScanFraudPhoneNumber` | Check if phone is banned | - |
| `WhitelistPhoneNumbers` | Add phones to whitelist | ValidateInput |

---

## ⚙️ Configuration

### Environment Variables

```env
# Application
APP_NAME=mybb-fraud-detection-system
GRPC_PORT=6101
REST_PORT=8101
CHECK_HEALTHY_REPO=true

# Logging
LOG_LEVEL=debug
LOG_DIRECTORY=

# Redis
REDIS_HOST=172.26.11.40
REDIS_PORT=6379
REDIS_DATABASE=6
REDIS_PASSWORD=

# Phone Fraud Detection
MAX_ATTEMPT=2
PREFIX_DIGIT_DETECTION=9
EXTEND_BANNED=true
ATTEMPT_DURATION=20m
BANNED_DURATION=24h

# Cancel Order Soft Ban
THRESHOLD_CANCEL_ORDER_ATTEMPT=5
THRESHOLD_CANCEL_ORDER_DURATION=2h
BAN_CANCEL_ORDER_DURATION=24h

# Cancel Order After Dispatch Soft Ban
THRESHOLD_CANCEL_ORDER_AFTER_DISPATCH_ATTEMPT=5
THRESHOLD_CANCEL_ORDER_AFTER_DISPATCH_DURATION=2h
BAN_CANCEL_ORDER_AFTER_DISPATCH_DURATION=24h

# OTP Attempt Soft Ban
THRESHOLD_OTP_ATTEMPT=3
THRESHOLD_OTP_ATTEMPT_DURATION=24h
BAN_OTP_ATTEMPT_DURATION=24h

# Wrong Password Soft Ban
THRESHOLD_WRONG_PASSWORD_ATTEMPT=5
THRESHOLD_WRONG_PASSWORD_DURATION=2m
BAN_WRONG_PASSWORD_DURATION=5m

# Database (Main)
DB_HOST=
DB_USERNAME=
DB_PASSWORD=
DB_SSL_MODE=disable
DB_PORT=5432
DB_NAME=mybbnew_fraud_detection_system_db
MAX_IDLE_CONNS=10
MAX_OPEN_CONNS=100

# Database (Legacy)
DB_LEGACY_HOST=
DB_LEGACY_USERNAME=
DB_LEGACY_PASSWORD=
DB_LEGACY_PORT=5432
DB_LEGACY_SSL_MODE=disabled
DB_LEGACY_NAME=mybb_gateway_db

# Debug
DEBUG_MODE=true
```

---

## 📂 Project Structure

```
fds/
├── main.go                    # Entry point
├── go.mod                     # Dependencies
├── Dockerfile                 # Container build
├── Jenkinsfile                # CI/CD pipeline
├── README.md                  # Project documentation
│
├── config/                    # Configuration
│   ├── default.go             # Default config values
│   ├── logger/                # Logger setup
│   └── repository/            # Repository initialization
│
├── constants/                 # Constants
│   ├── soft_ban.go            # Soft ban types
│   └── whitelist.go           # Whitelist constants
│
├── contract/                  # API contracts
│   ├── fds.proto              # gRPC proto definition
│   ├── fds.pb.go              # Generated protobuf
│   ├── fds_grpc.pb.go         # Generated gRPC
│   ├── fds.pb.gw.go           # REST gateway
│   ├── fds.swagger.json       # Swagger spec
│   ├── mapper.go              # DTO mappers
│   ├── response.go            # Response helpers
│   └── validate_message.go    # Input validation
│
├── model/                     # Domain models
│   ├── hard_banned.go         # Hard banned model
│   ├── soft_banned.go         # Soft banned model
│   └── migration.sql          # Database migrations
│
├── repository/                # Data access layer
│   ├── base_repository.go     # Repository base
│   ├── repoiface/             # Interfaces
│   │   ├── db.go
│   │   ├── fds.go
│   │   ├── legacy.go
│   │   └── redis.go
│   ├── repomock/              # Mocks for testing
│   ├── db/                    # Database implementations
│   │   ├── hard_banned_implementor.go
│   │   └── soft_banned_config_implementor.go
│   ├── redis/                 # Redis implementations
│   ├── fds/                   # FDS implementations
│   └── legacy/                # Legacy DB implementations
│
├── usecase/                   # Business logic
│   ├── base_usecase.go
│   ├── health_check.go
│   ├── scan_fraud_phone_number.go
│   ├── whitelist_phone_numbers.go
│   ├── soft_banned_counter.go
│   ├── get_soft_banned.go
│   ├── get_soft_banned_by_type.go
│   ├── revoke_soft_banned.go
│   ├── set_hard_banned.go
│   ├── check_Hard_banned.go
│   ├── get_hard_banned.go
│   ├── revoke_hard_banned.go
│   ├── spam_detector.go
│   └── *_test.go              # Unit tests
│
├── transport/                 # Transport layer
│   ├── base_transport.go
│   ├── health_check.go
│   ├── scan_fraud_phone_number.go
│   ├── whitelist_phone_numbers.go
│   ├── soft_banned_counter.go
│   ├── get_soft_banned.go
│   ├── get_soft_banned_by_type.go
│   ├── revoke_soft_banned.go
│   ├── set_hard_banned.go
│   ├── check_hard_banned.go
│   ├── get_hard_banned.go
│   └── revoke_hard_banned.go
│
├── server/                    # Server setup
│   ├── grpc.go
│   ├── rest.go
│   ├── rest_option.go
│   └── metric.go
│
├── util/                      # Utilities
│   ├── server.go
│   ├── util.go
│   └── interceptor/
│       ├── base_interceptor.go
│       ├── metadata_validator.go
│       └── validate_input.go
│
├── doc/                       # Documentation
│   └── doc.plantuml
│
└── k8s/                       # Kubernetes manifests
    ├── gcp-application.yaml
    └── huawei-application.yaml
```

---

## 📊 Database Schema

### hard_banned
```sql
CREATE TABLE hard_banned (
    id BIGSERIAL PRIMARY KEY,
    identifier VARCHAR(255) NOT NULL,
    login_token VARCHAR(255) NOT NULL,
    identifier_type VARCHAR(255) NOT NULL,  -- BBID, PHONE_NUMBER, DEVICE_ID
    banned_reason_en TEXT NOT NULL,
    banned_reason_id TEXT NOT NULL,
    user_info JSONB,  
    created_at TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP,
    deleted_at TIMESTAMP NULL
);
```

### soft_banned_configs
```sql
CREATE TABLE soft_banned_configs (
    id BIGSERIAL PRIMARY KEY,
    banned_type VARCHAR(50) NOT NULL,
    city VARCHAR(100) NOT NULL,
    threshold_attempt INT NOT NULL,
    threshold_duration VARCHAR(50) NOT NULL,
    ban_duration VARCHAR(50) NOT NULL,
    response_message_id VARCHAR(100) NOT NULL,
    response_message_en TEXT NOT NULL,
    banned_reason_id VARCHAR(100) NOT NULL,
    banned_reason_en TEXT NOT NULL,
    created_at TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP,
    deleted_at TIMESTAMP NULL
);
```

---

## 🔄 Redis Key Patterns

### Counter Keys
```
COUNTER:{banned_type}:{identifier}
```
Example: `COUNTER:CANCEL_ORDER:BB00123456`

### Banned Keys
```
BANNED:{banned_type}:{identifier}
```
Example: `BANNED:WRONG_PASSWORD:BB00123456`

---

## 🔒 Security Features

### Rate Limiting per Type

| Type | Max Attempts | Time Window | Ban Duration |
|------|-------------|-------------|--------------|
| Cancel Order | 5 | 2 hours | 24 hours |
| Cancel After Dispatch | 5 | 2 hours | 24 hours |
| OTP Attempt | 3 | 24 hours | 24 hours |
| Wrong Password | 5 | 2 minutes | 5 minutes |

### Identifier Types for Hard Ban
- **BBID**: Bluebird user ID
- **PHONE_NUMBER**: User's phone number
- **DEVICE_ID**: Mobile device identifier

---

## 🔗 Related Documentation

- [[api-reference|API Reference]]
- [[dependencies|Dependencies]]
- [[02-Work/Teams/MRG/02-services/authservice/README|Auth Service]] (uses FDS for validation)
- [[02-Work/Teams/MRG/00-overview/README|MRG Team Overview]]

---

## 🏷️ Tags

#mrg #service #fds #fraud-detection #security #grpc #documentation

---

*Last Updated*: 2025-01-05  
*Generated from*: Repository analysis
