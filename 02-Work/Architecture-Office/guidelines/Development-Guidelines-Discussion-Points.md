---
title: Development Guidelines - Discussion Points
created: '2026-01-21'
type: discussion-document
status: draft
related_meeting: 2026-01-20 Meeting VP Architect - Development Guidelines
owner: Architecture Team
---
# Development Guidelines - Discussion Points

> **Tujuan:** Dokumen ini berisi pertanyaan-pertanyaan yang perlu didiskusikan dan diputuskan bersama oleh tim Architect sebelum finalisasi Development Guidelines.
> 
> **Related Meeting:** [[2026-01-20 Meeting VP Architect - Development Guidelines]]

---

## Status Keputusan

| # | Aspek | Status | PIC | Tanggal Decided |
|---|-------|--------|-----|-----------------|
| 1 | Code Structure & Project Layout | ⬜ Pending | - | - |
| 2 | Coding Standards | ⬜ Pending | - | - |
| 3 | API Design Guidelines | ⬜ Pending | - | - |
| 4 | Database & Data Access | ⬜ Pending | - | - |
| 5 | Testing Standards | ⬜ Pending | - | - |
| 6 | Observability | ⬜ Pending | - | - |
| 7 | Security Guidelines | ⬜ Pending | - | - |
| 8 | Git & CI/CD Workflow | ⬜ Pending | - | - |
| 9 | Documentation Standards | ⬜ Pending | - | - |

---

## 1. Code Structure & Project Layout

### 1.1 Project Layout Standard

| #     | Pertanyaan                                                  | Opsi                                                                                                              | Keputusan                      |
| ----- | ----------------------------------------------------------- | ----------------------------------------------------------------------------------------------------------------- | ------------------------------ |
| 1.1.1 | Apakah perlu enforce standard project layout across squads? | a) Ya, wajib semua sama<br>b) Tidak, fleksibel per squad<br>c) Hybrid: core structure sama, detail fleksibel      | C<br>biar fleksibilitas tinggi |
| 1.1.2 | Jika ya, layout mana yang jadi standard?                    | a) golang-standards/project-layout<br>b) Custom Bluebird layout<br>c) Adopt dari squad tertentu yang sudah mature | B                              |
| 1.1.3 | Apakah perlu ada skeleton generator/template?               | a) Ya, buat CLI generator<br>b) Ya, template repository<br>c) Tidak perlu                                         | A                              |

### 1.2 Package Organization

| #     | Pertanyaan                                  | Opsi                                                                                                            | Keputusan |
| ----- | ------------------------------------------- | --------------------------------------------------------------------------------------------------------------- | --------- |
| 1.2.1 | Architecture pattern yang direkomendasikan? | a) Clean Architecture<br>b) Hexagonal Architecture<br>c) Domain-Driven Design<br>d) Fleksibel, tidak di-enforce | A         |
| 1.2.2 | Internal package structure?                 | a) By layer (handler, service, repository)<br>b) By domain/feature<br>c) Hybrid                                 | C         |
| 1.2.3 | Shared/common package handling?             | a) Monorepo shared module<br>b) Private Go module terpisah<br>c) Copy-paste allowed                             | A         |

### 1.3 Naming Conventions

| #     | Pertanyaan                 | Opsi                                                                                                           | Keputusan |
| ----- | -------------------------- | -------------------------------------------------------------------------------------------------------------- | --------- |
| 1.3.1 | File naming convention?    | a) snake_case.go<br>b) lowercase.go<br>c) Tidak di-enforce                                                     | A         |
| 1.3.2 | Package naming convention? | a) Single word lowercase (strict)<br>b) Allow multi-word dengan underscore<br>c) Follow Go standard saja       | A         |
| 1.3.3 | Interface naming?          | a) Prefix dengan I (IUserService)<br>b) Suffix dengan -er (UserReader)<br>c) Tanpa prefix/suffix (UserService) |           |

### 1.4 Repository Strategy

| # | Pertanyaan | Opsi | Keputusan |
|---|------------|------|-----------|
| 1.4.1 | Monorepo vs Multi-repo? | a) Monorepo per squad<br>b) Multi-repo per service<br>c) Hybrid (core monorepo + satellite repos)<br>d) Tidak di-enforce | |
| 1.4.2 | Jika monorepo, tooling apa yang dipakai? | a) Go workspace<br>b) Bazel<br>c) Makefile based<br>d) Custom tooling | |

---

## 2. Coding Standards

### 2.1 Error Handling

| # | Pertanyaan | Opsi | Keputusan |
|---|------------|------|-----------|
| 2.1.1 | Error handling approach? | a) Standard errors + fmt.Errorf wrapping<br>b) pkg/errors (deprecated)<br>c) Custom error package internal<br>d) errors.Join (Go 1.20+) | |
| 2.1.2 | Apakah perlu custom error types? | a) Ya, dengan error codes<br>b) Ya, dengan error categories<br>c) Tidak, standard error cukup | |
| 2.1.3 | Error wrapping mandatory? | a) Ya, wajib wrap dengan context<br>b) Tidak, opsional<br>c) Wajib di boundary layer saja | |
| 2.1.4 | Sentinel errors standard? | a) Definisikan di package errors internal<br>b) Per-package sentinel errors<br>c) Tidak pakai sentinel errors | |
| 2.1.5 | Stack trace di error? | a) Ya, selalu include<br>b) Hanya di development<br>c) Tidak perlu | |

### 2.2 Logging

| # | Pertanyaan | Opsi | Keputusan |
|---|------------|------|-----------|
| 2.2.1 | Logging library standard? | a) zerolog<br>b) zap<br>c) slog (Go 1.21+)<br>d) logrus<br>e) Fleksibel per squad | |
| 2.2.2 | Log format? | a) JSON (structured)<br>b) Text (human readable)<br>c) JSON di production, text di development | |
| 2.2.3 | Log levels yang dipakai? | a) DEBUG, INFO, WARN, ERROR, FATAL<br>b) TRACE, DEBUG, INFO, WARN, ERROR<br>c) Custom levels | |
| 2.2.4 | Mandatory fields di setiap log? | a) timestamp, level, message saja<br>b) + request_id, service_name<br>c) + user_id, trace_id<br>d) Custom per squad | |
| 2.2.5 | Sensitive data di log? | a) Strict no PII<br>b) Masked/redacted<br>c) Allowed di debug level | |
| 2.2.6 | Log sampling di production? | a) Ya, dengan rate<br>b) Tidak, log semua<br>c) Sampling untuk DEBUG saja | |

### 2.3 Configuration Management

| # | Pertanyaan | Opsi | Keputusan |
|---|------------|------|-----------|
| 2.3.1 | Config source priority? | a) Env vars only<br>b) File → Env vars<br>c) Env vars → File<br>d) Remote config (Consul/etcd) | |
| 2.3.2 | Config library? | a) viper<br>b) envconfig<br>c) koanf<br>d) Custom/standard library | |
| 2.3.3 | Config validation? | a) Validate di startup (fail fast)<br>b) Validate lazy<br>c) Tidak perlu validation | |
| 2.3.4 | Secret management? | a) Env vars<br>b) Vault (HashiCorp)<br>c) Cloud provider secrets manager<br>d) Kubernetes secrets | |
| 2.3.5 | Config struct approach? | a) Single global config struct<br>b) Per-package config<br>c) Hierarchical config | |

### 2.4 Context Usage

| # | Pertanyaan | Opsi | Keputusan |
|---|------------|------|-----------|
| 2.4.1 | Context sebagai first parameter? | a) Ya, wajib semua function<br>b) Wajib di public functions saja<br>c) Opsional | |
| 2.4.2 | Context value usage? | a) Hanya untuk request-scoped data<br>b) Boleh untuk DI<br>c) Strict: trace ID dan deadline saja | |
| 2.4.3 | Custom context keys? | a) Typed keys (unexported)<br>b) String keys dengan prefix<br>c) Shared package untuk keys | |
| 2.4.4 | Timeout/deadline standard? | a) Set di entry point saja<br>b) Setiap layer boleh set<br>c) Defined di config per operation | |

### 2.5 Concurrency Patterns

| # | Pertanyaan | Opsi | Keputusan |
|---|------------|------|-----------|
| 2.5.1 | Goroutine management? | a) errgroup standard<br>b) Custom worker pool<br>c) Fleksibel | |
| 2.5.2 | Channel vs Mutex preference? | a) Prefer channels<br>b) Prefer mutex untuk simple cases<br>c) Case by case | |
| 2.5.3 | Goroutine leak prevention? | a) Mandatory context cancellation<br>b) Linter enforcement<br>c) Code review saja | |
| 2.5.4 | Rate limiting pattern? | a) golang.org/x/time/rate<br>b) Custom implementation<br>c) External (Redis based) | |

### 2.6 Dependency Injection

| # | Pertanyaan | Opsi | Keputusan |
|---|------------|------|-----------|
| 2.6.1 | DI approach? | a) Manual constructor injection<br>b) Wire (Google)<br>c) fx (Uber)<br>d) dig (Uber) | |
| 2.6.2 | Interface definition location? | a) Di consumer package<br>b) Di provider package<br>c) Di shared package | |
| 2.6.3 | Constructor pattern? | a) New() returns struct<br>b) New() returns interface<br>c) Functional options pattern | |

---

## 3. API Design Guidelines

### 3.1 gRPC Standards

| # | Pertanyaan | Opsi | Keputusan |
|---|------------|------|-----------|
| 3.1.1 | Proto file organization? | a) Single proto per service<br>b) Split by domain<br>c) Monorepo proto | |
| 3.1.2 | Proto versioning? | a) Package versioning (v1, v2)<br>b) Field deprecation only<br>c) URL path versioning | |
| 3.1.3 | Proto style guide? | a) Google API design guide<br>b) Uber style guide<br>c) Custom Bluebird style | |
| 3.1.4 | Proto linting? | a) buf<br>b) protolint<br>c) Tidak pakai linter | |
| 3.1.5 | Proto breaking change detection? | a) buf breaking<br>b) Manual review<br>c) Tidak di-enforce | |

### 3.2 REST API Standards (jika ada)

| # | Pertanyaan | Opsi | Keputusan |
|---|------------|------|-----------|
| 3.2.1 | REST masih dipakai? | a) Ya, untuk public API<br>b) Ya, untuk internal juga<br>c) Tidak, full gRPC | |
| 3.2.2 | REST framework? | a) gin<br>b) echo<br>c) chi<br>d) fiber<br>e) standard net/http | |
| 3.2.3 | API versioning? | a) URL path (/v1/)<br>b) Header based<br>c) Query param | |
| 3.2.4 | Response format? | a) JSON:API spec<br>b) Custom envelope<br>c) Plain JSON | |

### 3.3 Error Response Standards

| # | Pertanyaan | Opsi | Keputusan |
|---|------------|------|-----------|
| 3.3.1 | gRPC error codes mapping? | a) Standard gRPC codes saja<br>b) Custom codes di details<br>c) Custom error proto | |
| 3.3.2 | Error response structure? | a) Google error model<br>b) RFC 7807 (Problem Details)<br>c) Custom structure | |
| 3.3.3 | Error codes registry? | a) Centralized di satu repo<br>b) Per-service codes<br>c) Tidak pakai codes | |
| 3.3.4 | Localized error messages? | a) Ya, dengan i18n<br>b) English only<br>c) Code saja, client handle message | |

### 3.4 API Documentation

| # | Pertanyaan | Opsi | Keputusan |
|---|------------|------|-----------|
| 3.4.1 | gRPC documentation? | a) Proto comments saja<br>b) Proto comments + generated docs<br>c) Separate documentation | |
| 3.4.2 | REST documentation? | a) OpenAPI/Swagger auto-generated<br>b) Manual OpenAPI spec<br>c) Tidak ada formal docs | |
| 3.4.3 | API changelog? | a) CHANGELOG.md per service<br>b) Git tags saja<br>c) Centralized changelog | |

---

## 4. Database & Data Access

### 4.1 Database Connection

| # | Pertanyaan | Opsi | Keputusan |
|---|------------|------|-----------|
| 4.1.1 | SQL driver? | a) database/sql + driver<br>b) sqlx<br>c) pgx (Postgres specific)<br>d) Fleksibel | |
| 4.1.2 | Connection pooling settings? | a) Standardisasi max connections<br>b) Per-service based on load<br>c) Tidak di-enforce | |
| 4.1.3 | Connection string management? | a) Full DSN di env var<br>b) Individual params (host, port, etc)<br>c) Secrets manager | |
| 4.1.4 | Read replica handling? | a) Separate connection pool<br>b) Proxy based (PgBouncer/ProxySQL)<br>c) Application level routing | |

### 4.2 Query Patterns

| # | Pertanyaan | Opsi | Keputusan |
|---|------------|------|-----------|
| 4.2.1 | Query builder vs Raw SQL? | a) Raw SQL only<br>b) Query builder (squirrel, goqu)<br>c) Hybrid | |
| 4.2.2 | ORM usage? | a) Tidak boleh ORM<br>b) GORM allowed<br>c) ent allowed<br>d) Fleksibel | |
| 4.2.3 | Prepared statements? | a) Wajib semua query<br>b) Untuk frequent queries saja<br>c) Tidak di-enforce | |
| 4.2.4 | Query timeout? | a) Global default<br>b) Per-query setting<br>c) Context based | |

### 4.3 Slow Query Detection

| # | Pertanyaan | Opsi | Keputusan |
|---|------------|------|-----------|
| 4.3.1 | Slow query threshold? | a) 100ms<br>b) 500ms<br>c) 1s<br>d) Configurable | |
| 4.3.2 | Slow query logging? | a) Application level logging<br>b) Database level (pg_stat_statements)<br>c) Both | |
| 4.3.3 | Query monitoring tools? | a) APM (Datadog, NewRelic, etc)<br>b) Custom metrics<br>c) Database native tools | |
| 4.3.4 | Query analysis di development? | a) EXPLAIN ANALYZE wajib untuk PR<br>b) Linter untuk N+1<br>c) Code review saja | |

### 4.4 Transaction Management

| # | Pertanyaan | Opsi | Keputusan |
|---|------------|------|-----------|
| 4.4.1 | Transaction pattern? | a) Repository method with closure<br>b) Unit of Work pattern<br>c) Manual begin/commit/rollback | |
| 4.4.2 | Nested transaction handling? | a) Savepoints<br>b) Tidak support nested<br>c) Propagation patterns | |
| 4.4.3 | Transaction timeout? | a) Global limit<br>b) Per-transaction setting<br>c) No limit | |
| 4.4.4 | Distributed transaction? | a) Saga pattern<br>b) 2PC<br>c) Eventual consistency<br>d) Tidak support | |

### 4.5 Migration

| # | Pertanyaan | Opsi | Keputusan |
|---|------------|------|-----------|
| 4.5.1 | Migration tool? | a) golang-migrate<br>b) goose<br>c) atlas<br>d) Custom scripts | |
| 4.5.2 | Migration strategy? | a) Forward only<br>b) Up/Down wajib<br>c) Fleksibel | |
| 4.5.3 | Migration di CI/CD? | a) Auto apply di deployment<br>b) Manual approval<br>c) Separate pipeline | |
| 4.5.4 | Schema versioning? | a) Timestamp based<br>b) Sequential number<br>c) Semantic versioning | |

### 4.6 Repository Pattern

| # | Pertanyaan | Opsi | Keputusan |
|---|------------|------|-----------|
| 4.6.1 | Repository interface standard? | a) Generic repository<br>b) Per-entity repository<br>c) CQRS (separate read/write) | |
| 4.6.2 | Pagination pattern? | a) Offset-based<br>b) Cursor-based<br>c) Keyset pagination<br>d) Support both | |
| 4.6.3 | Soft delete handling? | a) Mandatory soft delete<br>b) Hard delete allowed<br>c) Per-entity decision | |

---

## 5. Testing Standards

### 5.1 Unit Testing

| # | Pertanyaan | Opsi | Keputusan |
|---|------------|------|-----------|
| 5.1.1 | Test framework? | a) Standard testing package<br>b) testify<br>c) ginkgo/gomega<br>d) goconvey | |
| 5.1.2 | Assertion library? | a) testify/assert<br>b) testify/require<br>c) Standard + custom helpers<br>d) go-cmp | |
| 5.1.3 | Test file location? | a) Same package (*_test.go)<br>b) Separate _test package<br>c) Hybrid berdasarkan test type | |
| 5.1.4 | Table-driven tests? | a) Wajib untuk multiple cases<br>b) Recommended<br>c) Opsional | |
| 5.1.5 | Coverage minimum? | a) 80%<br>b) 70%<br>c) 60%<br>d) Tidak di-enforce | |

### 5.2 Mocking

| # | Pertanyaan | Opsi | Keputusan |
|---|------------|------|-----------|
| 5.2.1 | Mock generation tool? | a) mockgen (gomock)<br>b) mockery<br>c) moq<br>d) Manual mocks | |
| 5.2.2 | Mock location? | a) Same package<br>b) mocks/ subdirectory<br>c) internal/mocks/ | |
| 5.2.3 | Mock naming convention? | a) Mock prefix (MockUserService)<br>b) mock_ prefix file<br>c) _mock suffix | |
| 5.2.4 | External service mocking? | a) Interface + mock<br>b) httptest<br>c) Testcontainers<br>d) WireMock/MockServer | |

### 5.3 Integration Testing

| # | Pertanyaan | Opsi | Keputusan |
|---|------------|------|-----------|
| 5.3.1 | Database testing? | a) Testcontainers<br>b) Embedded database<br>c) Shared test database<br>d) SQLite for tests | |
| 5.3.2 | Test data management? | a) Fixtures<br>b) Factory/Builder pattern<br>c) Seeds<br>d) Fresh per test | |
| 5.3.3 | Integration test tagging? | a) Build tags (// +build integration)<br>b) Naming convention (_integration_test.go)<br>c) Separate directory | |
| 5.3.4 | External service testing? | a) Real services di staging<br>b) Mock servers<br>c) Contract testing (Pact) | |

### 5.4 Test Execution

| # | Pertanyaan | Opsi | Keputusan |
|---|------------|------|-----------|
| 5.4.1 | Parallel testing? | a) Wajib t.Parallel()<br>b) Recommended<br>c) Sequential default | |
| 5.4.2 | Test timeout? | a) Per-test timeout<br>b) Global timeout<br>c) Tidak di-enforce | |
| 5.4.3 | Race detection? | a) Wajib di CI<br>b) Opsional<br>c) Development only | |
| 5.4.4 | Flaky test handling? | a) Retry mechanism<br>b) Quarantine<br>c) Immediate fix required | |

---

## 6. Observability

### 6.1 Logging (Production)

| # | Pertanyaan | Opsi | Keputusan |
|---|------------|------|-----------|
| 6.1.1 | Log aggregation platform? | a) ELK Stack<br>b) Loki + Grafana<br>c) Cloud provider (CloudWatch, Stackdriver)<br>d) Datadog | |
| 6.1.2 | Log retention policy? | a) 7 days<br>b) 30 days<br>c) 90 days<br>d) Per-environment | |
| 6.1.3 | Log level di production? | a) INFO minimum<br>b) WARN minimum<br>c) Configurable per service | |

### 6.2 Metrics

| # | Pertanyaan | Opsi | Keputusan |
|---|------------|------|-----------|
| 6.2.1 | Metrics library? | a) Prometheus client_golang<br>b) OpenTelemetry metrics<br>c) StatsD<br>d) Cloud provider SDK | |
| 6.2.2 | Standard metrics wajib? | a) RED (Rate, Errors, Duration)<br>b) USE (Utilization, Saturation, Errors)<br>c) Custom per service | |
| 6.2.3 | Metrics naming convention? | a) Prometheus naming conventions<br>b) Custom prefix per service<br>c) OpenTelemetry semantic conventions | |
| 6.2.4 | Custom metrics approval? | a) Review by SRE/Platform team<br>b) Self-service dengan guidelines<br>c) Tidak perlu approval | |

### 6.3 Distributed Tracing

| # | Pertanyaan | Opsi | Keputusan |
|---|------------|------|-----------|
| 6.3.1 | Tracing implementation? | a) OpenTelemetry<br>b) Jaeger client<br>c) Zipkin<br>d) Cloud provider (X-Ray, Cloud Trace) | |
| 6.3.2 | Trace context propagation? | a) W3C Trace Context<br>b) B3 (Zipkin)<br>c) Custom headers | |
| 6.3.3 | Sampling strategy? | a) Head-based sampling<br>b) Tail-based sampling<br>c) Always sample<br>d) Configurable rate | |
| 6.3.4 | Span attributes standard? | a) OpenTelemetry semantic conventions<br>b) Custom conventions<br>c) Minimal (service, operation saja) | |

### 6.4 Health Checks

| # | Pertanyaan | Opsi | Keputusan |
|---|------------|------|-----------|
| 6.4.1 | Health check endpoints? | a) /health (liveness) + /ready (readiness)<br>b) Single /health endpoint<br>c) gRPC Health protocol | |
| 6.4.2 | Health check depth? | a) Basic (service running)<br>b) Shallow (+ critical dependencies)<br>c) Deep (all dependencies) | |
| 6.4.3 | Health check response format? | a) Simple 200/503<br>b) JSON dengan details<br>c) Kubernetes probe compatible | |

### 6.5 Alerting

| # | Pertanyaan | Opsi | Keputusan |
|---|------------|------|-----------|
| 6.5.1 | Alert definition location? | a) Terraform/IaC<br>b) Prometheus rules di repo<br>c) UI based (Grafana/PagerDuty)<br>d) Centralized repo | |
| 6.5.2 | Standard alerts per service? | a) Ya, template wajib<br>b) Recommendations saja<br>c) Per-team decision | |
| 6.5.3 | Alert severity levels? | a) P1-P4<br>b) Critical, Warning, Info<br>c) Custom per team | |

---

## 7. Security Guidelines

### 7.1 Input Validation

| # | Pertanyaan | Opsi | Keputusan |
|---|------------|------|-----------|
| 7.1.1 | Validation library? | a) go-playground/validator<br>b) ozzo-validation<br>c) Custom validation<br>d) Proto validation (protoc-gen-validate) | |
| 7.1.2 | Validation location? | a) Handler/API layer<br>b) Domain/Service layer<br>c) Both layers | |
| 7.1.3 | Validation error response? | a) Field-level errors<br>b) First error only<br>c) Aggregated errors | |

### 7.2 Authentication & Authorization

| # | Pertanyaan | Opsi | Keputusan |
|---|------------|------|-----------|
| 7.2.1 | Auth token format? | a) JWT<br>b) Opaque token<br>c) API Key<br>d) Combination | |
| 7.2.2 | Auth validation location? | a) API Gateway<br>b) Per-service middleware<br>c) Both | |
| 7.2.3 | Service-to-service auth? | a) mTLS<br>b) JWT/Token<br>c) API Key<br>d) No auth (network level) | |
| 7.2.4 | Authorization pattern? | a) RBAC<br>b) ABAC<br>c) ReBAC<br>d) Custom per service | |
| 7.2.5 | Permission checking? | a) Middleware<br>b) In handler<br>c) Domain layer<br>d) External service (OPA) | |

### 7.3 Secrets Management

| # | Pertanyaan | Opsi | Keputusan |
|---|------------|------|-----------|
| 7.3.1 | Secrets storage? | a) HashiCorp Vault<br>b) Cloud secrets manager<br>c) Kubernetes secrets<br>d) Environment variables | |
| 7.3.2 | Secrets rotation? | a) Automatic rotation<br>b) Manual rotation schedule<br>c) On-demand | |
| 7.3.3 | Secrets in local dev? | a) .env file (gitignored)<br>b) Local vault<br>c) Shared dev secrets | |

### 7.4 Data Protection

| # | Pertanyaan | Opsi | Keputusan |
|---|------------|------|-----------|
| 7.4.1 | PII handling guidelines? | a) Strict encryption at rest<br>b) Tokenization<br>c) Pseudonymization<br>d) Case by case | |
| 7.4.2 | Data encryption at rest? | a) Database level<br>b) Application level<br>c) Both<br>d) Not required | |
| 7.4.3 | Sensitive data logging? | a) Never log PII<br>b) Masked/redacted<br>c) Separate secure logs | |

### 7.5 Security Scanning

| # | Pertanyaan | Opsi | Keputusan |
|---|------------|------|-----------|
| 7.5.1 | Dependency scanning? | a) Dependabot<br>b) Snyk<br>c) govulncheck<br>d) Multiple tools | |
| 7.5.2 | SAST tools? | a) gosec<br>b) semgrep<br>c) SonarQube<br>d) Tidak pakai | |
| 7.5.3 | Container scanning? | a) Trivy<br>b) Clair<br>c) Cloud provider scanner<br>d) Tidak pakai | |
| 7.5.4 | Scan enforcement? | a) Block deployment on critical<br>b) Warning only<br>c) Periodic audit | |

---

## 8. Git & CI/CD Workflow

### 8.1 Branching Strategy

| # | Pertanyaan | Opsi | Keputusan |
|---|------------|------|-----------|
| 8.1.1 | Branching model? | a) GitHub Flow (main + feature)<br>b) GitFlow (main, develop, feature, release)<br>c) Trunk-based development<br>d) Per-squad decision | |
| 8.1.2 | Branch naming convention? | a) type/description (feature/add-login)<br>b) JIRA-ID/description<br>c) username/description<br>d) Tidak di-enforce | |
| 8.1.3 | Protected branches? | a) main only<br>b) main + develop<br>c) main + release branches | |

### 8.2 Commit Standards

| # | Pertanyaan | Opsi | Keputusan |
|---|------------|------|-----------|
| 8.2.1 | Commit message format? | a) Conventional Commits<br>b) JIRA ID prefix<br>c) Free form<br>d) Custom format | |
| 8.2.2 | Commit message enforcement? | a) Pre-commit hook<br>b) CI check<br>c) Code review saja<br>d) Tidak di-enforce | |
| 8.2.3 | Commit granularity? | a) Atomic commits (one change per commit)<br>b) Feature-complete commits<br>c) Squash on merge | |

### 8.3 Pull Request

| # | Pertanyaan | Opsi | Keputusan |
|---|------------|------|-----------|
| 8.3.1 | PR template? | a) Ya, standardisasi template<br>b) Per-repo template<br>c) Tidak pakai template | |
| 8.3.2 | PR size limit? | a) Max 400 lines<br>b) Max 200 lines<br>c) Tidak dibatasi<br>d) Guidelines saja | |
| 8.3.3 | Required reviewers? | a) Min 1 reviewer<br>b) Min 2 reviewers<br>c) CODEOWNERS based<br>d) Per-squad policy | |
| 8.3.4 | Review SLA? | a) 24 hours<br>b) 48 hours<br>c) Tidak ada SLA | |

### 8.4 CI Pipeline

| # | Pertanyaan | Opsi | Keputusan |
|---|------------|------|-----------|
| 8.4.1 | CI platform? | a) GitHub Actions<br>b) GitLab CI<br>c) Jenkins<br>d) CircleCI<br>e) Other | |
| 8.4.2 | Required CI checks? | a) Lint + Test + Build<br>b) + Security scan<br>c) + Integration tests<br>d) Custom per repo | |
| 8.4.3 | CI check enforcement? | a) All checks must pass<br>b) Critical checks only<br>c) Advisory (non-blocking) | |

### 8.5 CD Pipeline

| # | Pertanyaan | Opsi | Keputusan |
|---|------------|------|-----------|
| 8.5.1 | Deployment strategy? | a) Rolling update<br>b) Blue-green<br>c) Canary<br>d) Per-service decision | |
| 8.5.2 | Environment progression? | a) dev → staging → prod<br>b) dev → prod (with feature flags)<br>c) Custom per team | |
| 8.5.3 | Production deployment approval? | a) Manual approval required<br>b) Automatic with criteria<br>c) Per-team decision | |
| 8.5.4 | Rollback strategy? | a) Automatic on failure<br>b) Manual trigger<br>c) Redeploy previous version | |

### 8.6 Code Quality Gates

| # | Pertanyaan | Opsi | Keputusan |
|---|------------|------|-----------|
| 8.6.1 | Linter wajib? | a) golangci-lint dengan preset config<br>b) Per-squad config<br>c) Tidak wajib | |
| 8.6.2 | Linter rules standardization? | a) Shared .golangci.yml<br>b) Minimum required rules<br>c) Per-repo decision | |
| 8.6.3 | Format enforcement? | a) gofmt (strict)<br>b) goimports<br>c) gofumpt<br>d) Tidak di-enforce | |

---

## 9. Documentation Standards

### 9.1 Code Documentation

| # | Pertanyaan | Opsi | Keputusan |
|---|------------|------|-----------|
| 9.1.1 | Godoc requirement? | a) All exported functions/types<br>b) Package level saja<br>c) Tidak wajib | |
| 9.1.2 | Godoc format? | a) Full sentences, proper punctuation<br>b) Brief description OK<br>c) Tidak di-enforce | |
| 9.1.3 | Code comment language? | a) English only<br>b) Bahasa Indonesia allowed<br>c) Per-team decision | |
| 9.1.4 | Example in godoc? | a) Wajib untuk complex functions<br>b) Recommended<br>c) Opsional | |

### 9.2 README Standards

| # | Pertanyaan | Opsi | Keputusan |
|---|------------|------|-----------|
| 9.2.1 | README template? | a) Ya, standardisasi template<br>b) Minimum sections required<br>c) Tidak di-enforce | |
| 9.2.2 | Required README sections? | a) Overview, Setup, Usage, API<br>b) Overview, Setup saja<br>c) Per-team decision | |
| 9.2.3 | Badges di README? | a) Standard badges (build, coverage)<br>b) Opsional<br>c) Tidak perlu | |

### 9.3 Architecture Decision Records (ADR)

| # | Pertanyaan | Opsi | Keputusan |
|---|------------|------|-----------|
| 9.3.1 | ADR wajib? | a) Ya, untuk semua major decisions<br>b) Untuk breaking changes saja<br>c) Recommended tapi tidak wajib<br>d) Tidak pakai ADR | |
| 9.3.2 | ADR format? | a) MADR (Markdown ADR)<br>b) Michael Nygard format<br>c) Custom format | |
| 9.3.3 | ADR location? | a) docs/adr/ di repo<br>b) Centralized documentation repo<br>c) Confluence/Wiki | |
| 9.3.4 | ADR lifecycle? | a) Proposed → Accepted → Deprecated<br>b) Draft → Approved<br>c) No formal lifecycle | |

### 9.4 API Documentation

| # | Pertanyaan | Opsi | Keputusan |
|---|------------|------|-----------|
| 9.4.1 | API docs platform? | a) Generated dari proto/code<br>b) Swagger UI<br>c) Postman collections<br>d) Confluence/Wiki | |
| 9.4.2 | API changelog? | a) Per-release changelog<br>b) Per-PR documentation<br>c) Git history saja | |
| 9.4.3 | Breaking change documentation? | a) Mandatory migration guide<br>b) Changelog entry saja<br>c) Tidak ada requirement | |

### 9.5 Runbook & Operations

| # | Pertanyaan | Opsi | Keputusan |
|---|------------|------|-----------|
| 9.5.1 | Runbook requirement? | a) Wajib sebelum production<br>b) Recommended<br>c) Tidak wajib | |
| 9.5.2 | Runbook template? | a) Standard template<br>b) Minimum sections<br>c) Free form | |
| 9.5.3 | Runbook location? | a) docs/runbooks/ di repo<br>b) Centralized wiki<br>c) Confluence/Notion | |
| 9.5.4 | Runbook review? | a) Review by SRE/on-call team<br>b) Self-review<br>c) Tidak ada review | |

---

## Notes Meeting

### Keputusan yang Sudah Dibuat

| Tanggal | Aspek | Keputusan | Alasan |
|---------|-------|-----------|--------|
| | | | |

### Parking Lot (Perlu Diskusi Lebih Lanjut)

- 

### Action Items

- [ ] 

---

**Tags:** #architecture #guidelines #development #meeting-prep #discussion
