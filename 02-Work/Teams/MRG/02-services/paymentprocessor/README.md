---
tags:
  - mrg
  - service
  - paymentprocessor
  - payment
  - transaction
  - grpc
  - documentation
team: MRG
type: service-documentation
title: Payment Processor
status: production
created: '2025-01-07'
updated: '2025-01-07'
grpc_port: 6007
rest_port: 8007
repository: git.bluebird.id/mybb-ms/paymentprocessor
tech_stack:
  - go
  - grpc
  - postgresql
  - redis
  - pubsub
---
# Payment Processor

**Team**: MRG (Meta Reservation Gateway)  
**Status**: ✅ Production  
**Repository**: `git.bluebird.id/mybb-ms/paymentprocessor`

---

## 📋 Overview

Payment Processor adalah microservice yang bertanggung jawab untuk mengelola seluruh proses pembayaran dalam ekosistem MyBluebird. Service ini menjadi jantung payment orchestration yang menangani berbagai metode pembayaran mulai dari kartu kredit, e-wallet, corporate voucher, trip voucher, hingga promo redemption.

Service ini berperan sebagai gateway antara aplikasi MyBluebird dengan berbagai payment provider eksternal seperti UPG (Universal Payment Gateway), Corporate Portal, dan HOGateway.

### Fungsi Utama

- **Payment Processing** - Complete transaction, cancel transaction, refund, adjustment
- **Credit Card Management** - Register CC, remove CC, pre-authorization, void
- **Corporate Payment** - ECV (E-Corporate Voucher) verification, reservation, completion
- **Trip Voucher** - Add, verify, start, complete trip voucher transaction
- **E-Wallet Integration** - Login wallet, wallet balance, wallet transaction
- **Promo System** - Promo redemption, calculation, reserved amount management
- **Direct Payment** - Direct pay, direct payment v1.1 untuk berbagai payment method
- **Payment Method** - Get payment method, update payment method, get token
- **Transaction Management** - Reserve balance, check sufficient balance, transaction status
- **Callback Handling** - Handle callback dari payment gateway (pre-auth, charge status)
- **HO Transaction** - Handle hold-on transaction untuk payment tertunda
- **Refund Management** - Transaction refund, payment refund webhook
- **Driver Tips** - Tips driver via berbagai payment method

---

## 🛠️ Tech Stack

| Component | Technology |
|-----------|------------|
| Language | Go 1.24 |
| Protocol | gRPC + REST (gRPC-Gateway) + PubSub |
| Database | PostgreSQL |
| Cache | Redis (Primary & Legacy) |
| Message Queue | Google PubSub |
| Monitoring | Elastic APM |
| Container | Docker, Kubernetes |

---

## 🔑 Konsep Utama

### 1. Transaction Lifecycle

Payment transaction memiliki lifecycle yang kompleks:

- **Reserve**: Reserve balance untuk payment method tertentu
- **Start**: Mulai transaction (untuk voucher/wallet)
- **Complete**: Complete transaction setelah trip selesai
- **Cancel**: Cancel transaction jika trip dibatalkan
- **Void**: Void/rollback pre-authorization
- **Refund**: Refund pembayaran ke customer

### 2. Payment Method Types

Service mendukung berbagai payment method:

- **Cash** - Pembayaran tunai
- **Credit Card** - Kartu kredit via UPG Gateway
- **LinkAja** - E-wallet LinkAja
- **OVO** - E-wallet OVO
- **E-Corporate Voucher (ECV)** - Corporate voucher via Corporate Portal
- **Trip Voucher** - Trip voucher untuk corporate booking
- **Promo** - Promo code redemption

### 3. Payment Flow Patterns

**Standard Payment Flow:**
1. Get payment method list
2. Reserve balance (pre-check)
3. Start transaction
4. Complete transaction
5. Handle callback dari gateway

**Corporate Payment Flow (ECV):**
1. Verify ECV
2. Get ECV policy
3. Validate ECV policy
4. Reserve balance ECV
5. Complete ECV transaction
6. Void/Cancel jika diperlukan

**Pre-Authorization Flow (CC):**
1. Pre-auth CC reserve (hold amount)
2. Start transaction
3. Pre-auth CC status check
4. Pre-auth callback handling
5. Complete/void berdasarkan final amount

### 4. Promo Integration

- **Promo Redemption** - Redeem promo code dengan validasi
- **Promo Calculation** - Calculate promo amount sebelum payment
- **Reserved Budget** - Reserve promo budget untuk prevent double redemption
- **Marketing Promo** - Validate marketing promo eligibility

### 5. Transaction Synchronization

- **Async Processing** - PubSub untuk payment processing
- **Callback Handling** - Handle callback dari payment gateway
- **Status Sync** - Sync transaction status dengan order service
- **Error Recovery** - Transaction error logging dan retry mechanism

---

## 🔌 Dependencies

### Internal Services

| Service | Purpose | Protocol |
|---------|---------|----------|
| **UPG Gateway** | Universal payment gateway untuk CC, e-wallet | gRPC |
| **UPG MRG Gateway** | UPG gateway khusus untuk MRG | gRPC |
| **UPG Payment** | UPG payment processing | gRPC |
| **Corporate Portal (CP)** | Corporate voucher management | gRPC |
| **Promo Gateway** | Promo validation dan redemption | gRPC |
| **HOGateway** | Hold-on gateway untuk payment tertunda | gRPC |
| **Order Query** | Order information dan history | gRPC |
| **Taxi Order Processor (TOP)** | Order processing dan driver info | gRPC |
| **Citrans Order Processor (COP)** | Citrans order processing | gRPC |
| **User Service** | User profile dan payment method | gRPC |
| **Publisher** | Event publishing | gRPC |

### Client Libraries

| Library | Version | Purpose |
|---------|---------|---------|
| `upggatewayclient` | v0.0.23 | UPG gateway integration |
| `upgmrggatewayclient` | - | UPG MRG gateway integration |
| `upgpaymentclient` | - | UPG payment integration |
| `cpclient` | v1.1.1 | Corporate Portal integration |
| `promogatewayclient` | v1.1.16 | Promo gateway integration |
| `hogateway` | - | HO Gateway integration |
| `orderqueryclient` | v1.0.4 | Order query integration |
| `topclient` | v0.0.15 | Taxi order processor |
| `copclient` | v0.0.9 | Citrans order processor |
| `userclient` | v0.0.20 | User service integration |
| `commonmessaging` | v0.1.28 | PubSub messaging |
| `aphrodite` | v1.9.32 | Common framework |

### Infrastructure

| Component | Purpose |
|-----------|---------|
| **PostgreSQL** | Transaction data, payment logs, error logs |
| **Redis** | Session cache, temporary data, rate limiting |
| **Redis Legacy** | Legacy redis untuk backward compatibility |
| **Google PubSub** | Event streaming (payment events, HO transaction) |

### Repository Structure

```go
type Repository struct {
    DB                     PaymentDB
    Redis                  repoiface.Redis
    RedisLegacy            repoiface.RedisLegacy
    UPGGateway             repoiface.UPGGateway
    UPGMRGGateway          repoiface.UPGMRGGateway
    UPGPayment             repoiface.UPGPayment
    CorpPortal             repoiface.CorpPortal
    PromoGateway           repoiface.PromoGateway
    HOGateway              repoiface.HOGateway
    OrderQuery             repoiface.OrderQuery
    TaxiOrderProcessor     repoiface.TaxiOrderProcessor
    CitransOrderProcessor  repoiface.CitransOrderProcessor
    UserService            repoiface.UserService
    Publisher              repoiface.Publisher
    PubSub                 repoiface.PubSub
}

type PaymentDB struct {
    Transactions     repoiface.PaymentDB
    TxErrorLogDetail repoiface.PaymentDB
}
```

---

## 📡 API Contracts

### gRPC Service

**Package**: `paymentprocessor`  
**Proto File**: `contract/payment_processor.proto`  
**Ports**: gRPC `6007`, REST `8007`

### Methods Overview

#### Core Transaction Management

| Method | Description | Interceptors |
|--------|-------------|--------------|
| `CompleteTx` | Complete transaction setelah trip selesai | ValidateInput |
| `CancelTx` | Cancel transaction | ValidateInput |
| `TxStatus` | Get transaction status | ValidateInput |
| `TxCancelSync` | Sync cancel transaction | - |
| `ReserveBalance` | Reserve balance untuk payment method | ValidateInput |
| `CheckSufficientBalanceForOrder` | Check apakah balance cukup | - |
| `UpdateTxResult` | Update transaction result | ValidateInput |
| `AdjustmentTxResult` | Adjustment transaction result | ValidateInput |
| `TxCompletionCallback` | Transaction completion callback | ValidateInput |

#### Direct Payment

| Method | Description | Interceptors |
|--------|-------------|--------------|
| `DirectPay` | Direct payment untuk berbagai method | - |
| `DirectPaymentV1_1` | Direct payment v1.1 (enhanced) | - |
| `SinglePaymentMethod` | Single payment method transaction | ValidateInput |

#### Credit Card Management

| Method | Description | Interceptors |
|--------|-------------|--------------|
| `RegisterCC` | Register kartu kredit | ValidateMetadata |
| `RemoveCC` | Remove kartu kredit | ValidateMetadata |
| `PaymentCCGoldenbird` | Payment dengan CC untuk Goldenbird | ValidateInput |
| `VoidCCGoldenbird` | Void CC transaction Goldenbird | ValidateInput |

#### Pre-Authorization (Credit Card)

| Method | Description | Interceptors |
|--------|-------------|--------------|
| `PreauthCCReserve` | Reserve pre-authorization | ValidateInput |
| `PreauthCCStatus` | Check pre-auth status | ValidateInput |
| `PreauthCCCallback` | Pre-auth callback handler | ValidateInput |
| `CallbackPreAuth` | Callback pre-auth dari gateway | ValidateInput |

#### Corporate Payment (ECV)

| Method | Description | Interceptors |
|--------|-------------|--------------|
| `VerifyECV` | Verify corporate voucher | ValidateMetadata |
| `GetECVPolicy` | Get ECV policy | - |
| `ValidateECVPolicy` | Validate ECV policy | - |
| `GetECVCompanyCode` | Get company code | - |
| `ReserveBalanceECVGoldenbird` | Reserve ECV balance | ValidateInput |
| `CompleteTxECVGoldenbird` | Complete ECV transaction | ValidateInput |
| `VoidECVGoldenbird` | Void ECV transaction | ValidateInput |
| `CancelOrderECVGoldenbird` | Cancel order dengan ECV | ValidateInput |
| `VoucherAccountDetail` | Get voucher account detail | ValidateInput |

#### Trip Voucher

| Method | Description | Interceptors |
|--------|-------------|--------------|
| `AddTripVoucher` | Add trip voucher | ValidateMetadata, ValidateInput |
| `VerifyTripVoucher` | Verify trip voucher | - |
| `StartTripVoucherTx` | Start trip voucher transaction | - |
| `CompleteTripVoucherTx` | Complete trip voucher transaction | - |

#### E-Wallet

| Method | Description | Interceptors |
|--------|-------------|--------------|
| `LoginWallet` | Login e-wallet | ValidateMetadata, ValidateInput |
| `RemoveWallet` | Remove e-wallet | ValidateMetadata, ValidateInput |
| `GetEwalletURL` | Get e-wallet URL | - |
| `GetEwalletBalance` | Get e-wallet balance | - |
| `WalletBalance` | Get wallet balance v4 | ValidateInput |
| `StartWalletTx` | Start wallet transaction | - |
| `CompleteWalletTx` | Complete wallet transaction | - |

#### Promo System

| Method | Description | Interceptors |
|--------|-------------|--------------|
| `PromoRedeem` | Redeem promo code | ValidateInput |
| `PromoRedeemOld` | Redeem promo (old version) | ValidateInput |
| `PromoRedeemCalculation` | Calculate promo amount | ValidateInput |
| `PromoReservedAmount` | Reserve promo budget | ValidateInput |
| `PromoRemoveReservedBudget` | Remove reserved promo budget | ValidateInput |
| `ValidatePromo` | Validate promo | - |
| `ValidateMarketingPromo` | Validate marketing promo | - |

#### Payment Method

| Method | Description | Interceptors |
|--------|-------------|--------------|
| `GetPaymentMethod` | Get payment method list | - |
| `UpdatePaymentMethod` | Update payment method | - |
| `GetPaymentToken` | Get payment token | ValidateInput |
| `GetSupportedPaymentMethod` | Get supported payment methods | - |

#### Refund & Adjustment

| Method | Description | Interceptors |
|--------|-------------|--------------|
| `Refund` | Refund transaction | - |
| `TransactionRefund` | Transaction refund v2 | - |
| `PaymentRefundWebhook` | Payment refund webhook | - |
| `AdjustmentChargeTx` | Adjustment charge transaction | ValidateInput |
| `CallbackAdjustChargeStatus` | Callback adjustment charge | - |
| `CallbackUpdateChargeStatus` | Callback update charge status | - |
| `ChargeCancelFee` | Charge cancellation fee | - |

#### Hold-On Transaction

| Method | Description | Interceptors |
|--------|-------------|--------------|
| `HOTransaction` | Hold-on transaction | - |
| `HOECVTransaction` | HO ECV transaction | - |
| `ProcessHOTransaction` | Process HO transaction (PubSub) | - |
| `ProcessHOECVTransaction` | Process HO ECV transaction (PubSub) | - |
| `PushHOGoldenbirdCC` | Push HO Goldenbird CC | ValidateInput |

#### Goldenbird Specific

| Method | Description | Interceptors |
|--------|-------------|--------------|
| `UpdatePaymentGoldenbird` | Update payment Goldenbird | - |
| `CancelTxGoldenbird` | Cancel transaction Goldenbird | ValidateInput |

#### Miscellaneous

| Method | Description | Interceptors |
|--------|-------------|--------------|
| `HealthCheck` | Health check | - |
| `ValidateEpaymentProfile` | Validate epayment profile | ValidateInput |
| `NotifyReferral` | Notify referral | - |
| `TipsDriver` | Tips driver | - |
| `RechargeTx` | Recharge transaction | ValidateInput |
| `TransactionStatus` | Get transaction status | - |
| `CompleteTxOld` | Complete transaction (old) | - |

---

## 🎯 Payment Flow Examples

### 1. Standard Cash Payment Flow

```
Customer -> Reserve Balance (Check only)
         -> Complete Transaction
         -> Transaction Success
```

### 2. Credit Card Payment Flow

```
Customer -> Get Payment Method
         -> Register CC (if new card)
         -> Reserve Balance
         -> Start Transaction
         -> Complete Transaction
         -> Callback dari UPG Gateway
         -> Transaction Success
```

### 3. Pre-Auth Credit Card Flow

```
Customer -> Pre-auth CC Reserve (Hold amount)
         -> Start Transaction
         -> Trip Started
         -> Pre-auth CC Status Check
         -> Trip Completed
         -> Complete Transaction (Final amount)
         -> Pre-auth Callback
         -> Transaction Success
```

### 4. Corporate ECV Payment Flow

```
Customer -> Verify ECV
         -> Get ECV Policy
         -> Validate ECV Policy
         -> Reserve Balance ECV
         -> Start Transaction
         -> Complete Transaction ECV
         -> Transaction Success

If Cancel:
         -> Cancel Order ECV / Void ECV
```

### 5. Trip Voucher Payment Flow

```
Customer -> Verify Trip Voucher
         -> Add Trip Voucher (if needed)
         -> Start Trip Voucher Transaction
         -> Trip Started
         -> Trip Completed
         -> Complete Trip Voucher Transaction
         -> Transaction Success
```

### 6. Promo Redemption Flow

```
Customer -> Validate Promo
         -> Promo Calculation
         -> Reserve Promo Budget
         -> Promo Redeem
         -> Start Transaction
         -> Complete Transaction
         -> Transaction Success

If Cancel:
         -> Remove Reserved Promo Budget
```

### 7. E-Wallet Payment Flow

```
Customer -> Get E-wallet URL (for login)
         -> Login Wallet
         -> Get Wallet Balance
         -> Start Wallet Transaction
         -> Trip Completed
         -> Complete Wallet Transaction
         -> Transaction Success
```

---

## ⚙️ Configuration

### Environment Variables

```env
# Application
APP_NAME=payment-processor
GRPC_PORT=6007
REST_PORT=8007
LOG_LEVEL=INFO
LOG_DIRECTORY=

# Database
DB_HOST=
DB_HOST_READ=
DB_USERNAME=
DB_PASSWORD=
DB_SSL_MODE=
DB_PORT=
DB_NAME=

# Redis
REDIS_HOST=
REDIS_PORT=
REDIS_PASSWORD=
REDIS_DB=

# Redis URLs (alternative)
REDIS_URL=
REDIS_LEGACY_URL=

# Google Cloud
PROJECT_ID=mybluebird
PUBSUB_EMULATOR_HOST=
GOOGLE_APPLICATION_CREDENTIALS=
BROKER_CONFIG=cert/broker_config.yaml

# External Services - UPG
UPG_GATEWAY_HOST=
UPG_GATEWAY_PORT=
UPG_MRG_GATEWAY_HOST=
UPG_MRG_GATEWAY_PORT=
UPG_PAYMENT_HOST=
UPG_PAYMENT_PORT=

# External Services - Corporate
CORP_PORTAL_HOST=
CORP_PORTAL_PORT=

# External Services - Others
PROMO_GATEWAY_HOST=
PROMO_GATEWAY_PORT=
HO_GATEWAY_HOST=
HO_GATEWAY_PORT=
ORDER_QUERY_HOST=
ORDER_QUERY_PORT=
TAXI_ORDER_PROCESSOR_HOST=
TAXI_ORDER_PROCESSOR_PORT=
CITRANS_ORDER_PROCESSOR_HOST=
CITRANS_ORDER_PROCESSOR_PORT=
USER_SERVICE_HOST=
USER_SERVICE_PORT=
PUBLISHER_HOST=
PUBLISHER_PORT=

# Feature Flags
ENABLE_PREAUTH_CC=true
ENABLE_HO_TRANSACTION=true
ENABLE_PROMO_REDEMPTION=true

# Timeout Settings
PAYMENT_GATEWAY_TIMEOUT=30s
TRANSACTION_TIMEOUT=60s

# Retry Settings
MAX_RETRY_ATTEMPT=3
RETRY_INTERVAL=5s
```

---

## 📂 Project Structure

```
paymentprocessor/
├── main.go                      # Entry point
├── go.mod                       # Dependencies
├── Dockerfile                   # Container build
├── Jenkinsfile                  # CI/CD pipeline
│
├── cert/                        # Certificates
│   ├── pubsub.json              # PubSub credentials
│   └── broker_config.yaml       # Broker config
│
├── config/                      # Configuration
│   ├── default.go               # Default config values
│   ├── logger/                  # Logger setup
│   └── repository/              # Repository initialization
│
├── constants/                   # Constants
│   ├── goldenbird.go
│   ├── payment_category.go
│   ├── payment_type.go
│   ├── payment_text.go
│   ├── preauth.go
│   ├── push_notification.go
│   ├── redis.go
│   ├── topic.go
│   └── transactions_status.go
│
├── contract/                    # API contracts
│   ├── payment_processor.proto
│   ├── payment_processor.pb.go
│   ├── payment_processor_grpc.pb.go
│   ├── payment_processor.pb.gw.go
│   ├── payment_processor.swagger.json
│   ├── payment_processor.yaml
│   ├── payment_processor.broker.yaml
│   ├── response.go
│   ├── rest_formatter.go
│   ├── validate_input.go
│   ├── validate_metadata.go
│   └── formatter/               # Response formatters
│       └── payment_method.go
│
├── repository/                  # Data access layer
│   ├── base_repository.go
│   ├── repoiface/               # Interfaces (16 files)
│   │   ├── model/               # Data models
│   │   ├── upggateway.go
│   │   ├── upgmrggateway.go
│   │   ├── upgpayment.go
│   │   ├── corpportal.go
│   │   ├── promogateway.go
│   │   ├── hogateway.go
│   │   ├── orderquery.go
│   │   ├── taxiorderprocessor.go
│   │   ├── userservice.go
│   │   ├── paymentdb.go
│   │   ├── redis.go
│   │   ├── redislegacy.go
│   │   ├── publisher.go
│   │   └── pubsub.go
│   │
│   ├── repomock/                # Mocks (14 files)
│   │
│   ├── upggateway/              # UPG Gateway client
│   │   ├── base_upggateway.go
│   │   └── implementor.go
│   │
│   ├── upgmrggateway/           # UPG MRG Gateway client
│   │   ├── base_upgmrggateway.go
│   │   ├── endpoints.go
│   │   ├── implementor.go
│   │   └── signature.go
│   │
│   ├── upgpayment/              # UPG Payment client
│   │   ├── base_upgpayment.go
│   │   ├── endpoints.go
│   │   └── implementor.go
│   │
│   ├── corpportal/              # Corporate Portal client
│   │   ├── base_corpportal.go
│   │   ├── endpoints.go
│   │   └── implementor.go
│   │
│   ├── promogateway/            # Promo Gateway client
│   │   ├── base_promogateway.go
│   │   └── implementor.go
│   │
│   ├── hogateway/               # HO Gateway client
│   │   ├── base_hogateway.go
│   │   ├── endpoints.go
│   │   └── implementor.go
│   │
│   ├── orderquery/              # Order Query client
│   │   ├── base_orderquery.go
│   │   └── implementor.go
│   │
│   ├── taxiorderprocessor/     # Taxi Order Processor client
│   │   ├── base_taxiorderprocessor.go
│   │   └── implementor.go
│   │
│   ├── cititransorderprocessor/ # Citrans Order Processor
│   │   ├── base_cititransorderprocessor.go
│   │   └── implementor.go
│   │
│   ├── userservice/             # User Service client
│   │   ├── base_userservice.go
│   │   └── implementor.go
│   │
│   ├── paymentdb/               # Database repository
│   │   ├── base_paymentdb.go
│   │   ├── transactions.go
│   │   └── tx_error_log_detail.go
│   │
│   ├── redis/                   # Redis repository
│   │   ├── base_redis.go
│   │   └── implementor.go
│   │
│   ├── redislegacy/             # Redis Legacy
│   │   ├── base_redis.go
│   │   └── implementor.go
│   │
│   ├── publisher/               # Event publisher
│   │   ├── base_publisher.go
│   │   └── implementor.go
│   │
│   └── pubsub/                  # PubSub clients
│       ├── base_pubsub.go
│       ├── implementor.go
│       ├── payment_client.go
│       ├── order_client.go
│       ├── user_client.go
│       └── notif_client.go
│
├── usecase/                     # Business logic (100+ use cases with tests)
│   ├── base_usecase.go
│   ├── complete_tx.go
│   ├── cancel_tx.go
│   ├── reserve_balance.go
│   ├── promo_redeem.go
│   ├── payment_cc_goldenbird.go
│   ├── preauth_cc_reserve.go
│   ├── register_cc.go
│   ├── verify_ecv.go
│   └── ... (90+ more use cases with comprehensive tests)
│
├── transport/                   # Transport layer (100+ handlers)
│   ├── base_transport.go
│   ├── complete_tx.go
│   ├── cancel_tx.go
│   ├── reserve_balance.go
│   └── ... (95+ endpoint handlers)
│
├── server/                      # Server setup
│   ├── grpc.go
│   ├── rest.go
│   ├── rest_option.go
│   ├── broker.go                # PubSub broker
│   └── metric.go
│
├── util/                        # Utilities
│   ├── server.go
│   ├── converter.go
│   ├── crypto/                  # Crypto utilities
│   │   ├── profile.go
│   │   └── signature.go
│   ├── errors/                  # Error handling
│   │   ├── errors.go
│   │   └── mapping_upg.go
│   ├── interceptor/             # gRPC interceptors
│   │   ├── base_interceptor.go
│   │   ├── validate_input.go
│   │   └── validate_metadata.go
│   └── pubsub/                  # PubSub utilities
│       ├── subscription.go
│       └── topic.go
│
├── sql/                         # SQL migrations
│   └── 20241024-1402-alter-table-transactions.sql
│
└── k8s/                         # Kubernetes manifests
    ├── deployment.yaml
    ├── service.yaml
    └── huawei-application.yaml
```

---

## 📊 Database Schema

### Main Tables

| Table | Description |
|-------|-------------|
| `transactions` | Core transaction data |
| `tx_error_log_detail` | Transaction error log untuk debugging |

### Transaction Fields

```go
type Transaction struct {
    ID                  int64
    OrderID             int64
    CustomerID          string
    PaymentType         string
    PaymentCategory     string
    Amount              float64
    Status              string
    TransactionDate     time.Time
    PaymentToken        string
    PromoCode           string
    PromoAmount         float64
    RefundAmount        float64
    CancelFee           float64
    TipsAmount          float64
    PreAuthID           string
    ECVAccountID        string
    TripVoucherID       string
    WalletID            string
    CreatedAt           time.Time
    UpdatedAt           time.Time
}
```

---

## 🚨 Error Codes

Payment Processor menggunakan error code dengan prefix `PYPC`:

| Code | Description |
|------|-------------|
| PYPC-50000 | Internal Error |
| PYPC-40001 | Missing Parameter Error |
| PYPC-40002 | Invalid Parameter Error |
| PYPC-40003 | Parse Parameter Error |
| PYPC-40004 | Invalid Payment Status |
| PYPC-40901 | Promo Code Expired Error |
| PYPC-40902 | UPG Default Error |
| PYPC-40903 | UPG CC Expired Error |
| PYPC-40904 | UPG CC Insufficient Error |
| PYPC-40905 | UPG CC Limit Error |
| PYPC-40906 | UPG Trip Fare Error |
| PYPC-40907 | UPG CC Invalid Error |
| PYPC-40908 | UPG CC Lost/Stolen Error |
| PYPC-40909 | UPG CC Declined Error |
| PYPC-40910 | UPG Payment Gateway Error |
| PYPC-40911 | UPG Recharge Limit Error |
| PYPC-40912 | Balance Insufficient |
| PYPC-42000 | External Service Error |

---

## 🔗 Related Documentation

- [[api-reference|API Reference]]
- [[dependencies|Dependencies]]
- [[02-Work/Teams/MRG/00-overview/README|MRG Team Overview]]

---

## 🏷️ Tags

#mrg #service #paymentprocessor #payment #transaction #grpc #documentation

---

*Last Updated*: 2025-01-07  
*Generated from*: Repository analysis
