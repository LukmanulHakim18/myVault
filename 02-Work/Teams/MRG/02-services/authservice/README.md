---
tags:
  - mrg
  - service
  - authservice
  - authentication
  - jwt
  - otp
  - security
  - grpc
  - documentation
team: MRG
type: service-documentation
title: Auth Service
status: production
created: '2025-01-05'
updated: '2025-01-05'
grpc_port: 6017
rest_port: 8017
repository: git.bluebird.id/mybb-ms/authservice
tech_stack:
  - go
  - grpc
  - redis
  - pubsub
  - jwt
---
# Auth Service

**Team**: MRG (Meta Reservation Gateway)  
**Status**: ✅ Production  
**Repository**: `git.bluebird.id/mybb-ms/authservice`

---

## 📋 Overview

MyBB AuthService adalah microservice yang bertanggung jawab untuk mengelola autentikasi dan otorisasi pengguna dalam ekosistem MyBluebird. Service ini menyediakan mekanisme keamanan menggunakan JWT (JSON Web Token) untuk access token dan refresh token yang di-hash untuk keamanan maksimal.

### Fungsi Utama

- **User Authentication** - Login dengan phone/email dan password
- **Token Management** - Generate, validate, refresh, dan revoke tokens
- **OTP Verification** - Multi-channel OTP (WhatsApp, SMS, Email)
- **User Registration** - Validasi user, OTP verification, create account
- **Password Management** - Change password, forgot password, reset password
- **Session Security** - Session token management, rate limiting, fraud detection

---

## 🛠️ Tech Stack

| Component | Technology |
|-----------|------------|
| Language | Go 1.24 |
| Protocol | gRPC + REST (gRPC-Gateway) |
| Database | - |
| Cache | Redis |
| Message Queue | Google PubSub |
| Monitoring | Elastic APM |
| Security | JWT (RSA), bcrypt |
| Container | Docker, Kubernetes |

---

## 🔑 Konsep Utama

### 1. Dual Token System
- **Access Token**: JWT token dengan masa aktif pendek (2 jam) untuk autentikasi request
- **Refresh Token**: Hash token dengan masa aktif lebih panjang (16 jam) untuk mendapatkan access token baru

### 2. Multi-Channel OTP
- WhatsApp
- SMS  
- Email

### 3. Security Features
- Password encryption menggunakan bcrypt
- OTP dengan expiration time (5 menit default)
- Rate limiting untuk prevent brute force
- Wrong password counter dengan auto-ban mechanism (5 attempts → 24h ban)
- Token blacklisting untuk logout

---

## 🔌 Dependencies

### Internal Services

| Service | Purpose | Protocol |
|---------|---------|----------|
| **User Service** | CRUD user data, profile management, password validation | gRPC |
| **FDS Service** | Fraud Detection System untuk validasi phone number | gRPC |
| **Notification Center** | Kirim OTP via WhatsApp/SMS/Email | Google PubSub |
| **Legacy System** | Backward compatibility dengan sistem lama | HTTP REST |

### Client Libraries

| Library | Version | Purpose |
|---------|---------|---------|
| `fdsclient` | v0.0.5 | FDS integration |
| `userclient` | v0.0.11 | User service integration |
| `commonmessaging` | v0.0.19 | PubSub messaging |

### Infrastructure

| Component | Purpose |
|-----------|---------|
| **Redis** | Session storage, token cache, OTP cache, rate limiting |
| **Redis Stream** | Event streaming |

### Repository Structure

```go
type Repository struct {
    Redis              repoiface.RedisCache
    TokenGenerator     repoiface.TokenGenerator
    FDS                repoiface.FDS
    User               repoiface.User
    NotificationCenter repoiface.Notification
    Legacy             repoiface.Legacy
    RedisStream        repoiface.RedisStream
}
```

---

## 📡 API Contracts

### gRPC Service

**Package**: `authservice`  
**Proto File**: `contract/authservice.proto`  
**Ports**: gRPC `6017`, REST `8017`, Swagger `9017`

### Methods Overview

#### Token Management

| Method | Description | Interceptors |
|--------|-------------|--------------|
| `CreateToken` | Create access & refresh token | InputValidation |
| `RefreshToken` | Refresh expired access token | InputValidation, MetadataAfterLoginValidation |
| `ValidateToken` | Validate access token | InputValidation |

#### User Validation & OTP

| Method | Description | Interceptors |
|--------|-------------|--------------|
| `ValidateUser` | Validate user (phone/email) | InputValidation, MetadataValidation, FormatPhoneNumber |
| `ValidateUserWithProviderList` | Validate user with OTP options | InputValidation, MetadataValidation, FormatPhoneNumber |
| `SendOtp` | Send OTP via channel | OTT, InputValidation, MetadataValidation |
| `ValidateOTP` | Validate OTP code | InputValidation, MetadataValidation |

#### Authentication

| Method | Description | Interceptors |
|--------|-------------|--------------|
| `RegisterUser` | Register new user | InputValidation, MetadataValidation |
| `Login` | Login with password | InputValidation, MetadataValidation |
| `Logout` | Logout (revoke tokens) | InputValidation, MetadataAfterLoginValidation |
| `RevokeAllRefreshToken` | Logout from all devices | MetadataAfterLoginValidation |

#### Password Management

| Method | Description | Interceptors |
|--------|-------------|--------------|
| `ChangePassword` | Change password (logged in) | InputValidation, MetadataAfterLoginValidation |
| `ForgotPassword` | Request password reset | InputValidation, MetadataAfterLoginValidation |
| `ResetPassword` | Reset password with token | InputValidation |

#### Utility

| Method | Description | Interceptors |
|--------|-------------|--------------|
| `HealthCheck` | Health check | - |
| `GetTimeSync` | Get server time | - |
| `MigrateToken` | Migrate legacy token | InputValidation |
| `EncryptPass` | Encrypt password (internal) | - |
| `DecryptPass` | Decrypt password (internal) | - |

---

## 🔄 Authentication Flows

### New User Registration Flow
```
ValidateUser → SendOTP → ValidateOTP → RegisterUser → Auto Login
```

### Existing User Login Flow
```
ValidateUser → Login → Get Tokens
```

### Token Refresh Flow
```
Access Token Expired → RefreshToken → New Access Token
```

### Forgot Password Flow
```
ValidateUser → ForgotPassword → Email Sent → ResetPassword
```

Lihat detail lengkap di: [[authentication-flows|Authentication Flows]]

---

## ⚙️ Configuration

### Environment Variables

```env
# Application
APP_NAME=Mybb-Auth-Service
GRPC_PORT=6017
REST_PORT=8017
SWAGGER_PORT=9017

# JWT
JWT_PRIVATE_KEY_PATH=cert/private_key.pem
DURATION_ACCESS_TOKEN_ALIVE=2h
DURATION_REFRESH_TOKEN_ALIVE=16h
SALT_REFRESH_TOKEN=mybb-auth

# Redis
REDIS_HOST=172.26.11.40
REDIS_PORT=6379
REDIS_DATABASE=11
REDIS_PASSWORD=

# OTP Settings
SEND_OTP_MAX_ATTEMPT=5
RESEND_OTP_AFTER=7s
VALID_OTP_DURATION=24h

# One Time Token (OTT) - Password Encryption
OTT_FEATURE=false
OTT_TOLERANCE=30s
OTT_KEY=

# FDS Service
FDS_HOST=fds-service
FDS_PORT=6001

# User Service
USER_HOST=user-service
USER_PORT=6015

# Notification (PubSub)
NOTIFICATION_CENTER_PROJECT_ID=mybluebird
GOOGLE_APPLICATION_CREDENTIALS=cert/credentials.json
PUBSUB_ENV=stg

# APM
ELASTIC_APM_SERVER_URL=
ELASTIC_APM_SERVICE_NAME=mybb-auth-service
ELASTIC_APM_ENVIRONMENT=DEV
```

---

## 📂 Project Structure

```
authservice/
├── main.go                    # Entry point
├── go.mod                     # Dependencies
├── Dockerfile                 # Container build
├── Jenkinsfile                # CI/CD pipeline
│
├── cert/                      # Certificates
│   ├── private.pem            # JWT private key (RSA)
│   ├── public.pem             # JWT public key
│   └── login_partner.json     # Partner credentials
│
├── config/                    # Configuration
│   ├── default.go
│   ├── logger/
│   └── repository/
│
├── constant/                  # Constants
│   ├── constant.go
│   ├── custom_header.go
│   └── token_prefix.go
│
├── contract/                  # API contracts
│   ├── authservice.proto
│   ├── authservice.pb.go
│   ├── authservice_grpc.pb.go
│   ├── authservice.pb.gw.go
│   ├── authservice.swagger.json
│   ├── mapper.go
│   └── validate_input.go
│
├── model/                     # Domain models
│   └── dto_repo_redis.go
│
├── repository/                # Data access layer
│   ├── base_repository.go
│   ├── repoiface/             # Interfaces
│   ├── repomock/              # Mocks
│   ├── redis/                 # Redis implementation
│   ├── redisstream/           # Redis Stream
│   ├── fds/                   # FDS client
│   ├── user/                  # User service client
│   ├── notification/          # Notification client
│   ├── legacy/                # Legacy system client
│   └── tokengen/              # Token generator
│
├── usecase/                   # Business logic (with tests)
│   ├── base_usecase.go
│   ├── login.go
│   ├── register_user.go
│   ├── create_token.go
│   ├── refresh_token.go
│   ├── validate_access_token.go
│   ├── send_otp.go
│   ├── validate_otp.go
│   ├── change_password.go
│   ├── forgot_password.go
│   ├── reset_password.go
│   └── ... (+ test files)
│
├── transport/                 # Transport layer
│   ├── base_transport.go
│   └── ... (handlers)
│
├── server/                    # Server setup
│   ├── grpc.go
│   ├── rest.go
│   └── swagger.go
│
├── util/                      # Utilities
│   ├── interceptor/
│   ├── error/
│   ├── key_generator.go
│   └── util.go
│
├── doc/                       # Documentation
│   └── flow_sequence_diagram.svg
│
├── _doc/                      # Detailed docs
│   ├── 01-PROJECT-OVERVIEW.md
│   ├── 02-ARCHITECTURE.md
│   ├── 03-PROJECT-STRUCTURE.md
│   ├── 04-FEATURES.md
│   ├── 05-AUTHENTICATION-FLOWS.md
│   ├── 06-API-ENDPOINTS.md
│   ├── 07-DEPENDENCIES.md
│   ├── 08-CONFIGURATION.md
│   └── 09-DEPLOYMENT.md
│
└── k8s/                       # Kubernetes manifests
    └── huawei-application.yaml
```

---

## 🔒 Security Measures

### Rate Limiting
- **OTP sends**: Max 5 per session, 60s cooldown
- **Login attempts**: Max 5 wrong passwords, then 24h ban

### Token Security
- **Access tokens**: 2 hours TTL, signed with RSA private key
- **Refresh tokens**: 16 hours TTL, hash-based
- **Session tokens**: Time-limited, single-use
- **Reset tokens**: 1-hour expiration, one-time use

### Fraud Prevention
- **FDS integration**: Phone number fraud check
- **Attempt counters**: Track suspicious patterns
- **Blacklisting**: Invalid tokens cannot be reused

---

## 🔗 Related Documentation

- [[authentication-flows|Authentication Flows Detail]]
- [[api-reference|API Reference]]
- [[dependencies|Dependencies]]
- [[02-Work/Teams/MRG/00-overview/README|MRG Team Overview]]

---

## 🏷️ Tags

#mrg #service #authservice #authentication #jwt #otp #security #grpc #documentation

---

*Last Updated*: 2025-01-05  
*Generated from*: Repository analysis + existing _doc folder
