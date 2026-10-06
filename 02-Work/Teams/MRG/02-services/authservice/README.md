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
tier: P1
created: 2025-01-05
updated: 2026-04-07
grpc_port: 6017
rest_port: 8017
repository: git.bluebird.id/mybb-ms/authservice
tech_stack:
  - go
  - grpc
  - redis
  - pubsub
  - jwt
version: "1.0"
go_version: "1.21"
owner: alfian.maulana@bluebirdgroup.com
slo_availability: 99.9%
slo_latency_p99: 200ms
---
# Auth Service

**Team**: MRG (Meta Reservation Gateway)
**Status**: ✅ Production
**Repository**: `git.bluebird.id/mybb-ms/authservice`
**Branch utama**: `master-huawei`

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

## 🏷️ Service Identity

| Field | Value |
|-------|-------|
| **Service Name** | `auth-service` |
| **Repository** | `git.bluebird.id/mybb-ms/authservice` |
| **Team** | MRG (Meta Reservation Gateway) |
| **Owner / PIC** | Alfian Maulana |
| **Contact** | alfian.maulana@bluebirdgroup.com |
| **Tier / Criticality** | P1 |
| **Status** | Production |
| **Version** | 1.0 |
| **Go Version** | 1.21 |

---

## 📊 SLO (Service Level Objectives)

| Metric | Target |
|--------|--------|
| **Availability** | 99.9% |
| **Latency p99** | < 200ms |
| **Error Rate** | < 0.1% |

---

## 🛠️ Tech Stack

| Component | Technology |
|-----------|------------|
| Language | Go 1.21 |
| Protocol | gRPC + REST (gRPC-Gateway) |
| Cache | Redis |
| Message Queue | Google PubSub |
| Monitoring | Elastic APM |
| Security | JWT (RSA), bcrypt |
| Container | Docker, Kubernetes |
| CI/CD | Jenkins → ArgoCD |

---

## 🚀 Deployment

### Kubernetes

| Config | Value |
|--------|-------|
| **Namespace (prod/stg/dev)** | `microservices` |
| **Namespace (regress)** | `microservices-regress` |
| **Cluster** | `cce_huawei_mybluebird_dev_bluebirdgroup_mybluebird_dev-internal-cluster` |
| **Cloud** | Huawei Cloud |
| **Replicas** | 2 |
| **Resource Limits** | Not set |
| **gRPC Port** | 6017 |
| **REST Port** | 8017 |

### CI/CD Pipeline

| Stage | Branch | Keterangan |
|-------|--------|------------|
| Deploy to development | `development` | Auto deploy ke Huawei dev cluster |
| Deploy to staging | `staging` | Auto deploy ke Huawei dev cluster |
| Deploy to regress | `regress` | Auto deploy ke namespace `microservices-regress` |
| Deploy to production | tag `v*` | Manual via ArgoCD |

**ArgoCD Project**: `mybluebird`
**ArgoCD Repo**: `git@gitssh.bluebird.id:argocd/mybluebird.git`

### Health Check

```yaml
readinessProbe:
  grpc_health_probe: :6017
  initialDelaySeconds: 5
  periodSeconds: 3

livenessProbe:
  grpc_health_probe: :6017
  initialDelaySeconds: 30
  periodSeconds: 10
```

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
| `aphrodite` | v1.7.22 | Internal framework |

### Infrastructure

| Component | Purpose |
|-----------|---------|
| **Redis** | Session storage, token cache, OTP cache, rate limiting |
| **Redis Stream** | Event streaming |
| **Google PubSub** | Async notification messaging |

---

## 📡 API Contracts

### gRPC Service

**Package**: `authservice`
**Proto File**: `contract/authservice.proto`
**Ports**: gRPC `6017`, REST `8017`, Swagger `9017`

### Methods Overview

#### Token Management

| Method | Description |
|--------|-------------|
| `CreateToken` | Create access & refresh token |
| `RefreshToken` | Refresh expired access token |
| `ValidateToken` | Validate access token |

#### User Validation & OTP

| Method | Description |
|--------|-------------|
| `ValidateUser` | Validate user (phone/email) |
| `ValidateUserWithProviderList` | Validate user with OTP options |
| `SendOtp` | Send OTP via channel |
| `ValidateOTP` | Validate OTP code |

#### Authentication

| Method | Description |
|--------|-------------|
| `RegisterUser` | Register new user |
| `Login` | Login with password |
| `Logout` | Logout (revoke tokens) |
| `RevokeAllRefreshToken` | Logout from all devices |

#### Password Management

| Method | Description |
|--------|-------------|
| `ChangePassword` | Change password (logged in) |
| `ForgotPassword` | Request password reset |
| `ResetPassword` | Reset password with token |

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

### Token & Session

| Setting | Env Variable | Default |
|---------|--------------|---------|
| Access Token TTL | `DURATION_ACCESS_TOKEN_ALIVE` | 2h |
| Refresh Token TTL | `DURATION_REFRESH_TOKEN_ALIVE` | 16h |
| Refresh Token Salt | `SALT_REFRESH_TOKEN` | mybb-auth |
| Private Key Path | `JWT_PRIVATE_KEY_PATH` | cert/private_key.pem |

### OTP

| Setting | Env Variable | Default |
|---------|--------------|---------|
| Max OTP Attempts | `SEND_OTP_MAX_ATTEMPT` | 5 |
| Resend Cooldown | `RESEND_OTP_AFTER` | 7s |
| OTP Valid Duration | `VALID_OTP_DURATION` | 24h |

### One Time Token (OTT)

| Setting | Env Variable | Default |
|---------|--------------|---------|
| OTT Feature flag | `OTT_FEATURE` | false |
| OTT Tolerance | `OTT_TOLERANCE` | 30s |

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
- **FDS integration**: Phone number fraud check on ValidateUser
- **Attempt counters**: Track suspicious patterns
- **Blacklisting**: Invalid tokens cannot be reused

### Secrets di Kubernetes
| Secret | Isi |
|--------|-----|
| `secret-auth-service` | `REDIS_PASSWORD` |
| `secret-jwt-private-key` | RSA private key |
| `secret-jwt-public-key` | RSA public key |
| `secret-google-pubsub` | GCP service account credentials |

---

## 🚨 Runbook & Incident

| Resource | Link |
|----------|------|
| **On-call PIC** | Alfian Maulana (alfian.maulana@bluebirdgroup.com) |
| **Runbook** | _TODO: tambahkan link runbook_ |
| **Kibana / APM** | `ELASTIC_APM_SERVICE_NAME=mybb-auth-service` |
| **Monitoring** | Elastic APM + Grafana |

### Common Issues
- **Token invalid / expired**: Cek Redis connection, pastikan TTL config benar
- **OTP tidak terkirim**: Cek PubSub connectivity dan Notification Center
- **Login gagal massal**: Cek User Service availability via gRPC

---

## 📂 Project Structure

```
authservice/
├── main.go
├── go.mod
├── Dockerfile
├── Jenkinsfile
├── cert/                   # JWT keys & partner credentials
├── config/                 # App configuration
├── constant/               # Constants
├── contract/               # Proto + generated gRPC files
├── model/                  # Domain models
├── repository/             # Data access layer
│   ├── repoiface/          # Interfaces
│   ├── repomock/           # Mocks
│   ├── redis/
│   ├── redisstream/
│   ├── fds/
│   ├── user/
│   ├── notification/
│   ├── legacy/
│   └── tokengen/
├── usecase/                # Business logic + unit tests
├── transport/              # Transport layer (gRPC handlers)
├── server/                 # gRPC + REST + Swagger server setup
├── util/                   # Interceptors, error, utilities
├── k8s/                    # Kubernetes manifests
│   ├── deployment.yaml
│   ├── service.yaml
│   └── huawei-application.yaml
└── doc/                    # Flow diagrams
```

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

*Last Updated*: 2026-04-07
*Updated by*: Claude (dari repo analysis)
