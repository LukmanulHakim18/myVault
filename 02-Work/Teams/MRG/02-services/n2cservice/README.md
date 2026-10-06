---
title: "N2C Service"
type: service-documentation
team: MRG
status: production
owner: alfian.maulana@bluebirdgroup.com
tier: P1
version: "1.0"
go_version: "1.24"
grpc_port: 6045
rest_port: 8045
slo_availability: "99.9%"
slo_latency_p99: "<200ms"
repository: git.bluebird.id/mybb-ms/n2cservice
tags: [mrg, service, n2cservice, payment, preauth, n2c, documentation]
created: 2026-04-08
updated: 2026-04-08
---

# N2C Service (Nontunai to Cash)

**Team**: MRG (Meta Reservation Gateway)
**Status**: ✅ Production
**Repository**: `git.bluebird.id/mybb-ms/n2cservice`

---

## 📋 Overview

N2C Service (Nontunai to Cash) adalah microservice yang bertanggung jawab menentukan **eligibilitas pembayaran non-tunai** (kartu kredit dan e-wallet) selama perjalanan berlangsung. Service ini dipanggil secara periodik oleh sistem untuk memeriksa apakah saldo atau pre-auth pengguna mencukupi untuk menutup ongkos perjalanan aktual (argo). Jika tidak mencukupi, service akan memicu proses **force-to-cash** secara otomatis.

### Fungsi Utama

- **Eligibility Check** — Menentukan apakah pembayaran non-tunai masih valid saat trip berjalan
- **CC Pre-auth Management** — Increment pre-auth kartu kredit jika argo melebihi jumlah pre-auth sebelumnya
- **E-wallet Balance Check** — Memeriksa saldo e-wallet dan membandingkan dengan argo aktual
- **Force-to-Cash** — Memaksa perubahan metode pembayaran ke tunai jika non-tunai gagal
- **Push Notification** — Mengirim notifikasi ke user saat saldo tidak cukup atau terjadi force-to-cash

---

## 🏷️ Service Identity

| Field | Value |
|-------|-------|
| **Service Name** | `n2c-service` |
| **Repository** | `git.bluebird.id/mybb-ms/n2cservice` |
| **Team** | MRG (Meta Reservation Gateway) |
| **Owner / PIC** | Alfian Maulana |
| **Contact** | alfian.maulana@bluebirdgroup.com |
| **Tier / Criticality** | P1 |
| **Status** | Production |
| **Version** | 1.0 |
| **Go Version** | 1.24 |

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
| Language | Go 1.24 |
| Protocol | gRPC + REST (gRPC-Gateway) |
| Database | - |
| Cache | Redis |
| Message Broker | RabbitMQ + Google PubSub |
| Monitoring | Elastic APM, Prometheus |
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
| **gRPC Port** | 6045 |
| **REST Port** | 8045 |

### CI/CD Pipeline

| Branch | Environment | Namespace |
|--------|-------------|-----------|
| `development` | Development | `microservices` |
| `staging` | Staging | `microservices` |
| `regress` | Regression | `microservices-regress` |
| tag `v*` | Production | _TODO: konfirmasi_ |

**ArgoCD Project**: `mybluebird`
**ArgoCD Repo**: `git@gitssh.bluebird.id:argocd/mybluebird.git`
**Sync Policy**: Auto-prune, self-heal

---

## 🔑 Konsep Utama

### 1. Eligibility per Payment Type
Service membedakan logika eligibility berdasarkan tipe pembayaran:
- **Fixed Fare** → selalu eligible, tidak perlu cek
- **Credit Card (CC)** → cek pre-auth, increment jika argo melebihi jumlah pre-auth
- **E-wallet** → cek saldo real-time vs argo aktual
- **Cash / lainnya** → selalu eligible

### 2. CC Pre-auth Increment
Jika argo melebihi jumlah pre-auth yang ada, service akan:
1. Cancel pre-auth lama (via `PreAuthCancellation` atau `CancelTx` untuk legacy)
2. Buat pre-auth baru dengan jumlah = `last_amount + cycle_charge`
3. Jika re-preauth gagal → force-to-cash

### 3. E-wallet Balance Tracking
Saldo e-wallet di-cache di Redis (`BalanceNotif`) untuk mendeteksi apakah user sudah top-up. Jika saldo tidak pernah naik dan perjalanan berakhir → force-to-cash.

### 4. Force-to-Cash
Mekanisme fallback ketika pembayaran non-tunai tidak bisa dilanjutkan:
- Update locked fare ke 0 via TaxiOrderProcessor
- Kirim push notification ke user
- Tandai order sebagai `force_to_cash = true`

### 5. Order Detail Caching
Detail order dari TaxiOrderProcessor di-cache di Redis untuk mengurangi load gRPC call berulang selama trip berlangsung.

---

## 🔌 Dependencies

### Internal Services

| Service | Protocol | Library | Purpose |
|---------|----------|---------|---------|
| **TaxiOrderProcessor** | gRPC | `grpc-client` v1.7.7 | Get order detail, pre-auth info, update locked fare |
| **PaymentProcessor** | gRPC | `grpc-client` v1.7.7 | Re-preauth, cancel preauth, get e-wallet balance, cancel transaction |
| **OrderQuery** | gRPC | `grpc-client` v1.7.7 | Cek eligibility preauth increment, get order info |
| **ConfigService** | gRPC | `grpc-client` v1.7.7 | Get preauth config (cycle charge, amount) per product type |
| **Message Broker** | RabbitMQ / PubSub | `commonmessaging` v0.1.40 | Publish notifikasi ke user (force-to-cash, insufficient balance, re-preauth) |

### Infrastructure

| Component | Purpose |
|-----------|---------|
| **Redis** | Cache order detail, balance notification state per job_id |
| **RabbitMQ** | Message broker untuk push notification |
| **Google PubSub** | Alternatif message broker untuk notification |

### Client Libraries

| Library | Version | Purpose |
|---------|---------|---------|
| `grpc-client` | v1.7.7 | Internal gRPC client (TaxiOrderProcessor, PaymentProcessor, OrderQuery, ConfigService) |
| `commonmessaging` | v0.1.40 | Push notification messaging |
| `aphrodite` | v1.10.16 | Internal framework (config, logger, constants) |
| `bluebird-chassis` | v0.3.2 | Internal chassis framework |
| `watermill` | v1.4.2 | Message broker abstraction (RabbitMQ, PubSub, Kafka) |
| `prometheus/client_golang` | v1.23.2 | Metrics exposure |


---

## 📡 API Contracts

### gRPC Service

**Package**: `n2cservice`
**Proto File**: `contract/n2cservice.proto`
**Ports**: gRPC `6045`, REST `8045`

### Methods

| Method | HTTP | REST Path | Interceptors | Description |
|--------|------|-----------|--------------|-------------|
| `HealthCheck` | GET | `/` | - | Health check endpoint |
| `EligibilityByJobId` | POST | `/v1/payment/eligibility` | - | Cek eligibilitas pembayaran non-tunai berdasarkan order ID |

---

## 🔄 Business Flows

Lihat detail di: [[flows|N2C Flows]]

### Ringkasan Flow

```
EligibilityByJobId
├── Fixed Fare → ✅ Eligible
├── Credit Card
│   ├── Not preauth eligible → ✅ Eligible
│   ├── Non-preauth card → ✅ Eligible
│   ├── Argo ≤ preauth amount → ✅ Eligible
│   └── Argo > preauth amount
│       ├── Increment preauth berhasil → ✅ Eligible
│       └── Increment preauth gagal → ❌ Force-to-Cash
├── E-wallet
│   ├── Saldo cukup → ✅ Eligible
│   ├── Saldo tidak cukup (mid-trip) → ⚠️ Notifikasi top-up
│   └── Saldo tidak cukup (end-trip) → ❌ Force-to-Cash
└── Cash / lainnya → ✅ Eligible
```

---

## ⚙️ Configuration

### App

| Config Key | Default | Keterangan |
|------------|---------|------------|
| `app_name` | `n2cservice` | Nama aplikasi |
| `grpc_port` | `6045` | gRPC port |
| `rest_port` | `8045` | REST port |
| `log_level` | `info` | Log level |
| `pod_name` | `n2c-service` | Nama pod K8s |
| `check_healthy_repo` | `true` | Cek health dependency saat startup |
| `pubsub_env` | `""` | Environment PubSub |

### Service Connections

| Config Key | Default | Keterangan |
|------------|---------|------------|
| `paymentprocessor_host` | `""` | PaymentProcessor gRPC host |
| `paymentprocessor_port` | `0` | PaymentProcessor gRPC port |
| `configservice_host` | `""` | ConfigService gRPC host |
| `configservice_port` | `0` | ConfigService gRPC port |
| `taxiorderprocessor_host` | `""` | TaxiOrderProcessor gRPC host |
| `taxiorderprocessor_port` | `0` | TaxiOrderProcessor gRPC port |
| `orderquery_host` | `""` | OrderQuery gRPC host |
| `orderquery_port` | `0` | OrderQuery gRPC port |

### Infrastructure

| Config Key | Default | Keterangan |
|------------|---------|------------|
| `rabbitmq_client_uri` | `""` | RabbitMQ connection URI |
| `redis_url` | `""` | Redis connection URL |

---

## 🔒 Security Measures

### Kubernetes Secrets
| Secret | Isi |
|--------|-----|
| _TODO: konfirmasi dari ArgoCD/Helm values_ | Redis password, RabbitMQ credentials |

---

## 🚨 Runbook & Incident

| Resource | Value |
|----------|-------|
| **On-call PIC** | Alfian Maulana (alfian.maulana@bluebirdgroup.com) |
| **Runbook** | _TODO: tambahkan link runbook_ |
| **APM Service** | `n2cservice` |
| **Monitoring** | Elastic APM + Prometheus |

### Common Issues

| Issue | Kemungkinan Penyebab | Langkah Awal |
|-------|---------------------|--------------|
| `EligibilityByJobId` selalu return force-to-cash | TaxiOrderProcessor down / order not found | Cek gRPC connectivity ke TaxiOrderProcessor |
| Pre-auth increment gagal massal | PaymentProcessor down atau limit exceeded | Cek PaymentProcessor health + log `error.RePreAuth` |
| Notifikasi tidak terkirim | RabbitMQ / PubSub down | Cek MessageBroker health check |
| Redis cache miss berulang | Redis down atau TTL terlalu pendek | Cek Redis connection + `error.GetOrderDetail` |

---

## 📂 Project Structure

```
n2cservice/
├── main.go
├── go.mod
├── Dockerfile
├── Jenkinsfile
├── config/              # App configuration
│   └── default.go
├── constants/           # Constants (trip status, payment method, dll)
├── contract/            # Proto + generated gRPC + REST mapping
│   ├── n2cservice.proto
│   ├── n2cservice.yaml
│   └── *.pb.go
├── model/               # Domain models (BalanceNotif, dll)
├── pkg/                 # Internal packages
│   └── notification/    # Push notification helper
├── repository/          # Data access layer
│   ├── repoiface/       # Interfaces (6 dependencies)
│   ├── configservice/
│   ├── messagebroker/
│   ├── orderquery/
│   ├── paymentprocessor/
│   ├── redis/
│   └── taxiorderprocessor/
├── usecase/             # Business logic
│   ├── eligibility_by_job_id.go  ← Core business logic
│   └── health_check.go
├── transport/           # gRPC handlers
├── server/              # gRPC + REST server setup
├── util/                # Error handling, utilities
└── k8s/
    └── huawei-application.yaml
```

---

## 🔗 Related Documentation

- [[api-reference|API Reference]]
- [[dependencies|Dependencies]]
- [[flows|Business Flows]]
- [[02-Work/Teams/MRG/00-overview/README|MRG Team Overview]]

---

#mrg #service #n2cservice #payment #preauth #n2c #documentation

---

*Last Updated*: 2026-04-08
*Generated from*: Repository analysis — D:\code\go\mybb-ms\n2cservice
