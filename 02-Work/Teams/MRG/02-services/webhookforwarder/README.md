---
tags:
  - mrg
  - service
  - webhookforwarder
  - webhook
  - callback
  - grpc
  - documentation
team: MRG
type: service-documentation
title: Webhook Forwarder
status: production
created: '2026-02-27'
updated: '2026-02-27'
grpc_port: 6039
rest_port: 8039
repository: git.bluebird.id/mybb-ms/webhookforwarder
tech_stack:
  - go
  - grpc
  - rabbitmq
  - pubsub
---

# Webhook Forwarder

**Team**: MRG (Meta Reservation Gateway)  
**Status**: ✅ Production  
**Repository**: `git.bluebird.id/mybb-ms/webhookforwarder`

---

## 📋 Overview

Webhook Forwarder adalah microservice yang berfungsi sebagai **single entry point** untuk semua incoming callback/webhook dari external systems ke ekosistem MyBluebird. Service ini menerima callback, memvalidasi, lalu mem-publish ke message broker (RabbitMQ/PubSub) untuk diproses secara async oleh service terkait.

### Fungsi Utama

- **Order Callback** - Terima callback order state changes dari BBD (web & mobile)
- **Payment Callback** - Terima callback pre-auth CC, payment refund status, NicePay
- **Cititrans Callback** - Terima callback refund status dari Cititrans Order Processor
- **Document Callback** - Terima callback generate document dari Document Generator
- **Notification Callback** - Terima callback status notifikasi (token expired, dll)

### Pola Arsitektur

Semua endpoint mengikuti pola yang sama:

```
External System → Webhook Forwarder (validate + auth) → Publish to Broker → Consumer Service
```

Service ini **tidak melakukan business logic** — hanya menerima, memvalidasi, dan meneruskan ke broker.

---

## 🛠️ Tech Stack

| Component | Technology |
|-----------|------------|
| Language | Go 1.24 |
| Protocol | gRPC + REST (gRPC-Gateway) |
| Message Broker | RabbitMQ (primary) |
| Messaging Library | PubSub (commonmessaging) |
| Monitoring | Elastic APM, Prometheus |
| Container | Docker, Kubernetes |

---

## 🔌 Dependencies

### Internal Services

| Service | Purpose | Protocol |
|---------|---------|----------|
| **Old Payment Processor** | Forward pre-auth CC callback | HTTP REST |
| **Cititrans Order Processor** | Refund status reference | HTTP REST |

### Infrastructure

| Component | Purpose |
|-----------|---------|
| **RabbitMQ** | Primary message broker untuk publish callback events |

### Repository Structure

```go
type Repository struct {
    CititransOrderProcessor repoiface.CititransOrderProcessor
    Publisher               repoiface.Publisher
    OldPaymentProc          repoiface.OldPaymentProc
}
```

---

## 📡 API Contracts

### gRPC Service

**Package**: `webhookforwarder`  
**Proto File**: `contract/webhook_forwarder.proto`  
**Ports**: gRPC `6039`, REST `8039`

### Methods Overview

| Method | Source | Destination (Broker Topic) | Description |
|--------|--------|---------------------------|-------------|
| `HealthCheck` | - | - | Health check |
| `OrderCallbackWeb` | BBD/bb-order | `TOPIC_ORDER_CALLBACK` | Order state changes (web) |
| `OrderCallbackMobile` | BBD/bb-order | `TOPIC_ORDER_CALLBACK` | Order state changes (mobile) |
| `PreAuthCallback` | UPG | `TOPIC_PRE_AUTH_CALLBACK` | CC pre-auth result |
| `PaymentRefundStatus` | UPG | `TopicPaymentRefundStatus` | Payment refund status update |
| `PaymentNicepayCallback` | NicePay | - | NicePay payment callback |
| `CititransRefundStatus` | Cititrans | `TopicCititransRefundStatus` | Cititrans refund status |
| `GenerateDocumentCallback` | Document Generator | - | Document generation result (legacy) |
| `DocumentGeneratorCallback` | Document Generator | - | Document generation result (v2) |
| `ReceiveNotificationCallback` | Notification Provider | `TopicDeleteBBOneUserRecipientId` | Push notification status (token expired) |

---

## 🔄 Flow

### Order Callback Flow

```
BBD Dispatch → POST /order-callback-web (REST)
    ↓
Webhook Forwarder validates AUTH_KEY
    ↓
Publish to RabbitMQ: TOPIC_ORDER_CALLBACK
    ↓
Consumer (mybb-order-processing-go) processes event
```

### Pre-Auth Callback Flow

```
UPG → PreAuthCallback (gRPC/REST)
    ↓
Webhook Forwarder → Forward to Old Payment Processor (HTTP)
    ↓
If success → Publish to RabbitMQ: TOPIC_PRE_AUTH_CALLBACK
    ↓
Consumer (paymentprocessor) updates pre-auth status
```

### Notification Callback Flow

```
Push Notification Provider → ReceiveNotificationCallback
    ↓
Check status == TOKEN_EXPIRED
    ↓
Publish to PubSub: TopicDeleteBBOneUserRecipientId
    ↓
Consumer (notificationcenter) removes expired token
```

---

## ⚙️ Configuration

### Environment Variables

```env
# Application
APP_NAME=webhook-forwarder
GRPC_PORT=6039
REST_PORT=8039
LOG_LEVEL=info
LOG_DIRECTORY=

# Security
AUTH_KEY=  # Key untuk validasi incoming webhook

# Cititrans Order Processor
CITITRANS_ORDER_PROCESSOR_HOST=http://dev-mybb-api.gcp.bluebird.id/cititrans-order-processor

# Old Payment Processor
OLDPAYMENTPROC_HOST=

# Message Broker
RABBITMQ_CLIENT_URI=
RABBITMQ_GOROOSTER_TOPIC_NAME=

# Observability
ELASTIC_APM_SERVER_URL=
ELASTIC_APM_SERVICE_NAME=webhook-forwarder
ELASTIC_APM_ENVIRONMENT=production
```

---

## 📂 Project Structure

```
webhookforwarder/
├── main.go
├── go.mod
├── Dockerfile
├── Jenkinsfile
│
├── config/
│   ├── default.go             # Default config (grpc:6039, rest:8039)
│   ├── logger/
│   └── repository/
│
├── constant/
│   ├── constant.go
│   ├── success_code.go        # Success response codes
│   ├── receive_notification_callback_status.go
│   └── cititirant_update_order_file_url.go
│
├── contract/
│   ├── webhook_forwarder.proto
│   ├── webhook_forwarder.pb.go
│   ├── webhook_forwarder_grpc.pb.go
│   ├── webhook_forwarder.pb.gw.go
│   ├── webhook_forwarder.swagger.json
│   └── input_validator.go
│
├── model/
│   ├── event.go
│   ├── job_api.go
│   └── cititrans_order_processor.go
│
├── repository/
│   ├── base_repository.go
│   ├── repoiface/
│   ├── repomock/
│   ├── cititransorderprocessor/
│   ├── oldpaymentproc/
│   └── publisher/             # RabbitMQ publisher
│
├── usecase/
│   ├── base_usecase.go
│   ├── health_check.go
│   ├── order_callback_web.go
│   ├── order_callback_mobile.go
│   ├── pre_auth_callback.go
│   ├── payment_refund_status.go
│   ├── payment_nicepay_callback.go
│   ├── cititrans_refund_status.go
│   ├── generate_document_callback.go
│   ├── document_generator_callback.go
│   └── receive_notification_callback.go
│
├── transport/
│   └── ... (one file per endpoint)
│
├── server/
│   ├── grpc.go
│   ├── rest.go
│   ├── rest_option.go
│   └── metric.go
│
├── util/
│   ├── errors/
│   ├── interceptor/
│   └── server.go
│
└── k8s/
    └── huawei-application.yaml
```

---

## 🔗 Related Documentation

- [[02-Work/Teams/MRG/00-service-catalog|MRG Service Catalog]]
- [[02-Work/Teams/MRG/02-services/paymentprocessor/README|Payment Processor]] (consumer of pre-auth callback)
- [[02-Work/Teams/MRG/02-services/notificationcenter/README|Notification Center]] (consumer of notification callback)

---

#mrg #service #webhookforwarder #webhook #callback #grpc #documentation

*Last Updated*: 2026-02-27  
*Generated from*: Repository analysis at `D:\code\go\mybb-ms\webhookforwarder`
