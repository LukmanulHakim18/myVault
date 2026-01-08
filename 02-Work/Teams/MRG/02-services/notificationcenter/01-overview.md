# Notification Center Service - Overview

## Service Information
- **Service Name**: notificationcenter
- **Team**: MRG (Meta Reservation Gateway)
- **Domain**: Notification & Communication
- **Repository**: `git.bluebird.id/mybb-ms/notificationcenter`
- **Language**: Go
- **Architecture Pattern**: Layered Architecture + Strategy Pattern
- **Local Path**: `D:\code\go\mybb-ms\notificationcenter`

## Purpose
Service ini berfungsi sebagai **Centralized Notification Hub** untuk seluruh platform Bluebird, menangani berbagai channel komunikasi untuk MyBB App, Web Reservation, Corporate Booking, dan sistem internal lainnya.

## Core Capabilities

### 1. Multi-Channel Notification
- **Email**: Transactional emails (order confirmation, invoice, receipt, OTP)
- **SMS**: OTP, order updates, driver info
- **Push Notification**: In-app notifications via FCM (Firebase Cloud Messaging)
- **WhatsApp**: OTP dan order notifications

### 2. Multi-Protocol Support
- **gRPC**: Primary protocol untuk inter-service communication
- **REST**: HTTP API via gRPC Gateway untuk external integrations
- **Message Broker**: Event-driven notifications (Kafka/RabbitMQ)
- **PubSub**: Optional protocol untuk streaming notifications

### 3. Business Use Cases
- Order lifecycle notifications (created, payment received, driver assigned)
- Customer authentication (OTP via SMS/WhatsApp/Email)
- Receipt & invoice delivery
- Driver app notifications
- Customer care communications
- Legacy system integrations

## Key Features

### Email Strategy Pattern
Service menggunakan Strategy Pattern untuk routing email berdasarkan:
- **Channel**: WebReservation, MyBB, Corporate
- **Service Category**: RIDE, RENT
- **Vehicle Type**: Airport Transfer, BlueBird, SilverBird, GoldenBird

Lihat: [[02-email-strategy]]

### Rule Engine
Event-driven notification system menggunakan Rule Engine dengan Factory Pattern:
- Event listener untuk booking, order, payment events
- Dynamic rule registration
- Repository injection via factory

Lihat: [[03-rule-engine]]

## Architecture Highlights

### Layered Architecture
```
┌─────────────────────────────────┐
│  Transport Layer (gRPC/REST)    │
├─────────────────────────────────┤
│  Usecase Layer (Business Logic) │
├─────────────────────────────────┤
│  Repository Layer (Data Access) │
├─────────────────────────────────┤
│  External Services & Databases  │
└─────────────────────────────────┘
```

### Key Patterns
1. **Contract-First Design**: Protobuf untuk API definition
2. **Dependency Injection**: Manual wiring di main.go
3. **Interface Segregation**: Clean repository interfaces
4. **Factory Pattern**: Rule engine instantiation
5. **Strategy Pattern**: Email routing logic

## Technical Stack

### Core Dependencies
- **Database**: PostgreSQL (via GORM)
- **Cache**: Redis (duplicate request prevention, rate limiting)
- **Messaging**: Message broker untuk event-driven architecture
- **Email Providers**: 
  - BBOne Email Service
  - SendGrid (backup/alternative)
- **SMS Gateway**: DART SMS Gateway
- **Push Notification**: Firebase Cloud Messaging (FCM)
- **WhatsApp**: WhatsApp Meta API

### Observability
- **Logging**: Structured logging via `aphrodite/logger`
- **Metrics**: Prometheus metrics
- **Tracing**: Elastic APM integration
- **Health Check**: `/health` endpoint

## Service Status

### Maturity: **Production-Ready & Stable**
- ✅ Comprehensive test coverage
- ✅ Multi-protocol support
- ✅ Observability integration
- ✅ Graceful shutdown handling
- ⚠️ Legacy code migration in progress

### Active Development
- Refactoring legacy notification methods
- Standardizing error handling
- Enhancing email strategy coverage
- Improving documentation

## Architecture Assessment (Nov 2025)

**Overall Rating**: ⭐⭐⭐⭐ (4/5)

**Strengths**:
- Well-organized layered architecture
- Strong separation of concerns
- Excellent observability setup
- Production-grade error handling
- Comprehensive API coverage

**Areas for Improvement**:
1. Complete legacy code migration
2. Standardize error handling across all flows
3. Add C4 architecture diagrams
4. Enhance repository interface segregation

Source: `_docs/architecture_assessment.md`

## Related Documentation
- [[02-email-strategy|Email Strategy Pattern]]
- [[03-rule-engine|Rule Engine Architecture]]
- [[04-api-reference|API Reference]]
- [[05-dependencies|External Dependencies]]
- [[06-legacy-migration|Legacy Migration Plan]]

## Quick Links
- [[#API Documentation|API Docs]]
- [[#Configuration|Config Guide]]
- [[#Development|Dev Setup]]
- [[#Testing|Test Guide]]

---
**Last Updated**: 2025-01-07
**Documentation Status**: Complete
**Service Owner**: MRG Team