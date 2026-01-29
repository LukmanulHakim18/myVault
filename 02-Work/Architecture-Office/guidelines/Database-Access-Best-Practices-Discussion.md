---
title: Database Access Best Practices - Discussion Points
type: discussion-document
owner: Architecture Team
status: draft
created: '2026-01-21'
related_meeting: 2026-01-20 Meeting VP Architect - Development Guidelines
parent: Development-Guidelines-Discussion-Points
---
# Database Access Best Practices - Discussion Points

> **Tujuan:** Dokumen ini berisi pertanyaan-pertanyaan teknis untuk standardisasi cara akses database yang baik dan mekanisme deteksi slow query.
> 
> **Related Meeting:** [[2026-01-20 Meeting VP Architect - Development Guidelines]]
> **Parent Document:** [[Development-Guidelines-Discussion-Points]]

---

## Status Keputusan

| # | Aspek | Status | PIC | Tanggal Decided |
|---|-------|--------|-----|-----------------|
| 1 | Connection Management | ⬜ Pending | - | - |
| 2 | Query Execution Flow | ⬜ Pending | - | - |
| 3 | Transaction Management | ⬜ Pending | - | - |
| 4 | Slow Query Detection | ⬜ Pending | - | - |
| 5 | Query Optimization | ⬜ Pending | - | - |
| 6 | Monitoring & Alerting | ⬜ Pending | - | - |
| 7 | Developer Workflow | ⬜ Pending | - | - |

---

## 1. Connection Management

### 1.1 Connection Pooling

| # | Pertanyaan | Opsi | Keputusan |
|---|------------|------|-----------|
| 1.1.1 | Siapa yang manage connection pool? | a) Setiap service manage sendiri<br>b) Shared library/package internal<br>c) Sidecar/proxy (PgBouncer, ProxySQL) | |
| 1.1.2 | Default max open connections? | a) 10<br>b) 25<br>c) 50<br>d) Based on formula (CPU * 2 + 1)<br>e) Per-service tuning | |
| 1.1.3 | Default max idle connections? | a) Same as max open<br>b) Half of max open<br>c) Fixed number (5-10)<br>d) Per-service tuning | |
| 1.1.4 | Connection max lifetime? | a) 5 minutes<br>b) 30 minutes<br>c) 1 hour<br>d) No limit<br>e) Configurable | |
| 1.1.5 | Connection max idle time? | a) 1 minute<br>b) 5 minutes<br>c) 10 minutes<br>d) No limit | |
| 1.1.6 | Bagaimana handle connection exhaustion? | a) Block dan wait<br>b) Return error immediately<br>c) Wait dengan timeout<br>d) Auto-scale pool | |

### 1.2 Connection Initialization

| # | Pertanyaan | Opsi | Keputusan |
|---|------------|------|-----------|
| 1.2.1 | Kapan establish connection? | a) Lazy (on first query)<br>b) Eager (on startup)<br>c) Hybrid (min pool on startup) | |
| 1.2.2 | Health check connection saat startup? | a) Ya, wajib ping<br>b) Ya, dengan simple query<br>c) Tidak perlu | |
| 1.2.3 | Retry strategy saat connection gagal? | a) Exponential backoff<br>b) Fixed interval retry<br>c) Circuit breaker<br>d) Fail fast | |
| 1.2.4 | Max retry attempts? | a) 3 times<br>b) 5 times<br>c) Unlimited dengan backoff<br>d) Configurable | |

### 1.3 Read/Write Splitting

| # | Pertanyaan | Opsi | Keputusan |
|---|------------|------|-----------|
| 1.3.1 | Apakah perlu read replica? | a) Ya, wajib untuk production<br>b) Opsional per-service<br>c) Tidak perlu | |
| 1.3.2 | Bagaimana routing read/write? | a) Explicit di code (ReadDB/WriteDB)<br>b) Automatic detection (SELECT vs others)<br>c) Proxy level (ProxySQL/PgBouncer)<br>d) DNS based | |
| 1.3.3 | Consistency requirement untuk read? | a) Strong consistency (always primary)<br>b) Eventual consistency OK<br>c) Per-query decision<br>d) Configurable lag tolerance | |
| 1.3.4 | Handle replication lag? | a) Application level check<br>b) Proxy handles<br>c) Accept lag<br>d) Fallback to primary | |

### 1.4 Connection String & Credentials

| # | Pertanyaan | Opsi | Keputusan |
|---|------------|------|-----------|
| 1.4.1 | Format connection string? | a) Full DSN string<br>b) Separate params (host, port, user, etc)<br>c) URL format | |
| 1.4.2 | SSL/TLS requirement? | a) Wajib di semua environment<br>b) Wajib di production saja<br>c) Opsional | |
| 1.4.3 | SSL mode? | a) require<br>b) verify-ca<br>c) verify-full<br>d) Per-environment | |
| 1.4.4 | Credential rotation handling? | a) Restart service<br>b) Hot reload<br>c) Connection pool refresh<br>d) Vault dynamic secrets | |

---

## 2. Query Execution Flow

### 2.1 Query Layer Architecture

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
| 2.1.1 | Layer mana yang interact dengan DB? | a) Repository layer saja<br>b) Repository + dedicated DB layer<br>c) Service boleh langsung ke DB | |
| 2.1.2 | Apakah perlu abstraction layer di atas driver? | a) Ya, wrapper internal<br>b) Tidak, pakai driver langsung<br>c) Pakai library (sqlx, etc) | |
| 2.1.3 | Query builder vs Raw SQL? | a) Raw SQL only<br>b) Query builder wajib<br>c) Raw untuk complex, builder untuk simple<br>d) Per-team decision | |

### 2.2 Context Propagation

| # | Pertanyaan | Opsi | Keputusan |
|---|------------|------|-----------|
| 2.2.1 | Context wajib di semua DB operations? | a) Ya, wajib<br>b) Opsional<br>c) Wajib untuk queries, opsional untuk connection | |
| 2.2.2 | Apa yang harus ada di context untuk DB? | a) Timeout saja<br>b) Timeout + trace ID<br>c) Timeout + trace ID + user ID<br>d) Custom per-service | |
| 2.2.3 | Default query timeout? | a) 5 seconds<br>b) 10 seconds<br>c) 30 seconds<br>d) No default (use context)<br>e) Per-query type | |
| 2.2.4 | Handle context cancellation? | a) Graceful cancel query<br>b) Let query complete<br>c) Cancel + rollback if in transaction | |

### 2.3 Query Execution Pattern

| # | Pertanyaan | Opsi | Keputusan |
|---|------------|------|-----------|
| 2.3.1 | Prepared statements usage? | a) Wajib semua query<br>b) Auto-prepared by driver<br>c) Manual untuk frequent queries<br>d) Tidak pakai (query langsung) | |
| 2.3.2 | Parameterized query enforcement? | a) Wajib, no string concatenation<br>b) Linter check<br>c) Code review saja | |
| 2.3.3 | Batch operations pattern? | a) Single transaction batch<br>b) Bulk insert syntax<br>c) Parallel individual inserts<br>d) Per-case decision | |
| 2.3.4 | Pagination default pattern? | a) LIMIT OFFSET<br>b) Cursor/Keyset pagination<br>c) Hybrid based on data size<br>d) Per-case decision | |

### 2.4 Result Handling

| # | Pertanyaan | Opsi | Keputusan |
|---|------------|------|-----------|
| 2.4.1 | Row scanning pattern? | a) Struct scanning (sqlx)<br>b) Manual scan<br>c) Reflection-based<br>d) Code generation | |
| 2.4.2 | NULL handling? | a) sql.NullXxx types<br>b) Pointer fields<br>c) Custom null types<br>d) COALESCE di query | |
| 2.4.3 | Large result set handling? | a) Streaming with rows.Next()<br>b) Pagination mandatory<br>c) Limit result + warning<br>d) Per-case decision | |
| 2.4.4 | Empty result handling? | a) Return empty slice/nil<br>b) Return sentinel error<br>c) Return (result, found bool) | |

---

## 3. Transaction Management

### 3.1 Transaction Boundary

| #     | Pertanyaan                         | Opsi                                                                                                                           | Keputusan |
| ----- | ---------------------------------- | ------------------------------------------------------------------------------------------------------------------------------ | --------- |
| 3.1.1 | Di layer mana transaction dimulai? | a) Service layer<br>b) Repository layer<br>c) Handler layer<br>d) Dedicated transaction layer                                  |           |
| 3.1.2 | Transaction pattern?               | a) Closure-based (WithTransaction(func))<br>b) Manual Begin/Commit/Rollback<br>c) Unit of Work pattern<br>d) Decorator pattern |           |
| 3.1.3 | Siapa yang handle rollback?        | a) Defer di transaction starter<br>b) Explicit di setiap error<br>c) Framework/library handles                                 |           |

**Contoh Closure Pattern:**
```go
// Option A: Closure-based
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
// Option B: Manual
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

| # | Pertanyaan | Opsi | Keputusan |
|---|------------|------|-----------|
| 3.1.4 | Pilih pattern mana? | a) Closure-based<br>b) Manual<br>c) Fleksibel per-case | |

### 3.2 Transaction Isolation

| # | Pertanyaan | Opsi | Keputusan |
|---|------------|------|-----------|
| 3.2.1 | Default isolation level? | a) Read Committed<br>b) Repeatable Read<br>c) Serializable<br>d) Database default | |
| 3.2.2 | Kapan pakai isolation level berbeda? | a) Documented per-case<br>b) Per-query annotation<br>c) Tidak boleh ubah default | |
| 3.2.3 | Read-only transaction? | a) Ya, untuk read-heavy operations<br>b) Tidak perlu<br>c) Per-case decision | |

### 3.3 Nested Transaction

| # | Pertanyaan | Opsi | Keputusan |
|---|------------|------|-----------|
| 3.3.1 | Support nested transaction? | a) Ya, dengan savepoints<br>b) Tidak support<br>c) Propagation (reuse existing tx) | |
| 3.3.2 | Bagaimana detect sudah dalam transaction? | a) Context value<br>b) Transaction manager<br>c) Interface check<br>d) Tidak perlu detect | |

**Contoh Propagation Pattern:**
```go
// Querier interface - bisa *sql.DB atau *sql.Tx
type Querier interface {
    ExecContext(ctx context.Context, query string, args ...interface{}) (sql.Result, error)
    QueryContext(ctx context.Context, query string, args ...interface{}) (*sql.Rows, error)
    QueryRowContext(ctx context.Context, query string, args ...interface{}) *sql.Row
}

// Repository menerima Querier
func (r *Repository) Create(ctx context.Context, q Querier, data Entity) error {
    _, err := q.ExecContext(ctx, "INSERT INTO ...", ...)
    return err
}
```

| # | Pertanyaan | Opsi | Keputusan |
|---|------------|------|-----------|
| 3.3.3 | Pakai Querier interface pattern? | a) Ya, wajib<br>b) Ya, recommended<br>c) Tidak, repository pegang connection sendiri | |

### 3.4 Transaction Timeout & Limits

| # | Pertanyaan | Opsi | Keputusan |
|---|------------|------|-----------|
| 3.4.1 | Max transaction duration? | a) 5 seconds<br>b) 30 seconds<br>c) 1 minute<br>d) No limit<br>e) Configurable | |
| 3.4.2 | Lock wait timeout? | a) 5 seconds<br>b) 10 seconds<br>c) Database default<br>d) Configurable | |
| 3.4.3 | Deadlock retry? | a) Ya, automatic retry<br>b) Ya, dengan limit<br>c) Tidak, fail immediately | |
| 3.4.4 | Long transaction alerting? | a) Ya, alert > threshold<br>b) Log warning saja<br>c) Tidak perlu | |

---

## 4. Slow Query Detection

### 4.1 Definition & Threshold

| # | Pertanyaan | Opsi | Keputusan |
|---|------------|------|-----------|
| 4.1.1 | Definisi slow query? | a) > 100ms<br>b) > 500ms<br>c) > 1s<br>d) Tiered (warning: 500ms, critical: 2s)<br>e) Per-query type | |
| 4.1.2 | Apakah threshold berbeda per environment? | a) Ya, lebih strict di production<br>b) Ya, lebih lenient di production<br>c) Sama semua environment | |
| 4.1.3 | Apakah threshold berbeda per query type? | a) Ya (SELECT vs INSERT vs UPDATE)<br>b) Ya (simple vs complex)<br>c) Tidak, sama semua | |

### 4.2 Detection Mechanism

```
┌─────────────────────────────────────────────────────────────────┐
│                     Slow Query Detection Flow                    │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│  ┌──────────┐    ┌──────────────┐    ┌──────────────────────┐  │
│  │  Query   │───▶│  Middleware/ │───▶│  Execute Query       │  │
│  │  Start   │    │  Interceptor │    │  + Measure Duration  │  │
│  └──────────┘    └──────────────┘    └──────────────────────┘  │
│                                                │                 │
│                                                ▼                 │
│                                       ┌──────────────────┐      │
│                                       │ Duration > Threshold?│   │
│                                       └──────────────────┘      │
│                                          │           │          │
│                                         Yes          No         │
│                                          │           │          │
│                                          ▼           ▼          │
│                               ┌─────────────────┐  ┌────────┐  │
│                               │ Log Slow Query  │  │ Normal │  │
│                               │ + Emit Metric   │  │  Log   │  │
│                               │ + Alert?        │  └────────┘  │
│                               └─────────────────┘               │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

| # | Pertanyaan | Opsi | Keputusan |
|---|------------|------|-----------|
| 4.2.1 | Di mana detection dilakukan? | a) Application level (middleware/interceptor)<br>b) Database level (pg_stat_statements)<br>c) APM tool<br>d) Semua level | |
| 4.2.2 | Bagaimana implement di application? | a) Custom middleware/wrapper<br>b) Library hooks (sqlx, pgx hooks)<br>c) OpenTelemetry instrumentation<br>d) AOP/decorator | |
| 4.2.3 | Apakah perlu capture query plan? | a) Ya, untuk slow queries<br>b) Ya, semua queries di development<br>c) Tidak perlu | |

### 4.3 Logging Slow Query

| # | Pertanyaan | Opsi | Keputusan |
|---|------------|------|-----------|
| 4.3.1 | Informasi apa yang di-log? | a) Query + duration saja<br>b) + Parameters<br>c) + Stack trace<br>d) + Query plan | |
| 4.3.2 | Parameter logging untuk slow query? | a) Full parameters<br>b) Masked/truncated<br>c) Tidak log parameters | |
| 4.3.3 | Log level untuk slow query? | a) WARN<br>b) ERROR<br>c) INFO dengan tag<br>d) Separate slow query log | |
| 4.3.4 | Format log slow query? | a) Structured JSON<br>b) Human readable<br>c) Both (configurable) | |

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
| 4.3.5 | Apakah format di atas sudah cukup? | a) Ya<br>b) Perlu tambahan fields<br>c) Terlalu banyak, kurangi | |

### 4.4 Metrics & Monitoring

| # | Pertanyaan | Opsi | Keputusan |
|---|------------|------|-----------|
| 4.4.1 | Metrics apa yang di-track? | a) Query count + duration histogram<br>b) + Error rate<br>c) + Rows affected<br>d) + Connection pool stats | |
| 4.4.2 | Metric labels/dimensions? | a) query_name, status<br>b) + table_name<br>c) + operation (SELECT/INSERT/etc)<br>d) Minimal labels | |
| 4.4.3 | Histogram buckets untuk duration? | a) 10ms, 50ms, 100ms, 500ms, 1s, 5s<br>b) 100ms, 500ms, 1s, 2s, 5s, 10s<br>c) Custom buckets<br>d) Default library buckets | |

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

| # | Pertanyaan | Opsi | Keputusan |
|---|------------|------|-----------|
| 4.4.4 | Metrics naming convention? | a) db_ prefix<br>b) sql_ prefix<br>c) {service}_ prefix<br>d) OpenTelemetry semantic conventions | |

---

## 5. Query Optimization Guidelines

### 5.1 Index Guidelines

| # | Pertanyaan | Opsi | Keputusan |
|---|------------|------|-----------|
| 5.1.1 | Index naming convention? | a) idx_{table}_{columns}<br>b) {table}_{columns}_idx<br>c) Tidak di-enforce | |
| 5.1.2 | Wajib index untuk foreign keys? | a) Ya, wajib<br>b) Recommended<br>c) Per-case decision | |
| 5.1.3 | Composite index guidelines? | a) Documented best practices<br>b) Review oleh DBA<br>c) Tidak ada guidelines | |
| 5.1.4 | Unused index monitoring? | a) Ya, regular audit<br>b) Tooling otomatis<br>c) Tidak monitor | |

### 5.2 Query Review Process

| # | Pertanyaan | Opsi | Keputusan |
|---|------------|------|-----------|
| 5.2.1 | EXPLAIN ANALYZE wajib untuk PR? | a) Ya, untuk semua query baru<br>b) Ya, untuk query complex saja<br>c) Recommended, tidak wajib<br>d) Tidak perlu | |
| 5.2.2 | N+1 query detection? | a) Linter/static analysis<br>b) Runtime detection + log<br>c) Code review saja<br>d) APM detection | |
| 5.2.3 | Query complexity limit? | a) Ya, max joins/subqueries<br>b) Ya, max execution time estimate<br>c) Tidak ada limit<br>d) Guidelines saja | |

### 5.3 Caching Strategy

| # | Pertanyaan | Opsi | Keputusan |
|---|------------|------|-----------|
| 5.3.1 | Query result caching? | a) Application level (Redis)<br>b) ORM/library level<br>c) Database level (materialized view)<br>d) Tidak pakai cache | |
| 5.3.2 | Cache invalidation strategy? | a) TTL based<br>b) Event-driven<br>c) Manual invalidation<br>d) Hybrid | |
| 5.3.3 | Kapan wajib pakai cache? | a) Query > X ms<br>b) Query > Y req/s<br>c) Per-case decision<br>d) Tidak wajib | |

---

## 6. Monitoring & Alerting

### 6.1 Database Health Monitoring

| # | Pertanyaan | Opsi | Keputusan |
|---|------------|------|-----------|
| 6.1.1 | Metrics dari database yang di-monitor? | a) Connections, queries/sec, replication lag<br>b) + CPU, memory, disk<br>c) + Lock stats, dead tuples<br>d) All available metrics | |
| 6.1.2 | Monitoring tool? | a) Cloud provider native<br>b) Prometheus + exporters<br>c) Datadog/NewRelic<br>d) Custom solution | |
| 6.1.3 | Dashboard wajib per-service? | a) Ya, template standar<br>b) Ya, minimal requirements<br>c) Recommended saja | |

### 6.2 Alert Configuration

| # | Pertanyaan | Opsi | Keputusan |
|---|------------|------|-----------|
| 6.2.1 | Alert untuk slow query rate? | a) > 1% queries slow<br>b) > 5% queries slow<br>c) > N slow queries per minute<br>d) Tidak alert, monitor saja | |
| 6.2.2 | Alert untuk connection pool? | a) > 80% utilization<br>b) Wait time > threshold<br>c) Error rate > threshold<br>d) All above | |
| 6.2.3 | Alert untuk long running queries? | a) > 30 seconds<br>b) > 1 minute<br>c) > 5 minutes<br>d) Per-query type | |
| 6.2.4 | Alert untuk replication lag? | a) > 1 second<br>b) > 5 seconds<br>c) > 30 seconds<br>d) Tidak alert | |

### 6.3 Incident Response

| # | Pertanyaan | Opsi | Keputusan |
|---|------------|------|-----------|
| 6.3.1 | Runbook untuk slow query incident? | a) Ya, wajib ada<br>b) Generic DB runbook<br>c) Tidak ada formal runbook | |
| 6.3.2 | Siapa yang handle DB performance issue? | a) Service owner<br>b) DBA team<br>c) Platform/SRE team<br>d) Escalation path defined | |
| 6.3.3 | Kill query threshold? | a) Auto-kill > X minutes<br>b) Manual approval<br>c) Never auto-kill | |

---

## 7. Developer Workflow

### 7.1 Local Development

| #     | Pertanyaan                    | Opsi                                                                                          | Keputusan |
| ----- | ----------------------------- | --------------------------------------------------------------------------------------------- | --------- |
| 7.1.1 | Local database setup?         | a) Docker compose<br>b) Shared dev database<br>c) Embedded/in-memory<br>d) Cloud dev instance |           |
| 7.1.2 | Seed data untuk development?  | a) Fixtures wajib<br>b) Factory pattern<br>c) Copy from staging<br>d) Minimal seed            |           |
| 7.1.3 | Query logging di development? | a) Log semua queries<br>b) Log slow queries saja<br>c) Configurable                           |           |

### 7.2 Code Review Checklist

| # | Pertanyaan | Opsi | Keputusan |
|---|------------|------|-----------|
| 7.2.1 | Apakah perlu DB review checklist? | a) Ya, formal checklist<br>b) Guidelines saja<br>c) Tidak perlu | |
| 7.2.2 | Siapa yang review query changes? | a) Any reviewer<br>b) DBA/Database expert<br>c) Designated per-team | |

**Draft DB Review Checklist:**
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
| 7.2.3 | Apakah checklist di atas sudah cukup? | a) Ya<br>b) Perlu ditambah<br>c) Terlalu banyak | |

### 7.3 Testing Database Code

| # | Pertanyaan | Opsi | Keputusan |
|---|------------|------|-----------|
| 7.3.1 | Unit test untuk repository layer? | a) Wajib dengan mock<br>b) Wajib dengan real DB (testcontainers)<br>c) Opsional | |
| 7.3.2 | Integration test dengan real DB? | a) Wajib untuk critical paths<br>b) Opsional<br>c) Semua repository methods | |
| 7.3.3 | Test isolation strategy? | a) Transaction rollback per test<br>b) Truncate tables<br>c) Separate database per test<br>d) Shared state OK | |

---

## Implementation Examples

### Example: Database Wrapper dengan Slow Query Detection

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
| 7.4.1 | Apakah perlu buat shared library seperti contoh di atas? | a) Ya, library internal<br>b) Setiap squad implement sendiri<br>c) Pakai existing library | |

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

**Tags:** #architecture #database #best-practices #slow-query #meeting-prep #discussion


---

## 8. Caching Strategy (Redis)

### 8.1 Cache Pattern Selection

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
| 8.1.1 | Default caching pattern? | a) Cache-Aside (most common)<br>b) Read-Through<br>c) Write-Through<br>d) Per-case decision | |
| 8.1.2 | Siapa yang manage cache logic? | a) Repository layer<br>b) Dedicated cache layer/service<br>c) Service layer<br>d) Decorator/middleware | |
| 8.1.3 | Cache library/client untuk Go? | a) go-redis/redis<br>b) redigo<br>c) rueidis (high performance)<br>d) Per-squad decision | |

### 8.2 Cache Key Design

| # | Pertanyaan | Opsi | Keputusan |
|---|------------|------|-----------|
| 8.2.1 | Key naming convention? | a) {service}:{entity}:{id}<br>b) {entity}:{id}:{version}<br>c) {prefix}:{hash}<br>d) Tidak di-enforce | |
| 8.2.2 | Key prefix per service? | a) Ya, wajib service prefix<br>b) Ya, wajib environment prefix<br>c) Opsional<br>d) Managed by Redis namespace | |
| 8.2.3 | Key length limit? | a) Max 100 characters<br>b) Max 200 characters<br>c) No limit<br>d) Hashed jika terlalu panjang | |
| 8.2.4 | Composite key handling? | a) Join dengan colon (:)<br>b) Join dengan hash<br>c) Serialize ke JSON lalu hash | |

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
| 8.2.5 | Apakah key naming convention di atas sudah sesuai? | a) Ya<br>b) Perlu adjustment<br>c) Buat yang berbeda | |

### 8.3 TTL (Time-To-Live) Strategy

| # | Pertanyaan | Opsi | Keputusan |
|---|------------|------|-----------|
| 8.3.1 | Default TTL? | a) 5 minutes<br>b) 15 minutes<br>c) 1 hour<br>d) No default, wajib explicit<br>e) Per-entity type | |
| 8.3.2 | TTL untuk static/reference data? | a) 1 hour<br>b) 24 hours<br>c) 1 week<br>d) No expiry (manual invalidation) | |
| 8.3.3 | TTL untuk user session data? | a) 15 minutes<br>b) 30 minutes<br>c) Match session timeout<br>d) Sliding expiration | |
| 8.3.4 | TTL jitter untuk prevent thundering herd? | a) Ya, random ±10%<br>b) Ya, random ±20%<br>c) Tidak perlu | |
| 8.3.5 | Boleh cache tanpa TTL? | a) Tidak boleh<br>b) Boleh untuk specific cases (documented)<br>c) Boleh dengan approval | |

### 8.4 Cache Invalidation

| # | Pertanyaan | Opsi | Keputusan |
|---|------------|------|-----------|
| 8.4.1 | Primary invalidation strategy? | a) TTL-based (let it expire)<br>b) Event-driven (on data change)<br>c) Manual/explicit delete<br>d) Hybrid (TTL + event) | |
| 8.4.2 | Event-driven invalidation mechanism? | a) Publish ke Redis pub/sub<br>b) Message queue (Kafka, RabbitMQ)<br>c) Database triggers/CDC<br>d) Application level events | |
| 8.4.3 | Invalidation scope? | a) Single key delete<br>b) Pattern-based delete (SCAN + DEL)<br>c) Tag-based invalidation<br>d) Namespace flush | |
| 8.4.4 | Cross-service cache invalidation? | a) Event bus/message queue<br>b) Direct API call<br>c) Shared cache namespace<br>d) Tidak support (TTL saja) | |

**Contoh Event-Driven Invalidation:**
```go
// Saat data di-update
func (s *OrderService) UpdateOrder(ctx context.Context, order *Order) error {
    // 1. Update database
    if err := s.repo.Update(ctx, order); err != nil {
        return err
    }
    
    // 2. Invalidate cache
    cacheKey := fmt.Sprintf("order-service:order:%s", order.ID)
    if err := s.cache.Delete(ctx, cacheKey); err != nil {
        // Log warning, don't fail the operation
        s.logger.Warn("failed to invalidate cache", "key", cacheKey, "error", err)
    }
    
    // 3. Publish event untuk cross-service invalidation (optional)
    s.eventBus.Publish(ctx, "order.updated", order.ID)
    
    return nil
}
```

| # | Pertanyaan | Opsi | Keputusan |
|---|------------|------|-----------|
| 8.4.5 | Apakah pattern invalidation di atas sudah sesuai? | a) Ya<br>b) Perlu adjustment<br>c) Buat yang berbeda | |

### 8.5 Serialization

| # | Pertanyaan | Opsi | Keputusan |
|---|------------|------|-----------|
| 8.5.1 | Serialization format? | a) JSON<br>b) MessagePack<br>c) Protocol Buffers<br>d) Gob (Go native)<br>e) Per-case decision | |
| 8.5.2 | Compression untuk large values? | a) Ya, always compress<br>b) Ya, jika > X KB<br>c) Tidak perlu<br>d) Per-case decision | |
| 8.5.3 | Schema versioning di cache? | a) Include version di value<br>b) Include version di key<br>c) Tidak perlu (TTL handles) | |

### 8.6 Redis Connection Management

| # | Pertanyaan | Opsi | Keputusan |
|---|------------|------|-----------|
| 8.6.1 | Connection pooling? | a) Ya, dengan pool size limit<br>b) Single connection<br>c) Library default | |
| 8.6.2 | Default pool size? | a) 10<br>b) 20<br>c) CPU * 2<br>d) Per-service tuning | |
| 8.6.3 | Connection timeout? | a) 1 second<br>b) 5 seconds<br>c) 10 seconds<br>d) Configurable | |
| 8.6.4 | Read/Write timeout? | a) 1 second<br>b) 3 seconds<br>c) 5 seconds<br>d) Per-operation | |
| 8.6.5 | Retry strategy? | a) Exponential backoff<br>b) Fixed retry<br>c) No retry (fail fast)<br>d) Circuit breaker | |

### 8.7 Redis Data Structures Usage

| # | Pertanyaan | Opsi | Keputusan |
|---|------------|------|-----------|
| 8.7.1 | Kapan pakai String vs Hash? | a) Guidelines documented<br>b) String untuk simple, Hash untuk objects<br>c) Per-case decision | |
| 8.7.2 | Kapan pakai List vs Sorted Set? | a) Guidelines documented<br>b) List untuk queue, ZSet untuk ranking<br>c) Per-case decision | |
| 8.7.3 | Boleh pakai Redis Streams? | a) Ya, untuk event streaming<br>b) Tidak, pakai dedicated MQ<br>c) Per-case dengan approval | |
| 8.7.4 | Boleh pakai Redis as primary datastore? | a) Tidak boleh<br>b) Boleh untuk ephemeral data<br>c) Per-case dengan approval | |

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
| 8.7.5 | Apakah guidelines data structure di atas sudah cukup? | a) Ya<br>b) Perlu detail lebih | |

### 8.8 Cache Failure Handling

| # | Pertanyaan | Opsi | Keputusan |
|---|------------|------|-----------|
| 8.8.1 | Behavior saat Redis down? | a) Fallback ke database<br>b) Return error<br>c) Serve stale data (if available)<br>d) Circuit breaker + fallback | |
| 8.8.2 | Cache miss handling? | a) Query DB, populate cache<br>b) Query DB, async populate<br>c) Query DB only (no populate) | |
| 8.8.3 | Thundering herd prevention? | a) Distributed lock (single populate)<br>b) Request coalescing<br>c) Probabilistic early expiration<br>d) Tidak di-handle | |
| 8.8.4 | Cache warming on startup? | a) Ya, critical data<br>b) Tidak, lazy load<br>c) Background warming | |

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

| # | Pertanyaan | Opsi | Keputusan |
|---|------------|------|-----------|
| 8.8.5 | Apakah perlu circuit breaker untuk cache? | a) Ya, wajib<br>b) Recommended<br>c) Tidak perlu | |

### 8.9 Cache Metrics & Monitoring

| # | Pertanyaan | Opsi | Keputusan |
|---|------------|------|-----------|
| 8.9.1 | Metrics yang di-track? | a) Hit/miss ratio saja<br>b) + Latency<br>c) + Error rate<br>d) + Memory usage | |
| 8.9.2 | Alert untuk cache hit ratio? | a) Ya, jika < 80%<br>b) Ya, jika < 90%<br>c) Monitor saja, no alert | |
| 8.9.3 | Alert untuk cache latency? | a) Ya, P99 > threshold<br>b) Monitor saja<br>c) Tidak perlu | |

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

## 9. Additional Developer Practices

### 9.1 Distributed Locking

| # | Pertanyaan | Opsi | Keputusan |
|---|------------|------|-----------|
| 9.1.1 | Distributed lock implementation? | a) Redis (Redlock/single instance)<br>b) Database advisory locks<br>c) etcd/Consul<br>d) Tidak support | |
| 9.1.2 | Lock timeout default? | a) 5 seconds<br>b) 30 seconds<br>c) No default (wajib explicit)<br>d) Per-operation | |
| 9.1.3 | Lock key convention? | a) lock:{resource}:{id}<br>b) {service}:lock:{resource}<br>c) Tidak di-enforce | |
| 9.1.4 | Kapan wajib pakai distributed lock? | a) Documented use cases<br>b) Per-case decision<br>c) Avoid locks, use other patterns | |

**Contoh Use Cases Distributed Lock:**
```go
// Prevent duplicate payment processing
func (s *PaymentService) ProcessPayment(ctx context.Context, orderID string) error {
    lockKey := fmt.Sprintf("lock:payment:%s", orderID)
    
    lock, err := s.locker.Acquire(ctx, lockKey, 30*time.Second)
    if err != nil {
        return fmt.Errorf("could not acquire lock: %w", err)
    }
    defer lock.Release(ctx)
    
    // Check if already processed (idempotency)
    if processed, _ := s.repo.IsPaymentProcessed(ctx, orderID); processed {
        return nil // Already done
    }
    
    // Process payment...
    return s.processPaymentInternal(ctx, orderID)
}
```

### 9.2 Idempotency

| # | Pertanyaan | Opsi | Keputusan |
|---|------------|------|-----------|
| 9.2.1 | Idempotency key storage? | a) Database table<br>b) Redis dengan TTL<br>c) Combination<br>d) Per-case decision | |
| 9.2.2 | Idempotency key format? | a) Client-provided<br>b) Server-generated (hash of request)<br>c) Both supported | |
| 9.2.3 | Idempotency window/TTL? | a) 24 hours<br>b) 7 days<br>c) Configurable per endpoint<br>d) No expiry | |
| 9.2.4 | Wajib untuk endpoint mana? | a) All mutating operations<br>b) Payment/financial only<br>c) Documented per-endpoint | |

### 9.3 Optimistic vs Pessimistic Locking

| # | Pertanyaan | Opsi | Keputusan |
|---|------------|------|-----------|
| 9.3.1 | Default locking strategy? | a) Optimistic (version column)<br>b) Pessimistic (SELECT FOR UPDATE)<br>c) Per-case decision | |
| 9.3.2 | Optimistic lock implementation? | a) Version number column<br>b) Updated_at timestamp<br>c) Hash of row content | |
| 9.3.3 | Retry on optimistic lock failure? | a) Ya, dengan limit<br>b) Tidak, return conflict error<br>c) Per-case decision | |

**Contoh Optimistic Locking:**
```go
// Entity dengan version
type Order struct {
    ID        string
    Status    string
    Version   int  // Optimistic lock version
    UpdatedAt time.Time
}

// Repository update dengan version check
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

### 9.4 Audit Logging

| # | Pertanyaan | Opsi | Keputusan |
|---|------------|------|-----------|
| 9.4.1 | Audit log untuk data changes? | a) Ya, wajib untuk semua entities<br>b) Ya, untuk critical entities saja<br>c) Opsional<br>d) Tidak perlu | |
| 9.4.2 | Audit log implementation? | a) Application level (before/after)<br>b) Database triggers<br>c) CDC (Change Data Capture)<br>d) Separate audit table | |
| 9.4.3 | Apa yang di-log? | a) Who, when, what changed<br>b) + Old value, new value<br>c) + Request context (IP, user agent) | |
| 9.4.4 | Audit log storage? | a) Same database<br>b) Separate database<br>c) Log aggregation system<br>d) Data warehouse | |

### 9.5 Soft Delete vs Hard Delete

| # | Pertanyaan | Opsi | Keputusan |
|---|------------|------|-----------|
| 9.5.1 | Default delete strategy? | a) Soft delete (deleted_at column)<br>b) Hard delete<br>c) Per-entity decision | |
| 9.5.2 | Soft delete column name? | a) deleted_at (timestamp)<br>b) is_deleted (boolean)<br>c) deleted (boolean) + deleted_at | |
| 9.5.3 | Query filtering untuk soft delete? | a) Automatic (repository level)<br>b) Manual di setiap query<br>c) Database view | |
| 9.5.4 | Permanent delete policy? | a) Never<br>b) After X days/months<br>c) Manual process dengan approval<br>d) Per data retention policy | |

### 9.6 Database Migration Best Practices

| # | Pertanyaan | Opsi | Keputusan |
|---|------------|------|-----------|
| 9.6.1 | Backward compatible migrations wajib? | a) Ya, selalu<br>b) Ya, untuk production<br>c) Tidak wajib | |
| 9.6.2 | Breaking change procedure? | a) Blue-green database<br>b) Expand-contract pattern<br>c) Downtime window<br>d) Per-case decision | |
| 9.6.3 | Data migration separation? | a) Separate dari schema migration<br>b) Combined OK<br>c) Per-case decision | |
| 9.6.4 | Large table migration strategy? | a) Online schema change (pt-osc, gh-ost)<br>b) Batch migration<br>c) Maintenance window<br>d) Per-case decision | |

**Contoh Expand-Contract Migration:**
```
# Phase 1: Expand (add new column, keep old)
ALTER TABLE users ADD COLUMN email_new VARCHAR(255);

# Phase 2: Migrate data
UPDATE users SET email_new = LOWER(email) WHERE email_new IS NULL;

# Phase 3: Application code uses both columns
# (write to both, read from new)

# Phase 4: Contract (remove old column) - after all instances updated
ALTER TABLE users DROP COLUMN email;
ALTER TABLE users RENAME COLUMN email_new TO email;
```

### 9.7 Data Archival

| # | Pertanyaan | Opsi | Keputusan |
|---|------------|------|-----------|
| 9.7.1 | Data archival strategy? | a) Separate archive tables<br>b) Separate archive database<br>c) Cold storage (S3, etc)<br>d) Tidak ada archival | |
| 9.7.2 | Archival trigger? | a) Age-based (> X months)<br>b) Status-based (completed/closed)<br>c) Manual process<br>d) Combination | |
| 9.7.3 | Archived data accessibility? | a) Read-only queries<br>b) Restore on request<br>c) Not accessible from app | |

### 9.8 Multi-tenancy (jika applicable)

| # | Pertanyaan | Opsi | Keputusan |
|---|------------|------|-----------|
| 9.8.1 | Multi-tenancy model? | a) Shared database, shared schema<br>b) Shared database, separate schema<br>c) Separate database per tenant<br>d) Tidak multi-tenant | |
| 9.8.2 | Tenant isolation enforcement? | a) Application level (WHERE tenant_id = ?)<br>b) Row-level security (RLS)<br>c) Connection-level (set search_path)<br>d) Database per tenant | |
| 9.8.3 | Tenant context propagation? | a) Context value<br>b) Request header<br>c) JWT claim<br>d) Connection string | |

### 9.9 Graceful Degradation

| # | Pertanyaan | Opsi | Keputusan |
|---|------------|------|-----------|
| 9.9.1 | Behavior saat database partial failure? | a) Fail entire request<br>b) Return partial data<br>c) Serve cached/stale data<br>d) Per-endpoint decision | |
| 9.9.2 | Read replica failover? | a) Automatic ke primary<br>b) Return error<br>c) Serve stale dari cache | |
| 9.9.3 | Circuit breaker untuk database? | a) Ya, wajib<br>b) Recommended<br>c) Tidak perlu | |
| 9.9.4 | Bulkhead pattern untuk database? | a) Ya, separate pools per feature<br>b) Tidak, single pool<br>c) Per-case decision | |

### 9.10 Rate Limiting di Database Level

| # | Pertanyaan | Opsi | Keputusan |
|---|------------|------|-----------|
| 9.10.1 | Rate limit untuk expensive queries? | a) Ya, application level<br>b) Ya, database level (statement timeout)<br>c) Tidak perlu | |
| 9.10.2 | Connection rate limiting? | a) Ya, max connections per service<br>b) Ya, via PgBouncer/ProxySQL<br>c) Database default limits | |
| 9.10.3 | Query concurrency limit? | a) Ya, semaphore pattern<br>b) Ya, worker pool<br>c) Tidak, rely on connection pool | |

---

## Summary: Semua Aspek Database Best Practices

| # | Aspek | Deskripsi | Jumlah Pertanyaan |
|---|-------|-----------|-------------------|
| 1 | Connection Management | Pooling, init, read/write split, credentials | 18 |
| 2 | Query Execution Flow | Architecture, context, patterns, result handling | 15 |
| 3 | Transaction Management | Boundary, isolation, nested, timeout | 14 |
| 4 | Slow Query Detection | Threshold, mechanism, logging, metrics | 16 |
| 5 | Query Optimization | Index, review process, caching | 10 |
| 6 | Monitoring & Alerting | Health, alerts, incident response | 10 |
| 7 | Developer Workflow | Local dev, code review, testing | 10 |
| 8 | Caching Strategy (Redis) | Patterns, keys, TTL, invalidation, failure handling | 35 |
| 9 | Additional Practices | Locking, idempotency, audit, archival, etc | 32 |
| | **TOTAL** | | **~160 pertanyaan** |

---

**Tags:** #architecture #database #redis #caching #best-practices #meeting-prep
