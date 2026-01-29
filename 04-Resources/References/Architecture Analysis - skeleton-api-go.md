---
title: Architecture Analysis - skeleton-api-go
type: reference
tags:
  - architecture
  - clean-architecture
  - go
  - skeleton
  - microservices
repository: architect/my-skeleton/skeleton-api-go
created: '2025-01-23'
status: completed
---
# Architecture Analysis: skeleton-api-go

> **Repository**: https://git.bluebird.id/architect/my-skeleton/skeleton-api-go
> **Analysis Date**: 2025-01-23
> **Analyst**: Architecture Team

---

## Executive Summary

Skeleton `skeleton-api-go` menggunakan **Hybrid Clean Architecture + go-kit pattern**. Ini bukan pure Clean Architecture, tetapi merupakan adaptasi pragmatis yang cocok untuk microservices di environment Go.

| Metric | Value |
|--------|-------|
| **Primary Pattern** | Clean Architecture (80%) |
| **Secondary Influence** | go-kit framework pattern |
| **Hexagonal Elements** | Repository interfaces as "Ports" |
| **Compliance Level** | ⭐⭐⭐⭐ (4/5) |
| **Production Readiness** | ✅ Ready |

---

## Project Structure Overview

```
skeleton-api-go/
├── cmd/
│   ├── healthcheck/          # Health check binary
│   └── server/               # Main server binary
│
├── internal/
│   ├── config/               # Configuration management
│   ├── domain/               # 🔵 Layer 1: Entity (Core)
│   ├── usecase/              # 🟢 Layer 2: Business Logic
│   ├── endpoint/             # 🟡 Layer 2.5: Endpoint (go-kit)
│   ├── delivery/             # 🟠 Layer 3: Interface Adapters
│   │   ├── grpc/
│   │   ├── http/
│   │   └── messaging/
│   ├── repository/           # 🔴 Layer 4: Infrastructure
│   │   ├── interface/
│   │   ├── database/
│   │   ├── cache/
│   │   ├── grpc/
│   │   ├── rest/
│   │   └── mq/
│   ├── mapper/               # Data transformation
│   ├── validator/            # Validation logic
│   └── server/               # Server initialization
│
├── pkg/                      # Shared utilities
│   ├── apm/
│   ├── circuitbreaker/
│   ├── errors/
│   ├── featureflag/
│   ├── grpcclient/
│   ├── httpclient/
│   ├── httputil/
│   ├── logger/
│   ├── messaging/
│   ├── middleware/
│   ├── profiling/
│   ├── retry/
│   ├── sse/
│   └── tls/
│
├── proto/                    # Protocol Buffer definitions
├── migrations/               # Database migrations
├── deployments/              # Deployment configurations
├── tests/                    # Test files
└── tools/                    # Development tools
    └── generator/            # Code generator
```

---

## Layer Analysis

### Layer 1: Domain (Entity)

**Location**: `internal/domain/`

**Files**:
- `user.go` - User entity
- `pagination.go` - Pagination domain model

**Characteristics**:
- ✅ Pure data structures
- ✅ No external dependencies
- ✅ Contains validation tags
- ⚠️ No domain methods/behavior (anemic domain model)

**Code Sample**:
```go
// internal/domain/user.go
package domain

import "time"

type User struct {
    ID            string    `db:"id" validate:"omitempty,uuid4"`
    Username      string    `db:"username" validate:"required,min=3,max=50"`
    Email         string    `db:"email" validate:"required,email,max=100"`
    Password      string    `db:"password" validate:"required,min=8,max=255"`
    FullName      string    `db:"full_name" validate:"required,min=1,max=100"`
    Avatar        *string   `db:"avatar" validate:"omitempty,url,max=255"`
    Role          *string   `db:"role" validate:"omitempty,oneof=admin user"`
    IsActive      bool      `db:"is_active"`
    EmailVerified bool      `db:"email_verified"`
    CreatedAt     time.Time `db:"created_at"`
    UpdatedAt     time.Time `db:"updated_at"`
}
```

---

### Layer 2: Use Case (Application Business Rules)

**Location**: `internal/usecase/`

**Files**:
- `user_usecase.go` - Main usecase struct & helpers
- `user_usecase_interface.go` - Interface definition
- `add_user.go`, `delete_user.go`, etc. - Individual operations

**Characteristics**:
- ✅ Contains business logic
- ✅ Interface-based design
- ✅ Dependency injection ready
- ⚠️ Depends on concrete `*repository.Repository` (not interface)

**Code Sample**:
```go
// internal/usecase/user_usecase.go
package usecase

type UserUseCase struct {
    repo        *repository.Repository           // ⚠️ Concrete type
    validator   validator.UserValidatorInterface // ✅ Interface
    logger      *logger.Logger
    sseManager  SSEManager                       // ✅ Interface
    featureFlag FeatureFlagManager               // ✅ Interface
}

func NewUserUseCase(
    repo *repository.Repository,
    v validator.UserValidatorInterface,
    log *logger.Logger,
    sseManager SSEManager,
    featureFlag FeatureFlagManager,
) *UserUseCase {
    return &UserUseCase{
        repo:        repo,
        validator:   v,
        logger:      log,
        sseManager:  sseManager,
        featureFlag: featureFlag,
    }
}
```

---

### Layer 2.5: Endpoint (go-kit Pattern)

**Location**: `internal/endpoint/`

**Files**:
- `endpoint.go` - Base endpoint type definition
- `user_endpoint.go` - User-specific endpoints

**Characteristics**:
- ✅ Abstraction layer untuk transport-agnostic handlers
- ✅ Enables middleware chaining (logging, singleflight, etc.)
- ✅ Request/Response DTOs per operation
- ⚠️ Not part of standard Clean Architecture

**Code Sample**:
```go
// internal/endpoint/endpoint.go
package endpoint

import "context"

// Endpoint represents a single RPC method (go-kit style)
type Endpoint func(ctx context.Context, request interface{}) (response interface{}, err error)
```

```go
// internal/endpoint/user_endpoint.go
// Request/Response DTOs
type AddUserRequest struct {
    User *domain.User
}

type AddUserResponse struct {
    Err error
}

// Endpoint factory with middleware
func MakeUserEndpoints(uc usecase.UserUseCaseInterface) UserEndpoints {
    return UserEndpoints{
        AddUserEndpoint:     MakeAddUserEndpoint(uc),
        GetAllUsersEndpoint: withSingleflight(MakeGetAllUsersEndpoint(uc), GetAllUsersKey),
        // ...
    }
}
```

---

### Layer 3: Delivery (Interface Adapters - Inbound)

**Location**: `internal/delivery/`

**Subdirectories**:
- `grpc/` - gRPC service handlers
- `http/` - HTTP handlers
- `messaging/` - Message queue consumers

**Characteristics**:
- ✅ Transport-specific implementations
- ✅ Uses endpoint layer for business logic
- ✅ Handles protocol-specific concerns (marshaling, error codes)
- ✅ Uses mapper for DTO transformation

**Code Sample**:
```go
// internal/delivery/grpc/user_service.go
package grpc

type UserService struct {
    pb.UnimplementedUserGrpcServiceServer
    endpoints endpoint.UserEndpoints  // Depends on endpoint layer
    mapper    *mapper.UserMapper
    logger    *logger.Logger
}

func NewUserService(
    endpoints endpoint.UserEndpoints,
    m *mapper.UserMapper,
    log *logger.Logger,
) *UserService {
    return &UserService{
        endpoints: endpoints,
        mapper:    m,
        logger:    log,
    }
}
```

---

### Layer 4: Repository (Infrastructure - Outbound)

**Location**: `internal/repository/`

**Subdirectories**:
- `interface/` - Interface definitions (Ports)
- `database/` - SQL implementations
- `cache/` - Redis implementations
- `grpc/` - gRPC client implementations
- `rest/` - REST client implementations
- `mq/` - Message queue implementations

**Characteristics**:
- ✅ Interface-based design
- ✅ Multiple adapter implementations
- ⚠️ Interface definitions di sini (seharusnya di usecase layer untuk pure Clean Architecture)
- ⚠️ Aggregated Repository struct (potential God object)

**Interface Definitions**:
```go
// internal/repository/interface/interface.go
package interfaces

type DBReadWriter interface {
    io.Closer
    AddUser(ctx context.Context, user *domain.User) error
    GetAllUsers(ctx context.Context) ([]*domain.User, error)
    GetAllUsersPaginated(ctx context.Context, page, pageSize int) ([]*domain.User, int64, error)
    GetUserByID(ctx context.Context, id string) (*domain.User, error)
    UpdateUser(ctx context.Context, user *domain.User) error
    DeleteUser(ctx context.Context, id string) error
    SearchUsers(ctx context.Context, query string) ([]*domain.User, error)
}

type CacheReadWriter interface {
    CacheReader
    CacheWriter
}

type ConsumerManager interface {
    RegisterConsumer(topic string, subscription string, consumer messaging.Consumer) error
    RegisterConsumerWithConfig(topic string, subscription string, consumer messaging.Consumer, conf messaging.Config) error
    Close() error
}
```

**Aggregated Repository**:
```go
// internal/repository/repository.go
package repository

type Repository struct {
    DB               *sql.DB
    DBReadWriter     interfaces.DBReadWriter
    Cache            interfaces.CacheReadWriter
    CacheReplica     interfaces.CacheReader
    MessagePublisher interfaces.MessagePublisher
    GrpcRepository   interfaces.GrpcRepository
    RestRepository   interfaces.RestRepository
    ConsumerManager  interfaces.ConsumerManager
    Logger           *logger.Logger
    QueryTimeout     time.Duration
    io.Closer
}
```

---

## Dependency Flow Diagram

```
┌─────────────────────────────────────────────────────────────────────────────────┐
│                                                                                 │
│   EXTERNAL WORLD                                                                │
│   ═══════════════                                                               │
│        │                                                                        │
│        ▼                                                                        │
│   ┌─────────────────────────────────────────────────────────────────────────┐   │
│   │                         DELIVERY LAYER                                  │   │
│   │  ┌──────────┐    ┌──────────┐    ┌──────────────┐                       │   │
│   │  │   gRPC   │    │   HTTP   │    │  Messaging   │                       │   │
│   │  │ Handlers │    │ Handlers │    │  Consumers   │                       │   │
│   │  └────┬─────┘    └────┬─────┘    └──────┬───────┘                       │   │
│   └───────┼───────────────┼─────────────────┼───────────────────────────────┘   │
│           │               │                 │                                   │
│           └───────────────┼─────────────────┘                                   │
│                           │                                                     │
│                           ▼                                                     │
│   ┌─────────────────────────────────────────────────────────────────────────┐   │
│   │                         ENDPOINT LAYER (go-kit)                         │   │
│   │                                                                         │   │
│   │   ┌─────────────────────────────────────────────────────────────────┐   │   │
│   │   │  Middleware Chain: Logging → Singleflight → Validation → ...   │   │   │
│   │   └─────────────────────────────────────────────────────────────────┘   │   │
│   │                                                                         │   │
│   │   func(ctx, request) → (response, error)                                │   │
│   │                                                                         │   │
│   └───────────────────────────────┬─────────────────────────────────────────┘   │
│                                   │                                             │
│                                   ▼                                             │
│   ┌─────────────────────────────────────────────────────────────────────────┐   │
│   │                         USECASE LAYER                                   │   │
│   │                                                                         │   │
│   │   ┌───────────────┐    ┌───────────────┐    ┌───────────────┐           │   │
│   │   │  UserUseCase  │    │   Validator   │    │    Mapper     │           │   │
│   │   │               │    │               │    │               │           │   │
│   │   │ - Add()       │    │ - Validate()  │    │ - ToProto()   │           │   │
│   │   │ - GetAll()    │    │               │    │ - ToDomain()  │           │   │
│   │   │ - Update()    │    │               │    │               │           │   │
│   │   │ - Delete()    │    │               │    │               │           │   │
│   │   └───────┬───────┘    └───────────────┘    └───────────────┘           │   │
│   │           │                                                             │   │
│   └───────────┼─────────────────────────────────────────────────────────────┘   │
│               │                                                                 │
│               │ depends on interfaces                                           │
│               ▼                                                                 │
│   ┌─────────────────────────────────────────────────────────────────────────┐   │
│   │                      REPOSITORY INTERFACES                              │   │
│   │                                                                         │   │
│   │   ┌─────────────────┐  ┌─────────────────┐  ┌─────────────────┐         │   │
│   │   │  DBReadWriter   │  │ CacheReadWriter │  │ MessagePublisher│         │   │
│   │   └────────┬────────┘  └────────┬────────┘  └────────┬────────┘         │   │
│   └────────────┼────────────────────┼────────────────────┼──────────────────┘   │
│                │                    │                    │                      │
│                │ implemented by     │                    │                      │
│                ▼                    ▼                    ▼                      │
│   ┌─────────────────────────────────────────────────────────────────────────┐   │
│   │                      REPOSITORY IMPLEMENTATIONS                         │   │
│   │                                                                         │   │
│   │   ┌──────────┐  ┌──────────┐  ┌──────────┐  ┌──────────┐  ┌──────────┐  │   │
│   │   │ database │  │  cache   │  │   grpc   │  │   rest   │  │    mq    │  │   │
│   │   │ (SQL)    │  │ (Redis)  │  │ (client) │  │ (client) │  │(RabbitMQ)│  │   │
│   │   └────┬─────┘  └────┬─────┘  └────┬─────┘  └────┬─────┘  └────┬─────┘  │   │
│   └────────┼─────────────┼─────────────┼─────────────┼─────────────┼────────┘   │
│            │             │             │             │             │            │
│            ▼             ▼             ▼             ▼             ▼            │
│   ┌─────────────────────────────────────────────────────────────────────────┐   │
│   │                      EXTERNAL SYSTEMS                                   │   │
│   │                                                                         │   │
│   │   PostgreSQL      Redis       gRPC Services    REST APIs     RabbitMQ   │   │
│   │                                                                         │   │
│   └─────────────────────────────────────────────────────────────────────────┘   │
│                                                                                 │
└─────────────────────────────────────────────────────────────────────────────────┘

LEGEND:
═══════
─────►  : Dependency direction
Layer   : Outer layers depend on inner layers
```

---

## Architecture Pattern Comparison

### Mapping to Clean Architecture

| Skeleton Component               | Clean Architecture Layer | Compliance   |
| -------------------------------- | ------------------------ | ------------ |
| `internal/domain/`               | Entity                   | ✅ Full       |
| `internal/usecase/`              | Use Cases                | ✅ Full       |
| `internal/endpoint/`             | *(Not in Clean)*         | ➕ Extension  |
| `internal/delivery/`             | Interface Adapters       | ✅ Full       |
| `internal/repository/`           | Frameworks & Drivers     | ✅ Full       |
| `internal/repository/interface/` | *(Should be in usecase)* | ⚠️ Deviation |
| `internal/mapper/`               | Interface Adapters       | ✅ Full       |
| `internal/validator/`            | *(Could be in usecase)*  | ⚠️ Separate  |

### Mapping to Hexagonal Architecture

| Skeleton Component | Hexagonal Element | Compliance |
|--------------------|-------------------|------------|
| `internal/domain/` | Core Domain | ✅ Full |
| `internal/usecase/` | Application Services | ✅ Full |
| `internal/delivery/` | Primary/Driving Adapters | ✅ Full |
| `internal/repository/interface/` | Ports (Driven) | ✅ Full |
| `internal/repository/*` | Secondary/Driven Adapters | ✅ Full |

---

## Deviations from Pure Clean Architecture

### 1. Interface Location

| Aspect | Pure Clean | Skeleton | Impact |
|--------|------------|----------|--------|
| **Location** | Usecase layer defines interfaces | Repository layer defines interfaces | Medium |
| **Rationale** | "The inner layer shouldn't know about outer layer" | Practical grouping | - |
| **Risk** | - | Tighter coupling, harder to swap repositories | ⚠️ |

**Recommendation**: Consider moving interfaces to usecase layer for stricter adherence.

### 2. Endpoint Layer (go-kit Pattern)

| Aspect | Pure Clean | Skeleton | Impact |
|--------|------------|----------|--------|
| **Existence** | Not present | Present | Low |
| **Purpose** | - | Middleware chain, transport abstraction | ✅ Positive |
| **Benefit** | - | Singleflight, logging, metrics | ✅ Positive |

**Recommendation**: Keep this pattern - it's a good extension for Go microservices.

### 3. Aggregated Repository Struct

| Aspect      | Pure Clean                      | Skeleton                     | Impact |
| ----------- | ------------------------------- | ---------------------------- | ------ |
| **Pattern** | Per-entity repository interface | Single aggregated struct     | Medium |
| **Risk**    | -                               | God object, harder testing   | ⚠️     |
| **Benefit** | -                               | Simpler dependency injection | ✅      |

**Recommendation**: Monitor for complexity growth. Consider splitting if needed.

### 4. Usecase Dependencies

| Aspect | Pure Clean | Skeleton | Impact |
|--------|------------|----------|--------|
| **Repository** | Interface only | Concrete `*Repository` | High |
| **Other deps** | Interface | Interface (correct) | ✅ |

**Recommendation**: Change `*repository.Repository` to interface for better testability.

---

## Strengths

### ✅ Good Practices Identified

1. **Clear Layer Separation**
   - Each layer has distinct responsibilities
   - Minimal cross-layer dependencies

2. **Interface-Based Design**
   - Repository interfaces enable mocking
   - Usecase interfaces for flexibility

3. **go-kit Endpoint Pattern**
   - Enables powerful middleware chain
   - Singleflight already implemented
   - Transport-agnostic handlers

4. **Comprehensive Repository Support**
   - Database (SQL)
   - Cache (Redis with replica support)
   - Message Queue (RabbitMQ, Kafka, MQTT, PubSub)
   - gRPC client
   - REST client

5. **Production-Ready Utilities** (`pkg/`)
   - APM integration
   - Circuit breaker
   - Feature flags
   - Retry mechanism
   - SSE support
   - TLS configuration

6. **Code Generation Support**
   - `tools/generator/` for scaffolding
   - Proto-based service generation

---

## Weaknesses & Recommendations

### ⚠️ Areas for Improvement

| Issue | Current State | Recommendation | Priority |
|-------|---------------|----------------|----------|
| Interface location | In repository layer | Move to usecase layer | Medium |
| Concrete repository dependency | `*Repository` in usecase | Use interface instead | High |
| Anemic domain model | Entity = data only | Add domain methods | Low |
| Aggregated repository | Single struct | Consider splitting per-aggregate | Medium |

---

## Usage Guidelines

### When to Use This Skeleton

✅ **Recommended For**:
- New microservices in Bluebird ecosystem
- Services requiring multiple transports (gRPC + HTTP + MQ)
- Services needing caching, feature flags, APM
- Teams familiar with go-kit patterns

⚠️ **Consider Alternatives For**:
- Simple CRUD services (might be overkill)
- Pure domain-driven services (consider full DDD approach)
- Services requiring strict Clean Architecture compliance

### How to Extend

1. **Adding New Entity**
   ```
   1. Create domain model: internal/domain/order.go
   2. Create usecase: internal/usecase/order_usecase.go
   3. Create endpoints: internal/endpoint/order_endpoint.go
   4. Create delivery: internal/delivery/grpc/order_service.go
   5. Create repository: internal/repository/database/order_*.go
   6. Wire in server: internal/server/server.go
   ```

2. **Adding New Repository Type**
   ```
   1. Define interface: internal/repository/interface/new_repo.go
   2. Implement adapter: internal/repository/newadapter/implementation.go
   3. Add to Repository struct: internal/repository/repository.go
   4. Initialize in factory: create RepoFactory implementation
   ```

---

## Conclusion

`skeleton-api-go` adalah **Hybrid Clean Architecture** yang pragmatis dan production-ready untuk microservices Go di Bluebird. Meskipun ada beberapa deviasi dari pure Clean Architecture, pattern ini memberikan keseimbangan yang baik antara:

- **Maintainability**: Clear layer separation
- **Testability**: Interface-based design
- **Flexibility**: Multiple adapter support
- **Productivity**: Code generation & comprehensive utilities

**Final Verdict**: ✅ **Recommended for adoption** dengan awareness terhadap deviasi yang ada.

---

## References

- [Clean Architecture by Robert C. Martin](https://blog.cleancoder.com/uncle-bob/2012/08/13/the-clean-architecture.html)
- [go-kit](https://gokit.io/)
- [Hexagonal Architecture](https://alistair.cockburn.us/hexagonal-architecture/)

---

## Document History

| Version | Date | Author | Changes |
|---------|------|--------|---------|
| 1.0 | 2025-01-23 | Architecture Team | Initial analysis |
