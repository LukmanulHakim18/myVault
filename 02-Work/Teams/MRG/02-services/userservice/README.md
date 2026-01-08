---
tags:
  - mrg
  - service
  - userservice
  - user-management
  - profile
  - payment
  - grpc
  - documentation
team: MRG
type: service-documentation
title: User Service
status: production
created: '2025-01-05'
updated: '2025-01-05'
grpc_port: 6015
rest_port: 8015
repository: git.bluebird.id/mybb-ms/userservice
tech_stack:
  - go
  - grpc
  - postgresql
  - redis
  - pubsub
  - gcs
---
# User Service

**Team**: MRG (Meta Reservation Gateway)  
**Status**: ✅ Production  
**Repository**: `git.bluebird.id/mybb-ms/userservice`

---

## 📋 Overview

MyBB User Service adalah microservice yang bertanggung jawab untuk mengelola data pengguna dan fitur-fitur terkait user dalam ekosistem MyBluebird. Service ini menyediakan fungsionalitas lengkap untuk manajemen profil, alamat favorit, metode pembayaran, dan berbagai preferensi user.

### Fungsi Utama

- **Profile Management** - CRUD profil user, update profile, verifikasi email
- **Authentication Support** - Login, password management (change, reset)
- **Favorite Address** - Kelola alamat favorit user (v1 dan v2)
- **Favorite Trip** - Simpan trip favorit untuk quick reorder
- **Payment Method** - Kelola metode pembayaran user (personal & corporate)
- **Navigation State** - Track navigation state untuk UX improvement
- **Home Services** - Kelola menu home services di aplikasi
- **User Location** - Save/get last location user
- **Referral System** - Kelola referral code dan rewards
- **Account Management** - Delete account, whitelist check
- **Feature Flags** - Check enabled features per user
- **Notification Preferences** - Kelola preferensi notifikasi promo

---

## 🛠️ Tech Stack

| Component | Technology |
|-----------|------------|
| Language | Go 1.24 |
| Protocol | gRPC + REST (gRPC-Gateway) |
| Database | PostgreSQL |
| Cache | Redis |
| Geo Cache | Redis (separate DB) |
| Message Queue | Google PubSub |
| Storage | Google Cloud Storage |
| Monitoring | Elastic APM |
| Container | Docker, Kubernetes |

---

## 🔑 Konsep Utama

### 1. User Profile
- Profile berisi informasi dasar: name, email, phone, profile image
- Extended info: gender, DOB, domicile, occupation, hobby
- Verification status untuk email dan phone
- Legacy user flag untuk backward compatibility

### 2. Favorite Address System
- **V1**: Basic favorite address dengan tag, position, bookmark
- **V2**: Enhanced dengan location_name, address_short_name, address_long_name, driver_notes
- Maximum 20 favorite addresses per user
- Support reordering position

### 3. Payment Method Management
- **Personal Payment**: Cash, Credit Card, LinkAja, etc.
- **Corporate Payment**: Company voucher, Trip voucher
- Default payment setting (personal & corporate)
- Car type availability per payment method

### 4. Navigation User State
- Track visited states untuk onboarding/tutorial
- Key-value pairs untuk flexible state management

---

## 🔌 Dependencies

### Internal Services

| Service | Purpose | Protocol |
|---------|---------|----------|
| **Auth Service** | Token validation, authentication | gRPC |
| **Config Service** | Feature flags, whitelist config | gRPC |
| **Payment Processor** | Payment method validation, balance check | gRPC |
| **Promo Gateway** | Referral system, vouchers | gRPC |
| **Order Query** | Order history untuk favorite trip | gRPC |
| **Publisher** | Event publishing | gRPC |
| **Taxi Order Processor** | Outlet info, booth info | gRPC |
| **Map Service** | Location services | HTTP |

### Client Libraries

| Library | Version | Purpose |
|---------|---------|---------|
| `authclient` | v0.0.6 | Auth service integration |
| `configseviceclient` | v0.0.9 | Config service integration |
| `paymentprocessorclient` | v1.1.34 | Payment processor integration |
| `promogatewayclient` | v1.1.20 | Promo gateway integration |
| `orderqueryclient` | v1.0.4 | Order query integration |
| `topclient` | v0.0.18 | Taxi order processor integration |
| `commonmessaging` | v0.1.18 | PubSub messaging |
| `aphrodite` | v1.9.43 | Common framework |

### Infrastructure

| Component | Purpose |
|-----------|---------|
| **PostgreSQL** | Main data storage (users, favorite addresses, payment methods) |
| **Redis** | Session cache, rate limiting |
| **Redis Geo** | Geospatial data storage |
| **Redis Stream** | Event streaming |
| **Google Cloud Storage** | Profile image storage |
| **Google PubSub** | Event messaging |

### Repository Structure

```go
type Repository struct {
    Db                 Db
    Redis              repoiface.RedisRepository
    RedisStream        repoiface.RedisRepository
    RedisGeo           repoiface.RedisGeoRepository
    Storage            repoiface.StorageRepository
    Location           repoiface.LocationRepository
    PaymentProcessor   repoiface.PaymentProcessor
    PromoGateway       repoiface.PromoGateway
    ConfigService      repoiface.ConfigService
    OrderQuery         repoiface.OrderQuery
    AuthService        repoiface.AuthService
    Publisher          repoiface.Publisher
    TaxiOrderProcessor repoiface.TaxiOrderProcessor
    MapService         repoiface.MapService
}

type Db struct {
    Transaction            repoiface.DbTransaction
    User                   repoiface.UserRepository
    UserAdditionalInfo     repoiface.UserAdditionalInfoRepository
    UserVerification       repoiface.UserVerificationRepository
    UserResendOtpCount     repoiface.UserResendOtpCountRepository
    Device                 repoiface.DeviceRepository
    LoginSoftBanned        repoiface.LoginSoftBannedRepository
    FavoriteAddress        repoiface.FavoriteAddressRepository
    NavigationUserState    repoiface.NavigationUserStateRepository
    PaymentMethodUserRepo  repoiface.PaymentMethodUserRepository
    PaymentMethodTypeRepo  repoiface.PaymentMethodTypeRepository
    HomeServicesRepo       repoiface.HomeServicesRepository
    UserLocationRepository repoiface.UserLocationRepository
    FavoriteTrip           repoiface.FavoriteTripRepository
    Country                repoiface.CountryRepository
    UserNotification       repoiface.UserNotificationRepository
}
```

---

## 📡 API Contracts

### gRPC Service

**Package**: `userservice`  
**Proto File**: `contract/user-service.proto`  
**Ports**: gRPC `6015`, REST `8015`

### Methods Overview

#### Profile Management

| Method | Description | Interceptors |
|--------|-------------|--------------|
| `GetProfile` | Get user profile by ID | - |
| `UpdateProfile` | Update user profile | ValidateMetadata |
| `Verify` | Verify profile update (email/phone) | - |
| `ResendVerificationUpdateProfile` | Resend verification code | - |
| `UpdateLanguage` | Update user language preference | - |
| `RequestVerificationEmail` | Request email verification | - |

#### User Management

| Method | Description | Interceptors |
|--------|-------------|--------------|
| `GetUser` | Find user by internal_id/phone/email | - |
| `GetUserInternalIdByPhoneNumber` | Get internal ID by phone | ValidateInput |
| `CreateUser` | Create new user | DeviceVersion, Token |
| `Login` | User login | - |
| `ChangePassword` | Change password | - |
| `ResetPassword` | Reset password | - |
| `DeleteAccount` | Delete user account | ValidateInput, ValidateMetadata |
| `IsUserAllowedToDeleteAccount` | Check if deletion allowed | ValidateMetadata |

#### Favorite Address (V1)

| Method | Description | Interceptors |
|--------|-------------|--------------|
| `ListFavoriteAddress` | List user's favorite addresses | - |
| `CreateFavoriteAddress` | Create favorite address | - |
| `UpdateFavoriteAddress` | Update favorite address | - |
| `DeleteFavoriteAddress` | Delete favorite address | CustomHttpCodeResponseNoContent |

#### Favorite Address (V2)

| Method | Description | Interceptors |
|--------|-------------|--------------|
| `GetFavoriteAddressesV2` | Get favorite addresses (paginated) | - |
| `CreateFavoriteAddressV2` | Create favorite address v2 | - |
| `UpdateFavoriteAddressV2` | Update favorite address v2 | - |
| `DeleteFavoriteAddressV2` | Delete favorite address v2 | CustomHttpCodeResponseNoContent |
| `UpdateFavoriteAddressesPosition` | Reorder favorite addresses | - |
| `GetSearchLocationHistory` | Get search location history | - |

#### Favorite Trip

| Method | Description | Interceptors |
|--------|-------------|--------------|
| `AddFavoriteTrip` | Add favorite trip | ValidateInput, ValidateMetadata |
| `GetFavoriteTrip` | Get favorite trips | ValidateMetadata |
| `DeleteFavoriteTrip` | Delete favorite trip | ValidateInput, ValidateMetadata |

#### Payment Method

| Method | Description | Interceptors |
|--------|-------------|--------------|
| `GetPaymentMethodUser` | Get user's payment methods | PaymentMethod |
| `GetOrderPaymentMethodUser` | Get payment methods for order | PaymentMethod |
| `GetDefaultPaymentMethodUser` | Get default payment method | PaymentMethod |
| `GetDefaultPaymentMethodSubscription` | Get default subscription payment | PaymentMethod |
| `UpsertPaymentMethodUser` | Add/update payment method | - |
| `DeletePaymentMethodUser` | Delete payment method | - |
| `UpdateDefaultPayment` | Set default payment | - |

#### Navigation State

| Method | Description | Interceptors |
|--------|-------------|--------------|
| `GetNavigationUserState` | Get navigation state | ValidateInput, ValidateMetadata |
| `GetNavigationUserStateList` | Get all navigation states | ValidateInput, ValidateMetadata |
| `SetNavigationUserState` | Set navigation state | ValidateInput, ValidateMetadata |

#### Home Services & Location

| Method | Description | Interceptors |
|--------|-------------|--------------|
| `GetHomeServices` | Get home services menu | ValidateInput, ValidateMetadata |
| `SaveLastLocation` | Save user's last location | ValidateInput, ValidateMetadata |
| `GetLastLocation` | Get user's last location | ValidateMetadata |

#### Referral & Additional Info

| Method | Description | Interceptors |
|--------|-------------|--------------|
| `GetReferralList` | Get referral list | ValidateMetadata |
| `GetReferralExplanation` | Get referral explanation | ValidateInput, ValidateMetadata |
| `GetListOfHobby` | Get hobby list | ValidateMetadata |
| `GetListOfDomicile` | Get domicile list | ValidateMetadata |
| `GetListOfOccupation` | Get occupation list | ValidateMetadata |

#### Utility & Feature Flags

| Method | Description | Interceptors |
|--------|-------------|--------------|
| `HealthCheck` | Health check | - |
| `WhitelistCheck` | Check user whitelist | - |
| `CheckEnabledFeature` | Check if feature enabled | - |
| `GetCountryList` | Get country list | - |
| `StreetHailingIsUserEligibleToUseBarcodeFeature` | Check barcode eligibility | ValidateMetadata |

#### Notification Preferences

| Method | Description | Interceptors |
|--------|-------------|--------------|
| `UpdateLastStatusPromoNotification` | Update promo notification status | - |
| `GetLastStatusPromoNotification` | Get promo notification status | - |

#### Other

| Method | Description | Interceptors |
|--------|-------------|--------------|
| `StoreSchoolBusParentActiveStatus` | Store school bus status | ValidateInput |
| `GetOutletInfo` | Get outlet/booth info | - |

---

## ⚙️ Configuration

### Environment Variables

```env
# Application
APP_NAME=user-service
GRPC_PORT=6015
REST_PORT=8015
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
REDIS_GEO_DB=

# Redis URLs (alternative)
REDIS_URL=
REDIS_GEO_URL=
REDIS_STREAM_URL=
REDIS_REPLICA_URL=

# Google Cloud
PROJECT_ID=mybluebird
PUBSUB_EMULATOR_HOST=
GOOGLE_APPLICATION_CREDENTIALS=

# External Services
CONFIG_SERVICE_HOST=
CONFIG_SERVICE_PORT=
PROMO_GATEWAY_HOST=
PROMO_GATEWAY_PORT=
PAYMENT_PROCESSOR_HOST=
PAYMENT_PROCESSOR_PORT=
ORDER_QUERY_HOST=
ORDER_QUERY_PORT=
AUTH_SERVICE_HOST=
AUTH_SERVICE_PORT=
PUBLISHER_HOST=
PUBLISHER_PORT=
TAXI_ORDER_PROCESSOR_HOST=
TAXI_ORDER_PROCESSOR_PORT=
MAP_SERVICE_HOST=
MAP_SERVICE_TIMEOUT=
MAP_SERVICE_TOKEN=

# Feature Settings
MAX_FAVORITE_ADDRESSES=20
MAX_LOGIN_ATTEMPT=5
PHONE_VERIFICATION_RESENDABLE_INTERVAL=60s

# Soft Ban Settings
PROMO_NOTIFICATION_ON_OFF_TOGGLE_SOFTBANNED_DURATION=60s
PROMO_NOTIFICATION_ON_OFF_TOGGLE_TOO_MUCH_REQUEST_DURATION=60s

# Whitelist
WHITELIST_ELIGIBLE_USER_STREETHAILING_BARCODE_FEATURE=
```

---

## 📂 Project Structure

```
userservice/
├── main.go                    # Entry point
├── go.mod                     # Dependencies
├── Dockerfile                 # Container build
├── Jenkinsfile                # CI/CD pipeline
│
├── cert/                      # Certificates
│   ├── pubsub.json            # PubSub credentials
│   └── broker_config.yaml     # Broker config
│
├── config/                    # Configuration
│   ├── default.go             # Default config values
│   ├── logger/                # Logger setup
│   └── repository/            # Repository initialization
│
├── constant/                  # Constants
│   ├── feature.go
│   └── user_notification.go
│
├── contract/                  # API contracts
│   ├── user-service.proto
│   ├── user-service.pb.go
│   ├── user-service_grpc.pb.go
│   ├── user-service.pb.gw.go
│   ├── user-service.swagger.json
│   ├── mapper.go
│   └── validate.go
│
├── dto/                       # Data Transfer Objects
│   ├── country.go
│   ├── filter_query.go
│   └── redis.go
│
├── model/                     # Domain models
│   ├── user.go
│   ├── favorite_address.go
│   ├── favorite_address_v2.go
│   ├── favorite_trip.go
│   ├── payment_method.go
│   ├── user_notification.go
│   └── ... (13 more models)
│
├── repository/                # Data access layer
│   ├── base_repository.go
│   ├── repoiface/             # Interfaces (17 files)
│   ├── repomock/              # Mocks (14 files)
│   ├── repodatabase/          # Database implementations (15 files)
│   ├── reporedis/             # Redis implementations
│   ├── reporedisgeo/          # Redis Geo implementations
│   ├── repostorage/           # Storage implementations
│   ├── repolocation/          # Location implementations
│   ├── authservice/           # Auth service client
│   ├── configservice/         # Config service client
│   ├── paymentprocessor/      # Payment processor client
│   ├── promogateway/          # Promo gateway client
│   ├── orderquery/            # Order query client
│   ├── publisher/             # Publisher client
│   ├── top/                   # Taxi order processor client
│   └── mapservice/            # Map service client
│
├── usecase/                   # Business logic (40+ use cases with tests)
│   ├── base_usecase.go
│   ├── profile.go
│   ├── user.go
│   ├── favorite_address.go
│   ├── payment_method.go
│   └── ... (with comprehensive tests)
│
├── transport/                 # Transport layer (50+ handlers)
│   ├── base_transport.go
│   └── ... (one file per endpoint)
│
├── server/                    # Server setup
│   ├── grpc.go
│   ├── rest.go
│   ├── rest_option.go
│   ├── pubsub.go
│   └── metric.go
│
├── util/                      # Utilities
│   ├── server.go
│   ├── constants/             # Constants (10 files)
│   ├── interceptor/           # gRPC interceptors (8 files)
│   ├── lib/                   # Helper libraries (12 files)
│   └── servopt/               # Server options
│
├── doc/                       # Documentation (PlantUML diagrams)
│   ├── change_password.plantuml
│   ├── favorite_address.plantuml
│   └── ... (8 diagrams)
│
├── sql/                       # SQL migrations
│   ├── schema.sql
│   └── data.sql
│
└── k8s/                       # Kubernetes manifests
    ├── deployment.yaml
    ├── service.yaml
    ├── gcp-application.yaml
    └── huawei-application.yaml
```

---

## 📊 Database Schema

### Main Tables

| Table | Description |
|-------|-------------|
| `users` | Core user data |
| `favorite_addresses` | Favorite address v1 |
| `favorite_addresses_v2` | Enhanced favorite addresses |
| `payment_method_type` | Payment method master data |
| `payment_method_user` | User's payment methods |
| `users_location` | User's last location |
| `navigation_user_state` | User navigation states |
| `user_verification` | Email/phone verification |
| `login_soft_banned` | Login attempt tracking |

---

## 🔗 Related Documentation

- [[api-reference|API Reference]]
- [[dependencies|Dependencies]]
- [[02-Work/Teams/MRG/00-overview/README|MRG Team Overview]]
- [[02-Work/Teams/MRG/02-services/authservice/README|Auth Service]] (upstream)

---

## 🏷️ Tags

#mrg #service #userservice #user-management #profile #payment #grpc #documentation

---

*Last Updated*: 2025-01-05  
*Generated from*: Repository analysis
