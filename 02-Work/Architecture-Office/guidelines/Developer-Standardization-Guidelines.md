---
title: Developer Standardization Guidelines
type: guidelines
status: draft
owner: Architecture Team
created: '2026-01-28'
version: '1.0'
---
# Developer Standardization Guidelines

> **Purpose:** Dokumen standarisasi development untuk seluruh tim developer Bluebird
> **Status:** Draft
> **Owner:** Architecture Team
> **Created:** 2026-01-28
> **Source:** Merged dari Development-Guidelines-Discussion-Points.md & Database-Access-Best-Practices-Discussion.md

---

## 📋 Status Keputusan Overview

| Group | Nama | Status | PIC |
|-------|------|--------|-----|
| A | Foundation | ⬜ Pending | - |
| B | Coding Standards | ⬜ Pending | - |
| C | API Design | ⬜ Pending | - |
| D | Data Layer | ⬜ Pending | - |
| E | Testing | ⬜ Pending | - |
| F | Observability | ⬜ Pending | - |
| G | Security | ⬜ Pending | - |
| H | Workflow | ⬜ Pending | - |
| I | Documentation | ⬜ Pending | - |
| J | Advanced Patterns | ⬜ Pending | - |

---

# GROUP A: FOUNDATION

> **Keperluan:** Setup project baru, struktur repository, konektifitas antar repo

---

## A1. Project Layout Standard

| # | Pertanyaan | Opsi | Keputusan |
|---|------------|------|-----------|
| A1.1 | Apakah perlu enforce standard project layout across squads? | a) Ya, wajib semua sama<br>b) Tidak, fleksibel per squad<br>c) Hybrid: core structure sama, detail fleksibel | C |
| A1.2 | Jika ya, layout mana yang jadi standard? | a) golang-standards/project-layout<br>b) Custom Bluebird layout<br>c) Adopt dari squad tertentu yang sudah mature | B |
| A1.3 | Apakah perlu ada skeleton generator/template? | a) Ya, buat CLI generator<br>b) Ya, template repository<br>c) Tidak perlu | A |

---

## A2. Package Organization

| # | Pertanyaan | Opsi | Keputusan |
|---|------------|------|-----------|
| A2.1 | Architecture pattern yang direkomendasikan? | a) Clean Architecture<br>b) Hexagonal Architecture<br>c) Domain-Driven Design<br>d) Fleksibel, tidak di-enforce | A |
| A2.2 | Internal package structure? | a) By layer (handler, service, repository)<br>b) By domain/feature<br>c) Hybrid | C |
| A2.3 | Shared/common package handling? | a) Monorepo shared module<br>b) Private Go module terpisah<br>c) Copy-paste allowed | A |

---

## A3. Naming Conventions

| # | Pertanyaan | Opsi | Keputusan |
|---|------------|------|-----------|
| A3.1 | File naming convention? | a) snake_case.go<br>b) lowercase.go<br>c) Tidak di-enforce | A |
| A3.2 | Package naming convention? | a) Single word lowercase (strict)<br>b) Allow multi-word dengan underscore<br>c) Follow Go standard saja | A |
| A3.3 | Interface naming? | a) Prefix dengan I (IUserService)<br>b) Suffix dengan -er (UserReader)<br>c) Tanpa prefix/suffix (UserService) | |

---

## A4. Repository Strategy

| # | Pertanyaan | Opsi | Keputusan |
|---|------------|------|-----------|
| A4.1 | Monorepo vs Multi-repo? | a) Monorepo per squad<br>b) Multi-repo per service<br>c) Hybrid (core monorepo + satellite repos)<br>d) Tidak di-enforce | |
| A4.2 | Jika monorepo, tooling apa yang dipakai? | a) Go workspace<br>b) Bazel<br>c) Makefile based<br>d) Custom tooling | |

---

## A5. Repository Connectivity

| # | Pertanyaan | Opsi | Keputusan |
|---|------------|------|-----------|
| A5.1 | Private Go module hosting? | a) GitLab/GitHub private<br>b) Artifactory<br>c) Athens proxy<br>d) Direct git clone | |
| A5.2 | Module versioning strategy? | a) Semantic versioning strict<br>b) Git tags<br>c) Commit hash<br>d) Branch based | |
| A5.3 | Dependency update policy? | a) Automated (Dependabot/Renovate)<br>b) Manual periodic<br>c) On-demand | |
| A5.4 | Shared library breaking change handling? | a) Major version bump wajib<br>b) Deprecation period<br>c) Per-case decision | |
| A5.5 | Internal module authentication? | a) SSH key<br>b) Personal access token<br>c) Deploy token<br>d) CI/CD managed | |

---

# GROUP B: CODING STANDARDS

> **Keperluan:** Day-to-day coding, standar penulisan code

---

## B1. Error Handling

| # | Pertanyaan | Opsi | Keputusan |
|---|------------|------|-----------|
| B1.1 | Error handling approach? | a) Standard errors + fmt.Errorf wrapping<br>b) pkg/errors (deprecated)<br>c) Custom error package internal<br>d) errors.Join (Go 1.20+) | |
| B1.2 | Apakah perlu custom error types? | a) Ya, dengan error codes<br>b) Ya, dengan error categories<br>c) Tidak, standard error cukup | |
| B1.3 | Error wrapping mandatory? | a) Ya, wajib wrap dengan context<br>b) Tidak, opsional<br>c) Wajib di boundary layer saja | |
| B1.4 | Sentinel errors standard? | a) Definisikan di package errors internal<br>b) Per-package sentinel errors<br>c) Tidak pakai sentinel errors | |
| B1.5 | Stack trace di error? | a) Ya, selalu include<br>b) Hanya di development<br>c) Tidak perlu | |

---

## B2. Logging

| # | Pertanyaan | Opsi | Keputusan |
|---|------------|------|-----------|
| B2.1 | Logging library standard? | a) zerolog<br>b) zap<br>c) slog (Go 1.21+)<br>d) logrus<br>e) Fleksibel per squad | |
| B2.2 | Log format? | a) JSON (structured)<br>b) Text (human readable)<br>c) JSON di production, text di development | |
| B2.3 | Log levels yang dipakai? | a) DEBUG, INFO, WARN, ERROR, FATAL<br>b) TRACE, DEBUG, INFO, WARN, ERROR<br>c) Custom levels | |
| B2.4 | Mandatory fields di setiap log? | a) timestamp, level, message saja<br>b) + request_id, service_name<br>c) + user_id, trace_id<br>d) Custom per squad | |
| B2.5 | Sensitive data di log? | a) Strict no PII<br>b) Masked/redacted<br>c) Allowed di debug level | |
| B2.6 | Log sampling di production? | a) Ya, dengan rate<br>b) Tidak, log semua<br>c) Sampling untuk DEBUG saja | |

---

## B3. Configuration Management

| # | Pertanyaan | Opsi | Keputusan |
|---|------------|------|-----------|
| B3.1 | Config source priority? | a) Env vars only<br>b) File → Env vars<br>c) Env vars → File<br>d) Remote config (Consul/etcd) | |
| B3.2 | Config library? | a) viper<br>b) envconfig<br>c) koanf<br>d) Custom/standard library | |
| B3.3 | Config validation? | a) Validate di startup (fail fast)<br>b) Validate lazy<br>c) Tidak perlu validation | |
| B3.4 | Secret management? | a) Env vars<br>b) Vault (HashiCorp)<br>c) Cloud provider secrets manager<br>d) Kubernetes secrets | |
| B3.5 | Config struct approach? | a) Single global config struct<br>b) Per-package config<br>c) Hierarchical config | |

---

## B4. Context Usage

| # | Pertanyaan | Opsi | Keputusan |
|---|------------|------|-----------|
| B4.1 | Context sebagai first parameter? | a) Ya, wajib semua function<br>b) Wajib di public functions saja<br>c) Opsional | |
| B4.2 | Context value usage? | a) Hanya untuk request-scoped data<br>b) Boleh untuk DI<br>c) Strict: trace ID dan deadline saja | |
| B4.3 | Custom context keys? | a) Typed keys (unexported)<br>b) String keys dengan prefix<br>c) Shared package untuk keys | |
| B4.4 | Timeout/deadline standard? | a) Set di entry point saja<br>b) Setiap layer boleh set<br>c) Defined di config per operation | |

---

## B5. Concurrency Patterns

| # | Pertanyaan | Opsi | Keputusan |
|---|------------|------|-----------|
| B5.1 | Goroutine management? | a) errgroup standard<br>b) Custom worker pool<br>c) Fleksibel | |
| B5.2 | Channel vs Mutex preference? | a) Prefer channels<br>b) Prefer mutex untuk simple cases<br>c) Case by case | |
| B5.3 | Goroutine leak prevention? | a) Mandatory context cancellation<br>b) Linter enforcement<br>c) Code review saja | |
| B5.4 | Rate limiting pattern? | a) golang.org/x/time/rate<br>b) Custom implementation<br>c) External (Redis based) | |

---

## B6. Dependency Injection

| # | Pertanyaan | Opsi | Keputusan |
|---|------------|------|-----------|
| B6.1 | DI approach? | a) Manual constructor injection<br>b) Wire (Google)<br>c) fx (Uber)<br>d) dig (Uber) | |
| B6.2 | Interface definition location? | a) Di consumer package<br>b) Di provider package<br>c) Di shared package | |
| B6.3 | Constructor pattern? | a) New() returns struct<br>b) New() returns interface<br>c) Functional options pattern | |

---

# GROUP C: API DESIGN

> **Keperluan:** Service communication, API standards

---

## C1. gRPC Standards

| # | Pertanyaan | Opsi | Keputusan |
|---|------------|------|-----------|
| C1.1 | Proto file organization? | a) Single proto per service<br>b) Split by domain<br>c) Monorepo proto | |
| C1.2 | Proto versioning? | a) Package versioning (v1, v2)<br>b) Field deprecation only<br>c) URL path versioning | |
| C1.3 | Proto style guide? | a) Google API design guide<br>b) Uber style guide<br>c) Custom Bluebird style | |
| C1.4 | Proto linting? | a) buf<br>b) protolint<br>c) Tidak pakai linter | |
| C1.5 | Proto breaking change detection? | a) buf breaking<br>b) Manual review<br>c) Tidak di-enforce | |

---

## C2. REST API Standards

| # | Pertanyaan | Opsi | Keputusan |
|---|------------|------|-----------|
| C2.1 | REST masih dipakai? | a) Ya, untuk public API<br>b) Ya, untuk internal juga<br>c) Tidak, full gRPC | |
| C2.2 | REST framework? | a) gin<br>b) echo<br>c) chi<br>d) fiber<br>e) standard net/http | |
| C2.3 | API versioning? | a) URL path (/v1/)<br>b) Header based<br>c) Query param | |
| C2.4 | Response format? | a) JSON:API spec<br>b) Custom envelope<br>c) Plain JSON | |

---

## C3. Error Response Standards

| # | Pertanyaan | Opsi | Keputusan |
|---|------------|------|-----------|
| C3.1 | gRPC error codes mapping? | a) Standard gRPC codes saja<br>b) Custom codes di details<br>c) Custom error proto | |
| C3.2 | Error response structure? | a) Google error model<br>b) RFC 7807 (Problem Details)<br>c) Custom structure | |
| C3.3 | Error codes registry? | a) Centralized di satu repo<br>b) Per-service codes<br>c) Tidak pakai codes | |
| C3.4 | Localized error messages? | a) Ya, dengan i18n<br>b) English only<br>c) Code saja, client handle message | |

---

## C4. API Documentation

| # | Pertanyaan | Opsi | Keputusan |
|---|------------|------|-----------|
| C4.1 | gRPC documentation? | a) Proto comments saja<br>b) Proto comments + generated docs<br>c) Separate documentation | |
| C4.2 | REST documentation? | a) OpenAPI/Swagger auto-generated<br>b) Manual OpenAPI spec<br>c) Tidak ada formal docs | |
| C4.3 | API changelog? | a) CHANGELOG.md per service<br>b) Git tags saja<br>c) Centralized changelog | |

---

# GROUP D: DATA LAYER

> **Keperluan:** Database access, caching, query optimization

---

## D1. Connection Management

### D1.1 Connection Pooling

| # | Pertanyaan | Opsi | Keputusan |
|---|------------|------|-----------|
| D1.1.1 | Siapa yang manage connection pool? | a) Setiap service manage sendiri<br>b) Shared library/package internal<br>c) Sidecar/proxy (PgBouncer, ProxySQL) | |
| D1.1.2 | Default max open connections? | a) 10<br>b) 25<br>c) 50<br>d) Based on formula (CPU * 2 + 1)<br>e) Per-service tuning | |
| D1.1.3 | Default max idle connections? | a) Same as max open<br>b) Half of max open<br>c) Fixed number (5-10)<br>d) Per-service tuning | |
| D1.1.4 | Connection max lifetime? | a) 5 minutes<br>b) 30 minutes<br>c) 1 hour<br>d) No limit<br>e) Configurable | |
| D1.1.5 | Connection max idle time? | a) 1 minute<br>b) 5 minutes<br>c) 10 minutes<br>d) No limit | |
| D1.1.6 | Bagaimana handle connection exhaustion? | a) Block dan wait<br>b) Return error immediately<br>c) Wait dengan timeout<br>d) Auto-scale pool | |

### D1.2 Connection Initialization

| # | Pertanyaan | Opsi | Keputusan |
|---|------------|------|-----------|
| D1.2.1 | Kapan establish connection? | a) Lazy (on first query)<br>b) Eager (on startup)<br>c) Hybrid (min pool on startup) | |
| D1.2.2 | Health check connection saat startup? | a) Ya, wajib ping<br>b) Ya, dengan simple query<br>c) Tidak perlu | |
| D1.2.3 | Retry strategy saat connection gagal? | a) Exponential backoff<br>b) Fixed interval retry<br>c) Circuit breaker<br>d) Fail fast | |
| D1.2.4 | Max retry attempts? | a) 3 times<br>b) 5 times<br>c) Unlimited dengan backoff<br>d) Configurable | |

### D1.3 Read/Write Splitting

| # | Pertanyaan | Opsi | Keputusan |
|---|------------|------|-----------|
| D1.3.1 | Apakah perlu read replica? | a) Ya, wajib untuk production<br>b) Opsional per-service<br>c) Tidak perlu | |
| D1.3.2 | Bagaimana routing read/write? | a) Explicit di code (ReadDB/WriteDB)<br>b) Automatic detection (SELECT vs others)<br>c) Proxy level (ProxySQL/PgBouncer)<br>d) DNS based | |
| D1.3.3 | Consistency requirement untuk read? | a) Strong consistency (always primary)<br>b) Eventual consistency OK<br>c) Per-query decision<br>d) Configurable lag tolerance | |
| D1.3.4 | Handle replication lag? | a) Application level check<br>b) Proxy handles<br>c) Accept lag<br>d) Fallback to primary | |

### D1.4 Connection String & Credentials

| # | Pertanyaan | Opsi | Keputusan |
|---|------------|------|-----------|
| D1.4.1 | Format connection string? | a) Full DSN string<br>b) Separate params (host, port, user, etc)<br>c) URL format | |
| D1.4.2 | SSL/TLS requirement? | a) Wajib di semua environment<br>b) Wajib di production saja<br>c) Opsional | |
| D1.4.3 | SSL mode? | a) require<br>b) verify-ca<br>c) verify-full<br>d) Per-environment | |
| D1.4.4 | Credential rotation handling? | a) Restart service<br>b) Hot reload<br>c) Connection pool refresh<br>d) Vault dynamic secrets | |

---

## D2. Query Execution

### D2.1 Query Layer Architecture

```
┌─────────────────────────────────────────────────────────────────┐
│                         Handler/API Layer                        │
└─────────────────────────────────────────────────────────────────┘
                                 │
                                 ▼
┌─────────────────────────────────────────────────────────────────┐
│                         Service/UseCase Layer                    │
│  - Business logic                                                │
│  - Transaction boundary?                                         │
└─────────────────────────────────────────────────────────────────┘
                                 │
                                 ▼
┌─────────────────────────────────────────────────────────────────┐
│                         Repository Layer                         │
│  - Query construction                                            │
│  - Data mapping                                                  │
└─────────────────────────────────────────────────────────────────┘
                                 │
                                 ▼
┌─────────────────────────────────────────────────────────────────┐
│                      Database Access Layer                       │
│  - Connection management                                         │
│  - Query execution                                               │
│  - Logging & metrics                                             │
└─────────────────────────────────────────────────────────────────┘
                                 │
                                 ▼
┌─────────────────────────────────────────────────────────────────┐
│                           Database                               │
└─────────────────────────────────────────────────────────────────┘
```

| # | Pertanyaan | Opsi | Keputusan |
|---|------------|------|-----------|
| D2.1.1 | Layer mana yang interact dengan DB? | a) Repository layer saja<br>b) Repository + dedicated DB layer<br>c) Service boleh langsung ke DB | |
| D2.1.2 | Apakah perlu abstraction layer di atas driver? | a) Ya, wrapper internal<br>b) Tidak, pakai driver langsung<br>c) Pakai library (sqlx, etc) | |
| D2.1.3 | Query builder vs Raw SQL? | a) Raw SQL only<br>b) Query builder wajib<br>c) Raw untuk complex, builder untuk simple<br>d) Per-team decision | |
| D2.1.4 | SQL driver? | a) database/sql + driver<br>b) sqlx<br>c) pgx (Postgres specific)<br>d) Fleksibel | |
| D2.1.5 | ORM usage? | a) Tidak boleh ORM<br>b) GORM allowed<br>c) ent allowed<br>d) Fleksibel | |

### D2.2 Context Propagation

| # | Pertanyaan | Opsi | Keputusan |
|---|------------|------|-----------|
| D2.2.1 | Context wajib di semua DB operations? | a) Ya, wajib<br>b) Opsional<br>c) Wajib untuk queries, opsional untuk connection | |
| D2.2.2 | Apa yang harus ada di context untuk DB? | a) Timeout saja<br>b) Timeout + trace ID<br>c) Timeout + trace ID + user ID<br>d) Custom per-service | |
| D2.2.3 | Default query timeout? | a) 5 seconds<br>b) 10 seconds<br>c) 30 seconds<br>d) No default (use context)<br>e) Per-query type | |
| D2.2.4 | Handle context cancellation? | a) Graceful cancel query<br>b) Let query complete<br>c) Cancel + rollback if in transaction | |

### D2.3 Query Execution Pattern

| # | Pertanyaan | Opsi | Keputusan |
|---|------------|------|-----------|
| D2.3.1 | Prepared statements usage? | a) Wajib semua query<br>b) Auto-prepared by driver<br>c) Manual untuk frequent queries<br>d) Tidak pakai (query langsung) | |
| D2.3.2 | Parameterized query enforcement? | a) Wajib, no string concatenation<br>b) Linter check<br>c) Code review saja | |
| D2.3.3 | Batch operations pattern? | a) Single transaction batch<br>b) Bulk insert syntax<br>c) Parallel individual inserts<br>d) Per-case decision | |
| D2.3.4 | Pagination default pattern? | a) LIMIT OFFSET<br>b) Cursor/Keyset pagination<br>c) Hybrid based on data size<br>d) Per-case decision | |

### D2.4 Result Handling

| # | Pertanyaan | Opsi | Keputusan |
|---|------------|------|-----------|
| D2.4.1 | Row scanning pattern? | a) Struct scanning (sqlx)<br>b) Manual scan<br>c) Reflection-based<br>d) Code generation | |
| D2.4.2 | NULL handling? | a) sql.NullXxx types<br>b) Pointer fields<br>c) Custom null types<br>d) COALESCE di query | |
| D2.4.3 | Large result set handling? | a) Streaming with rows.Next()<br>b) Pagination mandatory<br>c) Limit result + warning<br>d) Per-case decision | |
| D2.4.4 | Empty result handling? | a) Return empty slice/nil<br>b) Return sentinel error<br>c) Return (result, found bool) | |

---

## D3. Transaction Management

### D3.1 Transaction Boundary

| # | Pertanyaan | Opsi | Keputusan |
|---|------------|------|-----------|
| D3.1.1 | Di layer mana transaction dimulai? | a) Service layer<br>b) Repository layer<br>c) Handler layer<br>d) Dedicated transaction layer | |
| D3.1.2 | Transaction pattern? | a) Closure-based (WithTransaction(func))<br>b) Manual Begin/Commit/Rollback<br>c) Unit of Work pattern<br>d) Decorator pattern | |
| D3.1.3 | Siapa yang handle rollback? | a) Defer di transaction starter<br>b) Explicit di setiap error<br>c) Framework/library handles | |
| D3.1.4 | Pilih pattern mana? | a) Closure-based<br>b) Manual<br>c) Fleksibel per-case | |

**Contoh Closure Pattern:**
```go
func (s *Service) TransferMoney(ctx context.Context, from, to string, amount int) error {
    return s.db.WithTransaction(ctx, func(tx *sql.Tx) error {
        if err := s.repo.Debit(ctx, tx, from, amount); err != nil {
            return err // auto rollback
        }
        if err := s.repo.Credit(ctx, tx, to, amount); err != nil {
            return err // auto rollback
        }
        return nil // auto commit
    })
}
```

**Contoh Manual Pattern:**
```go
func (s *Service) TransferMoney(ctx context.Context, from, to string, amount int) error {
    tx, err := s.db.BeginTx(ctx, nil)
    if err != nil {
        return err
    }
    defer tx.Rollback() // no-op if committed
    
    if err := s.repo.Debit(ctx, tx, from, amount); err != nil {
        return err
    }
    if err := s.repo.Credit(ctx, tx, to, amount); err != nil {
        return err
    }
    
    return tx.Commit()
}
```

### D3.2 Transaction Isolation

| # | Pertanyaan | Opsi | Keputusan |
|---|------------|------|-----------|
| D3.2.1 | Default isolation level? | a) Read Committed<br>b) Repeatable Read<br>c) Serializable<br>d) Database default | |
| D3.2.2 | Kapan pakai isolation level berbeda? | a) Documented per-case<br>b) Per-query annotation<br>c) Tidak boleh ubah default | |
| D3.2.3 | Read-only transaction? | a) Ya, untuk read-heavy operations<br>b) Tidak perlu<br>c) Per-case decision | |

### D3.3 Nested Transaction

| # | Pertanyaan | Opsi | Keputusan |
|---|------------|------|-----------|
| D3.3.1 | Support nested transaction? | a) Ya, dengan savepoints<br>b) Tidak support<br>c) Propagation (reuse existing tx) | |
| D3.3.2 | Bagaimana detect sudah dalam transaction? | a) Context value<br>b) Transaction manager<br>c) Interface check<br>d) Tidak perlu detect | |
| D3.3.3 | Pakai Querier interface pattern? | a) Ya, wajib<br>b) Ya, recommended<br>c) Tidak, repository pegang connection sendiri | |

**Contoh Querier Interface Pattern:**
```go
type Querier interface {
    ExecContext(ctx context.Context, query string, args ...interface{}) (sql.Result, error)
    QueryContext(ctx context.Context, query string, args ...interface{}) (*sql.Rows, error)
    QueryRowContext(ctx context.Context, query string, args ...interface{}) *sql.Row
}

func (r *Repository) Create(ctx context.Context, q Querier, data Entity) error {
    _, err := q.ExecContext(ctx, "INSERT INTO ...", ...)
    return err
}
```

### D3.4 Transaction Timeout & Limits

| # | Pertanyaan | Opsi | Keputusan |
|---|------------|------|-----------|
| D3.4.1 | Max transaction duration? | a) 5 seconds<br>b) 30 seconds<br>c) 1 minute<br>d) No limit<br>e) Configurable | |
| D3.4.2 | Lock wait timeout? | a) 5 seconds<br>b) 10 seconds<br>c) Database default<br>d) Configurable | |
| D3.4.3 | Deadlock retry? | a) Ya, automatic retry<br>b) Ya, dengan limit<br>c) Tidak, fail immediately | |
| D3.4.4 | Long transaction alerting? | a) Ya, alert > threshold<br>b) Log warning saja<br>c) Tidak perlu | |

---

## D4. Slow Query Detection

### D4.1 Definition & Threshold

| # | Pertanyaan | Opsi | Keputusan |
|---|------------|------|-----------|
| D4.1.1 | Definisi slow query? | a) > 100ms<br>b) > 500ms<br>c) > 1s<br>d) Tiered (warning: 500ms, critical: 2s)<br>e) Per-query type | |
| D4.1.2 | Apakah threshold berbeda per environment? | a) Ya, lebih strict di production<br>b) Ya, lebih lenient di production<br>c) Sama semua environment | |
| D4.1.3 | Apakah threshold berbeda per query type? | a) Ya (SELECT vs INSERT vs UPDATE)<br>b) Ya (simple vs complex)<br>c) Tidak, sama semua | |

### D4.2 Detection Mechanism

```
┌─────────────────────────────────────────────────────────────────┐
│                     Slow Query Detection Flow                    │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│  ┌──────────┐    ┌──────────────┐    ┌──────────────────────┐   │
│  │  Query   │───▶│  Middleware/ │───▶│  Execute Query       │   │
│  │  Start   │    │  Interceptor │    │  + Measure Duration  │   │
│  └──────────┘    └──────────────┘    └──────────────────────┘   │
│                                                │                 │
│                                                ▼                 │
│                                       ┌──────────────────┐      │
│                                       │ Duration > Threshold?│   │
│                                       └──────────────────┘      │
│                                          │           │          │
│                                         Yes          No         │
│                                          │           │          │
│                                          ▼           ▼          │
│                               ┌─────────────────┐  ┌────────┐   │
│                               │ Log Slow Query  │  │ Normal │   │
│                               │ + Emit Metric   │  │  Log   │   │
│                               │ + Alert?        │  └────────┘   │
│                               └─────────────────┘               │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

| # | Pertanyaan | Opsi | Keputusan |
|---|------------|------|-----------|
| D4.2.1 | Di mana detection dilakukan? | a) Application level (middleware/interceptor)<br>b) Database level (pg_stat_statements)<br>c) APM tool<br>d) Semua level | |
| D4.2.2 | Bagaimana implement di application? | a) Custom middleware/wrapper<br>b) Library hooks (sqlx, pgx hooks)<br>c) OpenTelemetry instrumentation<br>d) AOP/decorator | |
| D4.2.3 | Apakah perlu capture query plan? | a) Ya, untuk slow queries<br>b) Ya, semua queries di development<br>c) Tidak perlu | |

### D4.3 Logging Slow Query

| # | Pertanyaan | Opsi | Keputusan |
|---|------------|------|-----------|
| D4.3.1 | Informasi apa yang di-log? | a) Query + duration saja<br>b) + Parameters<br>c) + Stack trace<br>d) + Query plan | |
| D4.3.2 | Parameter logging untuk slow query? | a) Full parameters<br>b) Masked/truncated<br>c) Tidak log parameters | |
| D4.3.3 | Log level untuk slow query? | a) WARN<br>b) ERROR<br>c) INFO dengan tag<br>d) Separate slow query log | |
| D4.3.4 | Format log slow query? | a) Structured JSON<br>b) Human readable<br>c) Both (configurable) | |

**Contoh Log Format:**
```json
{
  "level": "warn",
  "msg": "slow query detected",
  "query": "SELECT * FROM orders WHERE user_id = $1 AND status = $2",
  "duration_ms": 1523,
  "threshold_ms": 500,
  "params": ["user_123", "pending"],
  "rows_affected": 1500,
  "caller": "orderRepository.FindByUserAndStatus:45",
  "trace_id": "abc123",
  "service": "order-service"
}
```

| # | Pertanyaan | Opsi | Keputusan |
|---|------------|------|-----------|
| D4.3.5 | Apakah format di atas sudah cukup? | a) Ya<br>b) Perlu tambahan fields<br>c) Terlalu banyak, kurangi | |

### D4.4 Slow Query Metrics

| # | Pertanyaan | Opsi | Keputusan |
|---|------------|------|-----------|
| D4.4.1 | Metrics apa yang di-track? | a) Query count + duration histogram<br>b) + Error rate<br>c) + Rows affected<br>d) + Connection pool stats | |
| D4.4.2 | Metric labels/dimensions? | a) query_name, status<br>b) + table_name<br>c) + operation (SELECT/INSERT/etc)<br>d) Minimal labels | |
| D4.4.3 | Histogram buckets untuk duration? | a) 10ms, 50ms, 100ms, 500ms, 1s, 5s<br>b) 100ms, 500ms, 1s, 2s, 5s, 10s<br>c) Custom buckets<br>d) Default library buckets | |
| D4.4.4 | Metrics naming convention? | a) db_ prefix<br>b) sql_ prefix<br>c) {service}_ prefix<br>d) OpenTelemetry semantic conventions | |

**Contoh Metrics:**
```
# Query duration histogram
db_query_duration_seconds_bucket{query="GetOrderByID",le="0.1"} 9500
db_query_duration_seconds_bucket{query="GetOrderByID",le="0.5"} 9800
db_query_duration_seconds_bucket{query="GetOrderByID",le="1.0"} 9950
db_query_duration_seconds_bucket{query="GetOrderByID",le="+Inf"} 10000

# Slow query counter
db_slow_queries_total{query="GetOrderByID",threshold="500ms"} 200

# Connection pool
db_pool_open_connections 25
db_pool_in_use_connections 10
db_pool_idle_connections 15
db_pool_wait_count_total 50
db_pool_wait_duration_seconds_total 2.5
```

---

## D5. Query Optimization

### D5.1 Index Guidelines

| # | Pertanyaan | Opsi | Keputusan |
|---|------------|------|-----------|
| D5.1.1 | Index naming convention? | a) idx_{table}_{columns}<br>b) {table}_{columns}_idx<br>c) Tidak di-enforce | |
| D5.1.2 | Wajib index untuk foreign keys? | a) Ya, wajib<br>b) Recommended<br>c) Per-case decision | |
| D5.1.3 | Composite index guidelines? | a) Documented best practices<br>b) Review oleh DBA<br>c) Tidak ada guidelines | |
| D5.1.4 | Unused index monitoring? | a) Ya, regular audit<br>b) Tooling otomatis<br>c) Tidak monitor | |

### D5.2 Query Review Process

| # | Pertanyaan | Opsi | Keputusan |
|---|------------|------|-----------|
| D5.2.1 | EXPLAIN ANALYZE wajib untuk PR? | a) Ya, untuk semua query baru<br>b) Ya, untuk query complex saja<br>c) Recommended, tidak wajib<br>d) Tidak perlu | |
| D5.2.2 | N+1 query detection? | a) Linter/static analysis<br>b) Runtime detection + log<br>c) Code review saja<br>d) APM detection | |
| D5.2.3 | Query complexity limit? | a) Ya, max joins/subqueries<br>b) Ya, max execution time estimate<br>c) Tidak ada limit<br>d) Guidelines saja | |

---

## D6. Migration

| # | Pertanyaan | Opsi | Keputusan |
|---|------------|------|-----------|
| D6.1 | Migration tool? | a) golang-migrate<br>b) goose<br>c) atlas<br>d) Custom scripts | |
| D6.2 | Migration strategy? | a) Forward only<br>b) Up/Down wajib<br>c) Fleksibel | |
| D6.3 | Migration di CI/CD? | a) Auto apply di deployment<br>b) Manual approval<br>c) Separate pipeline | |
| D6.4 | Schema versioning? | a) Timestamp based<br>b) Sequential number<br>c) Semantic versioning | |
| D6.5 | Backward compatible migrations wajib? | a) Ya, selalu<br>b) Ya, untuk production<br>c) Tidak wajib | |
| D6.6 | Breaking change procedure? | a) Blue-green database<br>b) Expand-contract pattern<br>c) Downtime window<br>d) Per-case decision | |
| D6.7 | Large table migration strategy? | a) Online schema change (pt-osc, gh-ost)<br>b) Batch migration<br>c) Maintenance window<br>d) Per-case decision | |

**Contoh Expand-Contract Migration:**
```sql
-- Phase 1: Expand (add new column, keep old)
ALTER TABLE users ADD COLUMN email_new VARCHAR(255);

-- Phase 2: Migrate data
UPDATE users SET email_new = LOWER(email) WHERE email_new IS NULL;

-- Phase 3: Application code uses both columns (write to both, read from new)

-- Phase 4: Contract (remove old column) - after all instances updated
ALTER TABLE users DROP COLUMN email;
ALTER TABLE users RENAME COLUMN email_new TO email;
```

---

## D7. Repository Pattern

| # | Pertanyaan | Opsi | Keputusan |
|---|------------|------|-----------|
| D7.1 | Repository interface standard? | a) Generic repository<br>b) Per-entity repository<br>c) CQRS (separate read/write) | |
| D7.2 | Pagination pattern? | a) Offset-based<br>b) Cursor-based<br>c) Keyset pagination<br>d) Support both | |
| D7.3 | Soft delete handling? | a) Mandatory soft delete<br>b) Hard delete allowed<br>c) Per-entity decision | |

---

## D8. Redis Caching Strategy

### D8.1 Cache Pattern Selection

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                         Common Caching Patterns                              │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                              │
│  CACHE-ASIDE (Lazy Loading)              READ-THROUGH                        │
│  ┌───────┐      ┌───────┐                ┌───────┐      ┌───────┐           │
│  │  App  │─1───▶│ Cache │                │  App  │─1───▶│ Cache │           │
│  │       │◀─2───│       │                │       │◀─4───│   │   │           │
│  │       │─3───▶│  DB   │                │       │      │   2   │           │
│  │       │◀─4───│       │                │       │      │   ▼   │           │
│  └───────┘      └───────┘                └───────┘      │  DB   │           │
│  App manages cache                       Cache manages DB read               │
│                                                                              │
│  WRITE-THROUGH                           WRITE-BEHIND (Write-Back)          │
│  ┌───────┐      ┌───────┐                ┌───────┐      ┌───────┐           │
│  │  App  │─1───▶│ Cache │                │  App  │─1───▶│ Cache │           │
│  │       │◀─3───│   │   │                │       │◀─2───│   │   │           │
│  │       │      │   2   │                │       │      │ async │           │
│  │       │      │   ▼   │                │       │      │   ▼   │           │
│  └───────┘      │  DB   │                └───────┘      │  DB   │           │
│  Sync write to both                      Async write to DB                   │
│                                                                              │
└─────────────────────────────────────────────────────────────────────────────┘
```

| # | Pertanyaan | Opsi | Keputusan |
|---|------------|------|-----------|
| D8.1.1 | Default caching pattern? | a) Cache-Aside (most common)<br>b) Read-Through<br>c) Write-Through<br>d) Per-case decision | |
| D8.1.2 | Siapa yang manage cache logic? | a) Repository layer<br>b) Dedicated cache layer/service<br>c) Service layer<br>d) Decorator/middleware | |
| D8.1.3 | Cache library/client untuk Go? | a) go-redis/redis<br>b) redigo<br>c) rueidis (high performance)<br>d) Per-squad decision | |

### D8.2 Cache Key Design

| # | Pertanyaan | Opsi | Keputusan |
|---|------------|------|-----------|
| D8.2.1 | Key naming convention? | a) {service}:{entity}:{id}<br>b) {entity}:{id}:{version}<br>c) {prefix}:{hash}<br>d) Tidak di-enforce | |
| D8.2.2 | Key prefix per service? | a) Ya, wajib service prefix<br>b) Ya, wajib environment prefix<br>c) Opsional<br>d) Managed by Redis namespace | |
| D8.2.3 | Key length limit? | a) Max 100 characters<br>b) Max 200 characters<br>c) No limit<br>d) Hashed jika terlalu panjang | |
| D8.2.4 | Composite key handling? | a) Join dengan colon (:)<br>b) Join dengan hash<br>c) Serialize ke JSON lalu hash | |

**Contoh Key Naming:**
```
# Pattern: {service}:{entity}:{identifier}
order-service:order:12345
order-service:user-orders:user_abc:page_1

# Pattern dengan version/tenant
v1:tenant_xyz:order-service:order:12345

# Pattern untuk computed/aggregate data
order-service:stats:daily:2026-01-21
order-service:user:user_abc:order-count
```

| # | Pertanyaan | Opsi | Keputusan |
|---|------------|------|-----------|
| D8.2.5 | Apakah key naming convention di atas sudah sesuai? | a) Ya<br>b) Perlu adjustment<br>c) Buat yang berbeda | |

### D8.3 TTL Strategy

| # | Pertanyaan | Opsi | Keputusan |
|---|------------|------|-----------|
| D8.3.1 | Default TTL? | a) 5 minutes<br>b) 15 minutes<br>c) 1 hour<br>d) No default, wajib explicit<br>e) Per-entity type | |
| D8.3.2 | TTL untuk static/reference data? | a) 1 hour<br>b) 24 hours<br>c) 1 week<br>d) No expiry (manual invalidation) | |
| D8.3.3 | TTL untuk user session data? | a) 15 minutes<br>b) 30 minutes<br>c) Match session timeout<br>d) Sliding expiration | |
| D8.3.4 | TTL jitter untuk prevent thundering herd? | a) Ya, random ±10%<br>b) Ya, random ±20%<br>c) Tidak perlu | |
| D8.3.5 | Boleh cache tanpa TTL? | a) Tidak boleh<br>b) Boleh untuk specific cases (documented)<br>c) Boleh dengan approval | |

### D8.4 Cache Invalidation

| # | Pertanyaan | Opsi | Keputusan |
|---|------------|------|-----------|
| D8.4.1 | Primary invalidation strategy? | a) TTL-based (let it expire)<br>b) Event-driven (on data change)<br>c) Manual/explicit delete<br>d) Hybrid (TTL + event) | |
| D8.4.2 | Event-driven invalidation mechanism? | a) Publish ke Redis pub/sub<br>b) Message queue (Kafka, RabbitMQ)<br>c) Database triggers/CDC<br>d) Application level events | |
| D8.4.3 | Invalidation scope? | a) Single key delete<br>b) Pattern-based delete (SCAN + DEL)<br>c) Tag-based invalidation<br>d) Namespace flush | |
| D8.4.4 | Cross-service cache invalidation? | a) Event bus/message queue<br>b) Direct API call<br>c) Shared cache namespace<br>d) Tidak support (TTL saja) | |

**Contoh Event-Driven Invalidation:**
```go
func (s *OrderService) UpdateOrder(ctx context.Context, order *Order) error {
    // 1. Update database
    if err := s.repo.Update(ctx, order); err != nil {
        return err
    }
    
    // 2. Invalidate cache
    cacheKey := fmt.Sprintf("order-service:order:%s", order.ID)
    if err := s.cache.Delete(ctx, cacheKey); err != nil {
        s.logger.Warn("failed to invalidate cache", "key", cacheKey, "error", err)
    }
    
    // 3. Publish event untuk cross-service invalidation
    s.eventBus.Publish(ctx, "order.updated", order.ID)
    
    return nil
}
```

| # | Pertanyaan | Opsi | Keputusan |
|---|------------|------|-----------|
| D8.4.5 | Apakah pattern invalidation di atas sudah sesuai? | a) Ya<br>b) Perlu adjustment<br>c) Buat yang berbeda | |

### D8.5 Serialization

| # | Pertanyaan | Opsi | Keputusan |
|---|------------|------|-----------|
| D8.5.1 | Serialization format? | a) JSON<br>b) MessagePack<br>c) Protocol Buffers<br>d) Gob (Go native)<br>e) Per-case decision | |
| D8.5.2 | Compression untuk large values? | a) Ya, always compress<br>b) Ya, jika > X KB<br>c) Tidak perlu<br>d) Per-case decision | |
| D8.5.3 | Schema versioning di cache? | a) Include version di value<br>b) Include version di key<br>c) Tidak perlu (TTL handles) | |

### D8.6 Redis Connection Management

| # | Pertanyaan | Opsi | Keputusan |
|---|------------|------|-----------|
| D8.6.1 | Connection pooling? | a) Ya, dengan pool size limit<br>b) Single connection<br>c) Library default | |
| D8.6.2 | Default pool size? | a) 10<br>b) 20<br>c) CPU * 2<br>d) Per-service tuning | |
| D8.6.3 | Connection timeout? | a) 1 second<br>b) 5 seconds<br>c) 10 seconds<br>d) Configurable | |
| D8.6.4 | Read/Write timeout? | a) 1 second<br>b) 3 seconds<br>c) 5 seconds<br>d) Per-operation | |
| D8.6.5 | Retry strategy? | a) Exponential backoff<br>b) Fixed retry<br>c) No retry (fail fast)<br>d) Circuit breaker | |

### D8.7 Redis Data Structures

| # | Pertanyaan | Opsi | Keputusan |
|---|------------|------|-----------|
| D8.7.1 | Kapan pakai String vs Hash? | a) Guidelines documented<br>b) String untuk simple, Hash untuk objects<br>c) Per-case decision | |
| D8.7.2 | Kapan pakai List vs Sorted Set? | a) Guidelines documented<br>b) List untuk queue, ZSet untuk ranking<br>c) Per-case decision | |
| D8.7.3 | Boleh pakai Redis Streams? | a) Ya, untuk event streaming<br>b) Tidak, pakai dedicated MQ<br>c) Per-case dengan approval | |
| D8.7.4 | Boleh pakai Redis as primary datastore? | a) Tidak boleh<br>b) Boleh untuk ephemeral data<br>c) Per-case dengan approval | |

**Guidelines Data Structure:**
```
STRING  → Simple key-value, counters, serialized objects
HASH    → Object dengan multiple fields, partial update needed
LIST    → Queue, recent items, activity feed
SET     → Unique items, tags, membership check
ZSET    → Leaderboard, time-series, rate limiting
STREAM  → Event log, message queue (if approved)
```

| # | Pertanyaan | Opsi | Keputusan |
|---|------------|------|-----------|
| D8.7.5 | Apakah guidelines data structure di atas sudah cukup? | a) Ya<br>b) Perlu detail lebih | |

### D8.8 Cache Failure Handling

| # | Pertanyaan | Opsi | Keputusan |
|---|------------|------|-----------|
| D8.8.1 | Behavior saat Redis down? | a) Fallback ke database<br>b) Return error<br>c) Serve stale data (if available)<br>d) Circuit breaker + fallback | |
| D8.8.2 | Cache miss handling? | a) Query DB, populate cache<br>b) Query DB, async populate<br>c) Query DB only (no populate) | |
| D8.8.3 | Thundering herd prevention? | a) Distributed lock (single populate)<br>b) Request coalescing<br>c) Probabilistic early expiration<br>d) Tidak di-handle | |
| D8.8.4 | Cache warming on startup? | a) Ya, critical data<br>b) Tidak, lazy load<br>c) Background warming | |
| D8.8.5 | Apakah perlu circuit breaker untuk cache? | a) Ya, wajib<br>b) Recommended<br>c) Tidak perlu | |

**Contoh Circuit Breaker Pattern:**
```go
type CachedRepository struct {
    repo    Repository
    cache   Cache
    breaker *circuitbreaker.Breaker
}

func (r *CachedRepository) GetOrder(ctx context.Context, id string) (*Order, error) {
    // Try cache first (with circuit breaker)
    if r.breaker.Allow() {
        order, err := r.cache.Get(ctx, "order:"+id)
        if err == nil && order != nil {
            return order, nil
        }
        if err != nil {
            r.breaker.RecordFailure()
        }
    }
    
    // Fallback to database
    order, err := r.repo.GetOrder(ctx, id)
    if err != nil {
        return nil, err
    }
    
    // Async cache population (if breaker allows)
    if r.breaker.Allow() {
        go r.cache.Set(ctx, "order:"+id, order, 15*time.Minute)
    }
    
    return order, nil
}
```

### D8.9 Cache Metrics

| # | Pertanyaan | Opsi | Keputusan |
|---|------------|------|-----------|
| D8.9.1 | Metrics yang di-track? | a) Hit/miss ratio saja<br>b) + Latency<br>c) + Error rate<br>d) + Memory usage | |
| D8.9.2 | Alert untuk cache hit ratio? | a) Ya, jika < 80%<br>b) Ya, jika < 90%<br>c) Monitor saja, no alert | |
| D8.9.3 | Alert untuk cache latency? | a) Ya, P99 > threshold<br>b) Monitor saja<br>c) Tidak perlu | |

**Contoh Cache Metrics:**
```
# Hit/Miss
cache_requests_total{result="hit",cache="order"} 95000
cache_requests_total{result="miss",cache="order"} 5000

# Latency
cache_operation_duration_seconds_bucket{operation="get",le="0.001"} 90000
cache_operation_duration_seconds_bucket{operation="get",le="0.01"} 98000
cache_operation_duration_seconds_bucket{operation="get",le="0.1"} 100000

# Errors
cache_errors_total{operation="get",error="timeout"} 50
cache_errors_total{operation="get",error="connection"} 10
```

---

# GROUP E: TESTING

> **Keperluan:** Quality assurance, test standards

---

## E1. Unit Testing

| # | Pertanyaan | Opsi | Keputusan |
|---|------------|------|-----------|
| E1.1 | Test framework? | a) Standard testing package<br>b) testify<br>c) ginkgo/gomega<br>d) goconvey | |
| E1.2 | Assertion library? | a) testify/assert<br>b) testify/require<br>c) Standard + custom helpers<br>d) go-cmp | |
| E1.3 | Test file location? | a) Same package (*_test.go)<br>b) Separate _test package<br>c) Hybrid berdasarkan test type | |
| E1.4 | Table-driven tests? | a) Wajib untuk multiple cases<br>b) Recommended<br>c) Opsional | |
| E1.5 | Coverage minimum? | a) 80%<br>b) 70%<br>c) 60%<br>d) Tidak di-enforce | |

---

## E2. Mocking

| # | Pertanyaan | Opsi | Keputusan |
|---|------------|------|-----------|
| E2.1 | Mock generation tool? | a) mockgen (gomock)<br>b) mockery<br>c) moq<br>d) Manual mocks | |
| E2.2 | Mock location? | a) Same package<br>b) mocks/ subdirectory<br>c) internal/mocks/ | |
| E2.3 | Mock naming convention? | a) Mock prefix (MockUserService)<br>b) mock_ prefix file<br>c) _mock suffix | |
| E2.4 | External service mocking? | a) Interface + mock<br>b) httptest<br>c) Testcontainers<br>d) WireMock/MockServer | |

---

## E3. Integration Testing

| # | Pertanyaan | Opsi | Keputusan |
|---|------------|------|-----------|
| E3.1 | Database testing? | a) Testcontainers<br>b) Embedded database<br>c) Shared test database<br>d) SQLite for tests | |
| E3.2 | Test data management? | a) Fixtures<br>b) Factory/Builder pattern<br>c) Seeds<br>d) Fresh per test | |
| E3.3 | Integration test tagging? | a) Build tags (// +build integration)<br>b) Naming convention (_integration_test.go)<br>c) Separate directory | |
| E3.4 | External service testing? | a) Real services di staging<br>b) Mock servers<br>c) Contract testing (Pact) | |

---

## E4. Test Execution

| # | Pertanyaan | Opsi | Keputusan |
|---|------------|------|-----------|
| E4.1 | Parallel testing? | a) Wajib t.Parallel()<br>b) Recommended<br>c) Sequential default | |
| E4.2 | Test timeout? | a) Per-test timeout<br>b) Global timeout<br>c) Tidak di-enforce | |
| E4.3 | Race detection? | a) Wajib di CI<br>b) Opsional<br>c) Development only | |
| E4.4 | Flaky test handling? | a) Retry mechanism<br>b) Quarantine<br>c) Immediate fix required | |

---

# GROUP F: OBSERVABILITY

> **Keperluan:** Monitoring, debugging, alerting

---

## F1. Logging (Production)

| # | Pertanyaan | Opsi | Keputusan |
|---|------------|------|-----------|
| F1.1 | Log aggregation platform? | a) ELK Stack<br>b) Loki + Grafana<br>c) Cloud provider (CloudWatch, Stackdriver)<br>d) Datadog | |
| F1.2 | Log retention policy? | a) 7 days<br>b) 30 days<br>c) 90 days<br>d) Per-environment | |
| F1.3 | Log level di production? | a) INFO minimum<br>b) WARN minimum<br>c) Configurable per service | |

---

## F2. Metrics

| # | Pertanyaan | Opsi | Keputusan |
|---|------------|------|-----------|
| F2.1 | Metrics library? | a) Prometheus client_golang<br>b) OpenTelemetry metrics<br>c) StatsD<br>d) Cloud provider SDK | |
| F2.2 | Standard metrics wajib? | a) RED (Rate, Errors, Duration)<br>b) USE (Utilization, Saturation, Errors)<br>c) Custom per service | |
| F2.3 | Metrics naming convention? | a) Prometheus naming conventions<br>b) Custom prefix per service<br>c) OpenTelemetry semantic conventions | |
| F2.4 | Custom metrics approval? | a) Review by SRE/Platform team<br>b) Self-service dengan guidelines<br>c) Tidak perlu approval | |

---

## F3. Distributed Tracing

| # | Pertanyaan | Opsi | Keputusan |
|---|------------|------|-----------|
| F3.1 | Tracing implementation? | a) OpenTelemetry<br>b) Jaeger client<br>c) Zipkin<br>d) Cloud provider (X-Ray, Cloud Trace) | |
| F3.2 | Trace context propagation? | a) W3C Trace Context<br>b) B3 (Zipkin)<br>c) Custom headers | |
| F3.3 | Sampling strategy? | a) Head-based sampling<br>b) Tail-based sampling<br>c) Always sample<br>d) Configurable rate | |
| F3.4 | Span attributes standard? | a) OpenTelemetry semantic conventions<br>b) Custom conventions<br>c) Minimal (service, operation saja) | |

---

## F4. Health Checks

| # | Pertanyaan | Opsi | Keputusan |
|---|------------|------|-----------|
| F4.1 | Health check endpoints? | a) /health (liveness) + /ready (readiness)<br>b) Single /health endpoint<br>c) gRPC Health protocol | |
| F4.2 | Health check depth? | a) Basic (service running)<br>b) Shallow (+ critical dependencies)<br>c) Deep (all dependencies) | |
| F4.3 | Health check response format? | a) Simple 200/503<br>b) JSON dengan details<br>c) Kubernetes probe compatible | |

---

## F5. Alerting

| # | Pertanyaan | Opsi | Keputusan |
|---|------------|------|-----------|
| F5.1 | Alert definition location? | a) Terraform/IaC<br>b) Prometheus rules di repo<br>c) UI based (Grafana/PagerDuty)<br>d) Centralized repo | |
| F5.2 | Standard alerts per service? | a) Ya, template wajib<br>b) Recommendations saja<br>c) Per-team decision | |
| F5.3 | Alert severity levels? | a) P1-P4<br>b) Critical, Warning, Info<br>c) Custom per team | |

---

## F6. Database Monitoring

| # | Pertanyaan | Opsi | Keputusan |
|---|------------|------|-----------|
| F6.1 | Metrics dari database yang di-monitor? | a) Connections, queries/sec, replication lag<br>b) + CPU, memory, disk<br>c) + Lock stats, dead tuples<br>d) All available metrics | |
| F6.2 | Monitoring tool? | a) Cloud provider native<br>b) Prometheus + exporters<br>c) Datadog/NewRelic<br>d) Custom solution | |
| F6.3 | Dashboard wajib per-service? | a) Ya, template standar<br>b) Ya, minimal requirements<br>c) Recommended saja | |
| F6.4 | Alert untuk slow query rate? | a) > 1% queries slow<br>b) > 5% queries slow<br>c) > N slow queries per minute<br>d) Tidak alert, monitor saja | |
| F6.5 | Alert untuk connection pool? | a) > 80% utilization<br>b) Wait time > threshold<br>c) Error rate > threshold<br>d) All above | |
| F6.6 | Alert untuk long running queries? | a) > 30 seconds<br>b) > 1 minute<br>c) > 5 minutes<br>d) Per-query type | |
| F6.7 | Alert untuk replication lag? | a) > 1 second<br>b) > 5 seconds<br>c) > 30 seconds<br>d) Tidak alert | |

---

# GROUP G: SECURITY

> **Keperluan:** Security compliance, data protection

---

## G1. Input Validation

| # | Pertanyaan | Opsi | Keputusan |
|---|------------|------|-----------|
| G1.1 | Validation library? | a) go-playground/validator<br>b) ozzo-validation<br>c) Custom validation<br>d) Proto validation (protoc-gen-validate) | |
| G1.2 | Validation location? | a) Handler/API layer<br>b) Domain/Service layer<br>c) Both layers | |
| G1.3 | Validation error response? | a) Field-level errors<br>b) First error only<br>c) Aggregated errors | |

---

## G2. Authentication & Authorization

| # | Pertanyaan | Opsi | Keputusan |
|---|------------|------|-----------|
| G2.1 | Auth token format? | a) JWT<br>b) Opaque token<br>c) API Key<br>d) Combination | |
| G2.2 | Auth validation location? | a) API Gateway<br>b) Per-service middleware<br>c) Both | |
| G2.3 | Service-to-service auth? | a) mTLS<br>b) JWT/Token<br>c) API Key<br>d) No auth (network level) | |
| G2.4 | Authorization pattern? | a) RBAC<br>b) ABAC<br>c) ReBAC<br>d) Custom per service | |
| G2.5 | Permission checking? | a) Middleware<br>b) In handler<br>c) Domain layer<br>d) External service (OPA) | |

---

## G3. Secrets Management

| # | Pertanyaan | Opsi | Keputusan |
|---|------------|------|-----------|
| G3.1 | Secrets storage? | a) HashiCorp Vault<br>b) Cloud secrets manager<br>c) Kubernetes secrets<br>d) Environment variables | |
| G3.2 | Secrets rotation? | a) Automatic rotation<br>b) Manual rotation schedule<br>c) On-demand | |
| G3.3 | Secrets in local dev? | a) .env file (gitignored)<br>b) Local vault<br>c) Shared dev secrets | |

---

## G4. Data Protection

| # | Pertanyaan | Opsi | Keputusan |
|---|------------|------|-----------|
| G4.1 | PII handling guidelines? | a) Strict encryption at rest<br>b) Tokenization<br>c) Pseudonymization<br>d) Case by case | |
| G4.2 | Data encryption at rest? | a) Database level<br>b) Application level<br>c) Both<br>d) Not required | |
| G4.3 | Sensitive data logging? | a) Never log PII<br>b) Masked/redacted<br>c) Separate secure logs | |

---

## G5. Security Scanning

| # | Pertanyaan | Opsi | Keputusan |
|---|------------|------|-----------|
| G5.1 | Dependency scanning? | a) Dependabot<br>b) Snyk<br>c) govulncheck<br>d) Multiple tools | |
| G5.2 | SAST tools? | a) gosec<br>b) semgrep<br>c) SonarQube<br>d) Tidak pakai | |
| G5.3 | Container scanning? | a) Trivy<br>b) Clair<br>c) Cloud provider scanner<br>d) Tidak pakai | |
| G5.4 | Scan enforcement? | a) Block deployment on critical<br>b) Warning only<br>c) Periodic audit | |

---

# GROUP H: WORKFLOW

> **Keperluan:** Development process, CI/CD

---

## H1. Branching Strategy

| # | Pertanyaan | Opsi | Keputusan |
|---|------------|------|-----------|
| H1.1 | Branching model? | a) GitHub Flow (main + feature)<br>b) GitFlow (main, develop, feature, release)<br>c) Trunk-based development<br>d) Per-squad decision | |
| H1.2 | Branch naming convention? | a) type/description (feature/add-login)<br>b) JIRA-ID/description<br>c) username/description<br>d) Tidak di-enforce | |
| H1.3 | Protected branches? | a) main only<br>b) main + develop<br>c) main + release branches | |

---

## H2. Commit Standards

| # | Pertanyaan | Opsi | Keputusan |
|---|------------|------|-----------|
| H2.1 | Commit message format? | a) Conventional Commits<br>b) JIRA ID prefix<br>c) Free form<br>d) Custom format | |
| H2.2 | Commit message enforcement? | a) Pre-commit hook<br>b) CI check<br>c) Code review saja<br>d) Tidak di-enforce | |
| H2.3 | Commit granularity? | a) Atomic commits (one change per commit)<br>b) Feature-complete commits<br>c) Squash on merge | |

---

## H3. Pull Request

| # | Pertanyaan | Opsi | Keputusan |
|---|------------|------|-----------|
| H3.1 | PR template? | a) Ya, standardisasi template<br>b) Per-repo template<br>c) Tidak pakai template | |
| H3.2 | PR size limit? | a) Max 400 lines<br>b) Max 200 lines<br>c) Tidak dibatasi<br>d) Guidelines saja | |
| H3.3 | Required reviewers? | a) Min 1 reviewer<br>b) Min 2 reviewers<br>c) CODEOWNERS based<br>d) Per-squad policy | |
| H3.4 | Review SLA? | a) 24 hours<br>b) 48 hours<br>c) Tidak ada SLA | |

---

## H4. CI Pipeline

| # | Pertanyaan | Opsi | Keputusan |
|---|------------|------|-----------|
| H4.1 | CI platform? | a) GitHub Actions<br>b) GitLab CI<br>c) Jenkins<br>d) CircleCI<br>e) Other | |
| H4.2 | Required CI checks? | a) Lint + Test + Build<br>b) + Security scan<br>c) + Integration tests<br>d) Custom per repo | |
| H4.3 | CI check enforcement? | a) All checks must pass<br>b) Critical checks only<br>c) Advisory (non-blocking) | |

---

## H5. CD Pipeline

| # | Pertanyaan | Opsi | Keputusan |
|---|------------|------|-----------|
| H5.1 | Deployment strategy? | a) Rolling update<br>b) Blue-green<br>c) Canary<br>d) Per-service decision | |
| H5.2 | Environment progression? | a) dev → staging → prod<br>b) dev → prod (with feature flags)<br>c) Custom per team | |
| H5.3 | Production deployment approval? | a) Manual approval required<br>b) Automatic with criteria<br>c) Per-team decision | |
| H5.4 | Rollback strategy? | a) Automatic on failure<br>b) Manual trigger<br>c) Redeploy previous version | |

---

## H6. Code Quality Gates

| # | Pertanyaan | Opsi | Keputusan |
|---|------------|------|-----------|
| H6.1 | Linter wajib? | a) golangci-lint dengan preset config<br>b) Per-squad config<br>c) Tidak wajib | |
| H6.2 | Linter rules standardization? | a) Shared .golangci.yml<br>b) Minimum required rules<br>c) Per-repo decision | |
| H6.3 | Format enforcement? | a) gofmt (strict)<br>b) goimports<br>c) gofumpt<br>d) Tidak di-enforce | |

---

# GROUP I: DOCUMENTATION

> **Keperluan:** Knowledge sharing, onboarding

---

## I1. Code Documentation

| # | Pertanyaan | Opsi | Keputusan |
|---|------------|------|-----------|
| I1.1 | Godoc requirement? | a) All exported functions/types<br>b) Package level saja<br>c) Tidak wajib | |
| I1.2 | Godoc format? | a) Full sentences, proper punctuation<br>b) Brief description OK<br>c) Tidak di-enforce | |
| I1.3 | Code comment language? | a) English only<br>b) Bahasa Indonesia allowed<br>c) Per-team decision | |
| I1.4 | Example in godoc? | a) Wajib untuk complex functions<br>b) Recommended<br>c) Opsional | |

---

## I2. README Standards

| # | Pertanyaan | Opsi | Keputusan |
|---|------------|------|-----------|
| I2.1 | README template? | a) Ya, standardisasi template<br>b) Minimum sections required<br>c) Tidak di-enforce | |
| I2.2 | Required README sections? | a) Overview, Setup, Usage, API<br>b) Overview, Setup saja<br>c) Per-team decision | |
| I2.3 | Badges di README? | a) Standard badges (build, coverage)<br>b) Opsional<br>c) Tidak perlu | |

---

## I3. Architecture Decision Records (ADR)

| # | Pertanyaan | Opsi | Keputusan |
|---|------------|------|-----------|
| I3.1 | ADR wajib? | a) Ya, untuk semua major decisions<br>b) Untuk breaking changes saja<br>c) Recommended tapi tidak wajib<br>d) Tidak pakai ADR | |
| I3.2 | ADR format? | a) MADR (Markdown ADR)<br>b) Michael Nygard format<br>c) Custom format | |
| I3.3 | ADR location? | a) docs/adr/ di repo<br>b) Centralized documentation repo<br>c) Confluence/Wiki | |
| I3.4 | ADR lifecycle? | a) Proposed → Accepted → Deprecated<br>b) Draft → Approved<br>c) No formal lifecycle | |

---

## I4. API Documentation

| # | Pertanyaan | Opsi | Keputusan |
|---|------------|------|-----------|
| I4.1 | API docs platform? | a) Generated dari proto/code<br>b) Swagger UI<br>c) Postman collections<br>d) Confluence/Wiki | |
| I4.2 | API changelog? | a) Per-release changelog<br>b) Per-PR documentation<br>c) Git history saja | |
| I4.3 | Breaking change documentation? | a) Mandatory migration guide<br>b) Changelog entry saja<br>c) Tidak ada requirement | |

---

## I5. Runbook & Operations

| # | Pertanyaan | Opsi | Keputusan |
|---|------------|------|-----------|
| I5.1 | Runbook requirement? | a) Wajib sebelum production<br>b) Recommended<br>c) Tidak wajib | |
| I5.2 | Runbook template? | a) Standard template<br>b) Minimum sections<br>c) Free form | |
| I5.3 | Runbook location? | a) docs/runbooks/ di repo<br>b) Centralized wiki<br>c) Confluence/Notion | |
| I5.4 | Runbook review? | a) Review by SRE/on-call team<br>b) Self-review<br>c) Tidak ada review | |

---

# GROUP J: ADVANCED PATTERNS

> **Keperluan:** Complex scenarios, advanced use cases

---

## J1. Distributed Locking

| # | Pertanyaan | Opsi | Keputusan |
|---|------------|------|-----------|
| J1.1 | Distributed lock implementation? | a) Redis (Redlock/single instance)<br>b) Database advisory locks<br>c) etcd/Consul<br>d) Tidak support | |
| J1.2 | Lock timeout default? | a) 5 seconds<br>b) 30 seconds<br>c) No default (wajib explicit)<br>d) Per-operation | |
| J1.3 | Lock key convention? | a) lock:{resource}:{id}<br>b) {service}:lock:{resource}<br>c) Tidak di-enforce | |
| J1.4 | Kapan wajib pakai distributed lock? | a) Documented use cases<br>b) Per-case decision<br>c) Avoid locks, use other patterns | |

**Contoh Use Case Distributed Lock:**
```go
func (s *PaymentService) ProcessPayment(ctx context.Context, orderID string) error {
    lockKey := fmt.Sprintf("lock:payment:%s", orderID)
    
    lock, err := s.locker.Acquire(ctx, lockKey, 30*time.Second)
    if err != nil {
        return fmt.Errorf("could not acquire lock: %w", err)
    }
    defer lock.Release(ctx)
    
    // Check if already processed (idempotency)
    if processed, _ := s.repo.IsPaymentProcessed(ctx, orderID); processed {
        return nil
    }
    
    return s.processPaymentInternal(ctx, orderID)
}
```

---

## J2. Idempotency

| # | Pertanyaan | Opsi | Keputusan |
|---|------------|------|-----------|
| J2.1 | Idempotency key storage? | a) Database table<br>b) Redis dengan TTL<br>c) Combination<br>d) Per-case decision | |
| J2.2 | Idempotency key format? | a) Client-provided<br>b) Server-generated (hash of request)<br>c) Both supported | |
| J2.3 | Idempotency window/TTL? | a) 24 hours<br>b) 7 days<br>c) Configurable per endpoint<br>d) No expiry | |
| J2.4 | Wajib untuk endpoint mana? | a) All mutating operations<br>b) Payment/financial only<br>c) Documented per-endpoint | |

---

## J3. Optimistic vs Pessimistic Locking

| # | Pertanyaan | Opsi | Keputusan |
|---|------------|------|-----------|
| J3.1 | Default locking strategy? | a) Optimistic (version column)<br>b) Pessimistic (SELECT FOR UPDATE)<br>c) Per-case decision | |
| J3.2 | Optimistic lock implementation? | a) Version number column<br>b) Updated_at timestamp<br>c) Hash of row content | |
| J3.3 | Retry on optimistic lock failure? | a) Ya, dengan limit<br>b) Tidak, return conflict error<br>c) Per-case decision | |

**Contoh Optimistic Locking:**
```go
type Order struct {
    ID        string
    Status    string
    Version   int  // Optimistic lock version
    UpdatedAt time.Time
}

func (r *OrderRepository) Update(ctx context.Context, order *Order) error {
    result, err := r.db.ExecContext(ctx, `
        UPDATE orders 
        SET status = $1, version = version + 1, updated_at = NOW()
        WHERE id = $2 AND version = $3
    `, order.Status, order.ID, order.Version)
    
    if err != nil {
        return err
    }
    
    rowsAffected, _ := result.RowsAffected()
    if rowsAffected == 0 {
        return ErrOptimisticLockConflict
    }
    
    return nil
}
```

---

## J4. Audit Logging

| # | Pertanyaan | Opsi | Keputusan |
|---|------------|------|-----------|
| J4.1 | Audit log untuk data changes? | a) Ya, wajib untuk semua entities<br>b) Ya, untuk critical entities saja<br>c) Opsional<br>d) Tidak perlu | |
| J4.2 | Audit log implementation? | a) Application level (before/after)<br>b) Database triggers<br>c) CDC (Change Data Capture)<br>d) Separate audit table | |
| J4.3 | Apa yang di-log? | a) Who, when, what changed<br>b) + Old value, new value<br>c) + Request context (IP, user agent) | |
| J4.4 | Audit log storage? | a) Same database<br>b) Separate database<br>c) Log aggregation system<br>d) Data warehouse | |

---

## J5. Soft Delete

| # | Pertanyaan | Opsi | Keputusan |
|---|------------|------|-----------|
| J5.1 | Default delete strategy? | a) Soft delete (deleted_at column)<br>b) Hard delete<br>c) Per-entity decision | |
| J5.2 | Soft delete column name? | a) deleted_at (timestamp)<br>b) is_deleted (boolean)<br>c) deleted (boolean) + deleted_at | |
| J5.3 | Query filtering untuk soft delete? | a) Automatic (repository level)<br>b) Manual di setiap query<br>c) Database view | |
| J5.4 | Permanent delete policy? | a) Never<br>b) After X days/months<br>c) Manual process dengan approval<br>d) Per data retention policy | |

---

## J6. Data Archival

| # | Pertanyaan | Opsi | Keputusan |
|---|------------|------|-----------|
| J6.1 | Data archival strategy? | a) Separate archive tables<br>b) Separate archive database<br>c) Cold storage (S3, etc)<br>d) Tidak ada archival | |
| J6.2 | Archival trigger? | a) Age-based (> X months)<br>b) Status-based (completed/closed)<br>c) Manual process<br>d) Combination | |
| J6.3 | Archived data accessibility? | a) Read-only queries<br>b) Restore on request<br>c) Not accessible from app | |

---

## J7. Graceful Degradation

| # | Pertanyaan | Opsi | Keputusan |
|---|------------|------|-----------|
| J7.1 | Behavior saat database partial failure? | a) Fail entire request<br>b) Return partial data<br>c) Serve cached/stale data<br>d) Per-endpoint decision | |
| J7.2 | Read replica failover? | a) Automatic ke primary<br>b) Return error<br>c) Serve stale dari cache | |
| J7.3 | Circuit breaker untuk database? | a) Ya, wajib<br>b) Recommended<br>c) Tidak perlu | |
| J7.4 | Bulkhead pattern untuk database? | a) Ya, separate pools per feature<br>b) Tidak, single pool<br>c) Per-case decision | |

---

## J8. Rate Limiting (Database Level)

| # | Pertanyaan | Opsi | Keputusan |
|---|------------|------|-----------|
| J8.1 | Rate limit untuk expensive queries? | a) Ya, application level<br>b) Ya, database level (statement timeout)<br>c) Tidak perlu | |
| J8.2 | Connection rate limiting? | a) Ya, max connections per service<br>b) Ya, via PgBouncer/ProxySQL<br>c) Database default limits | |
| J8.3 | Query concurrency limit? | a) Ya, semaphore pattern<br>b) Ya, worker pool<br>c) Tidak, rely on connection pool | |

---

## J9. Incident Response (Database)

| # | Pertanyaan | Opsi | Keputusan |
|---|------------|------|-----------|
| J9.1 | Runbook untuk slow query incident? | a) Ya, wajib ada<br>b) Generic DB runbook<br>c) Tidak ada formal runbook | |
| J9.2 | Siapa yang handle DB performance issue? | a) Service owner<br>b) DBA team<br>c) Platform/SRE team<br>d) Escalation path defined | |
| J9.3 | Kill query threshold? | a) Auto-kill > X minutes<br>b) Manual approval<br>c) Never auto-kill | |

---

## J10. Multi-tenancy (jika applicable)

| # | Pertanyaan | Opsi | Keputusan |
|---|------------|------|-----------|
| J10.1 | Multi-tenancy model? | a) Shared database, shared schema<br>b) Shared database, separate schema<br>c) Separate database per tenant<br>d) Tidak multi-tenant | |
| J10.2 | Tenant isolation enforcement? | a) Application level (WHERE tenant_id = ?)<br>b) Row-level security (RLS)<br>c) Connection-level (set search_path)<br>d) Database per tenant | |
| J10.3 | Tenant context propagation? | a) Context value<br>b) Request header<br>c) JWT claim<br>d) Connection string | |

---

# IMPLEMENTATION EXAMPLES

## Database Wrapper dengan Slow Query Detection

```go
package database

import (
    "context"
    "database/sql"
    "log/slog"
    "time"
)

type Config struct {
    SlowQueryThreshold time.Duration
    LogAllQueries      bool
}

type DB struct {
    *sql.DB
    config Config
    logger *slog.Logger
}

func (db *DB) QueryContext(ctx context.Context, query string, args ...interface{}) (*sql.Rows, error) {
    start := time.Now()
    
    rows, err := db.DB.QueryContext(ctx, query, args...)
    
    duration := time.Since(start)
    db.logQuery(ctx, query, args, duration, err)
    
    return rows, err
}

func (db *DB) logQuery(ctx context.Context, query string, args []interface{}, duration time.Duration, err error) {
    fields := []any{
        "query", query,
        "duration_ms", duration.Milliseconds(),
        "trace_id", getTraceID(ctx),
    }
    
    if err != nil {
        db.logger.Error("query failed", append(fields, "error", err)...)
        return
    }
    
    if duration > db.config.SlowQueryThreshold {
        db.logger.Warn("slow query detected", 
            append(fields, "threshold_ms", db.config.SlowQueryThreshold.Milliseconds())...)
        slowQueryCounter.Inc()
        return
    }
    
    if db.config.LogAllQueries {
        db.logger.Debug("query executed", fields...)
    }
}
```

| # | Pertanyaan | Opsi | Keputusan |
|---|------------|------|-----------|
| IMP.1 | Apakah perlu buat shared library seperti contoh di atas? | a) Ya, library internal<br>b) Setiap squad implement sendiri<br>c) Pakai existing library | |

---

## DB Review Checklist

- [ ] Query menggunakan parameterized query (no SQL injection risk)
- [ ] Index tersedia untuk WHERE clause
- [ ] LIMIT digunakan untuk queries yang bisa return banyak rows
- [ ] Transaction scope minimal dan appropriate
- [ ] Error handling lengkap
- [ ] Connection/transaction tidak leak
- [ ] EXPLAIN ANALYZE dilampirkan untuk query baru
- [ ] Tidak ada N+1 query pattern
- [ ] Timeout di-set via context

| # | Pertanyaan | Opsi | Keputusan |
|---|------------|------|-----------|
| IMP.2 | Apakah checklist di atas sudah cukup? | a) Ya<br>b) Perlu ditambah<br>c) Terlalu banyak | |

---

# SUMMARY

| Group | Nama | Jumlah Pertanyaan |
|-------|------|-------------------|
| A | Foundation | 15 |
| B | Coding Standards | 28 |
| C | API Design | 17 |
| D | Data Layer | 95 |
| E | Testing | 17 |
| F | Observability | 22 |
| G | Security | 18 |
| H | Workflow | 21 |
| I | Documentation | 17 |
| J | Advanced Patterns | 32 |
| | **TOTAL** | **~282 pertanyaan** |

---

# MEETING NOTES

## Keputusan yang Sudah Dibuat

| Tanggal | Group | Aspek | Keputusan | Alasan |
|---------|-------|-------|-----------|--------|
| | | | | |

## Parking Lot (Perlu Diskusi Lebih Lanjut)

- 

## Action Items

- [ ] 

---

**Tags:** #architecture #guidelines #development #standardization #database #redis #security #testing #observability

---

**Related Files:**
- [[Development-Guidelines-Discussion-Points]] (Source)
- [[Database-Access-Best-Practices-Discussion]] (Source)
