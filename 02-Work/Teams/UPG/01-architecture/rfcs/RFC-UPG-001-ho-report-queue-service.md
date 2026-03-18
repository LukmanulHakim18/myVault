---
title: 'RFC: HO Report Queue Service'
tags:
  - rfc
  - upg
  - ho-report
  - architecture
status: draft
created: '2026-02-13'
author: UPG Team
---
# RFC: HO Report Queue Service

## Metadata

| Field | Value |
|-------|-------|
| RFC ID | UPG-RFC-001 |
| Title | HO Report Queue Service |
| Author | UPG Team |
| Status | Draft |
| Created | 2026-02-13 |

---

## 1. Summary

Service baru untuk menangani pelaporan charging ke HO secara sequential dan guaranteed delivery. Service ini menjadi intermediary antara `card_payment` dan HO untuk menghindari HO dibom dengan multiple request secara bersamaan.

---

## 2. Problem Statement

1. HO tidak kuat menerima multiple push secara bersamaan
2. Reporting charging harus sequential per order
3. Harus guaranteed delivery (tidak boleh ada yang hilang)

### Contoh Kasus

Order 430k dengan preauth terakhir 290k:

| Step | Action | Result | Sisa Utang |
|------|--------|--------|------------|
| 1 | Full charge 430k | GAGAL | - |
| 2 | Charge 290k (sesuai preauth) | SUCCESS | 140k |
| 3 | Charge 140k (immediate) | SUCCESS | 0 |

Push ke HO harus urut: gagal → 290k → 140k

---

## 3. Goals

1. Sequential processing per order
2. Guaranteed delivery ke HO
3. Retry mechanism dengan alerting
4. Tidak membebani HO dengan concurrent requests

---

## 4. Non-Goals

1. Real-time processing (acceptable delay via scheduler)
2. Transformasi data (pass-through ke HO)

---

## 5. Architecture

### 5.1 High Level

```
card_payment ──RabbitMQ──> ho-report-queue ──REST──> HO
                               │
                               ├── PostgreSQL (dedicated)
                               ├── Redis (existing UPG)
                               └── GMS (alert)
```

### 5.2 Components

| Component | Type | Responsibility |
|-----------|------|----------------|
| Consumer | Long-running pod | Consume RabbitMQ, persist ke DB |
| Scheduler | K8s CronJob | Batch process, push ke HO |
| DB | PostgreSQL | Store parent + item |
| Redis | Existing UPG | Distributed lock |
| GMS Client | - | Alert untuk `need_handle` |

### 5.3 Flow Diagram

```
┌─────────────┐     ┌─────────────┐     ┌─────────────────┐
│card_payment │────>│  RabbitMQ   │────>│  ho-report-queue│
└─────────────┘     └─────────────┘     │    (Consumer)   │
                                        └────────┬────────┘
                                                 │
                                                 ▼
                                        ┌─────────────────┐
                                        │   PostgreSQL    │
                                        │  (parent/item)  │
                                        └────────┬────────┘
                                                 │
                    ┌────────────────────────────┴───────┐
                    │                                    │
                    ▼                                    ▼
           ┌─────────────────┐                 ┌─────────────────┐
           │    Scheduler    │                 │      Redis      │
           │   (K8s CronJob) │────lock────────>│   (UPG existing)│
           └────────┬────────┘                 └─────────────────┘
                    │
                    ▼
           ┌─────────────────┐
           │       HO        │
           └────────┬────────┘
                    │
                    ▼ (if attempt > 3)
           ┌─────────────────┐
           │       GMS       │
           └─────────────────┘
```

---

## 6. Detailed Design

### 6.1 Consumer Flow

```
1. Consume message dari RabbitMQ
2. Upsert parent by order_id
3. Insert item dengan status "pending"
4. Ack message
```

### 6.2 Scheduler Flow

```
1. Get cursor dari Redis (default 0)
2. Fetch parents dengan pending items (WHERE id > cursor, LIMIT 100)
3. Jika kosong, reset cursor, exit
4. Loop per parent:
   a. Cek context (SIGTERM?)
   b. Try lock Redis SETNX
   c. Jika gagal lock, skip ke next parent (continue)
   d. Fetch items ORDER BY transaction_time
   e. Loop per item:
      - status "success" → skip ke next item
      - status "need_handle" → break, skip ke next parent
      - status "pending" → push ke HO
        - success → update status "success"
        - fail → increment attempt
          - attempt > 3 → status "need_handle", alert GMS
          - break, skip ke next parent
   f. Release lock
   g. Update cursor
5. Exit
```

### 6.3 Graceful Shutdown

```
1. SIGTERM received
2. Cancel context
3. Stop processing new parent
4. Finish current parent
5. Release lock
6. Update cursor
7. Exit
```

---

## 7. Data Design

### 7.1 Tables

```sql
-- Parent table
CREATE TABLE parent (
    id BIGSERIAL PRIMARY KEY,
    order_id VARCHAR(50) NOT NULL,
    created_at TIMESTAMP NOT NULL DEFAULT NOW(),
    updated_at TIMESTAMP NOT NULL DEFAULT NOW(),
    
    CONSTRAINT uq_parent_order_id UNIQUE (order_id)
);

CREATE INDEX idx_parent_order_id ON parent(order_id);

-- Item table
CREATE TABLE item (
    id BIGSERIAL PRIMARY KEY,
    parent_id BIGINT NOT NULL,
    transaction_time TIMESTAMP NOT NULL,
    request JSONB NOT NULL,
    response JSONB,
    status VARCHAR(20) NOT NULL DEFAULT 'pending',
    attempt INT NOT NULL DEFAULT 0,
    created_at TIMESTAMP NOT NULL DEFAULT NOW(),
    updated_at TIMESTAMP NOT NULL DEFAULT NOW(),
    
    CONSTRAINT fk_item_parent FOREIGN KEY (parent_id) REFERENCES parent(id),
    CONSTRAINT chk_item_status CHECK (status IN ('pending', 'success', 'need_handle'))
);

CREATE INDEX idx_item_parent_id ON item(parent_id);
CREATE INDEX idx_item_status ON item(status);
CREATE INDEX idx_item_parent_status ON item(parent_id, status);
```

### 7.2 ER Diagram

```
┌─────────────────────────┐
│         parent          │
├─────────────────────────┤
│ id          BIGSERIAL   │──┐
│ order_id    VARCHAR(50) │  │
│ created_at  TIMESTAMP   │  │
│ updated_at  TIMESTAMP   │  │
└─────────────────────────┘  │
                             │ 1:N
┌─────────────────────────┐  │
│          item           │  │
├─────────────────────────┤  │
│ id          BIGSERIAL   │  │
│ parent_id   BIGINT      │──┘
│ transaction_time TIMESTAMP
│ request     JSONB       │
│ response    JSONB       │
│ status      VARCHAR(20) │
│ attempt     INT         │
│ created_at  TIMESTAMP   │
│ updated_at  TIMESTAMP   │
└─────────────────────────┘
```

### 7.3 Status Flow

```
pending ──success──> success
   │
   └──fail (attempt > 3)──> need_handle
```

---

## 8. Configuration

| Config | Default | Keterangan |
|--------|---------|------------|
| `SCHEDULER_INTERVAL` | `*/5 * * * *` | Cron expression |
| `SCHEDULER_BATCH_SIZE` | `100` | Max parent per cycle |
| `LOCK_TTL` | `10m` | Redis lock TTL |
| `MAX_ATTEMPT` | `3` | Max retry sebelum need_handle |
| `ACTIVE_DEADLINE_SECONDS` | `540` | K8s job timeout (9 menit) |

---

## 9. Failure Scenarios

| Scenario | Behavior |
|----------|----------|
| Consumer crash | Message requeue (RabbitMQ) |
| Scheduler crash mid-process | Lock auto-expire 10 menit, cursor tersimpan, next cycle lanjut |
| HO timeout | Item attempt++, retry next cycle |
| HO down prolonged | Item jadi `need_handle` setelah 3x, alert GMS |
| DB down | Consumer/Scheduler fail, restart by K8s |
| Redis down | Lock fail, scheduler skip semua parent |

---

## 10. Monitoring & Alerting

| Metric | Alert Condition |
|--------|-----------------|
| Item `need_handle` | Immediate alert ke GMS |
| Scheduler execution time | > 8 menit |
| Consumer lag | RabbitMQ queue depth > threshold |
| Failed push to HO | Error rate > threshold |

---

## 11. Security

- RabbitMQ: credential via K8s secret
- PostgreSQL: credential via K8s secret
- Redis: credential via K8s secret (existing UPG)
- HO endpoint: mTLS / API key (TBD)

---

## 12. Dependencies

| Dependency | Type | Notes |
|------------|------|-------|
| RabbitMQ | Existing | UPG infrastructure |
| Redis | Existing | UPG infrastructure |
| PostgreSQL | New | Dedicated instance |
| HO | External | REST API |
| GMS | External | Alert system |

---

## 13. Rollout Plan

| Phase | Action |
|-------|--------|
| 1 | Deploy DB, migrate schema |
| 2 | Deploy consumer (shadow mode, consume tapi tidak process) |
| 3 | Deploy scheduler (disabled) |
| 4 | Enable consumer (mulai persist) |
| 5 | Enable scheduler (mulai push ke HO) |
| 6 | Monitor & tuning |

---

## 14. Open Questions

1. API contract RabbitMQ message (discuss dengan UPG)
2. API contract HO payload (discuss dengan UPG)
3. GMS alert format (discuss dengan UPG)
4. HO authentication method (mTLS / API key?)

---

## Appendix

### A. Redis Key Pattern

| Key | Purpose | TTL |
|-----|---------|-----|
| `ho_report:lock:{parent_id}` | Distributed lock per parent | 10 menit |
| `ho_report:scheduler:cursor` | Cursor untuk batch processing | 10 menit |

### B. Kubernetes Manifest

#### CronJob

```yaml
apiVersion: batch/v1
kind: CronJob
metadata:
  name: ho-report-scheduler
  namespace: upg
spec:
  schedule: "*/5 * * * *"
  concurrencyPolicy: Forbid
  successfulJobsHistoryLimit: 3
  failedJobsHistoryLimit: 3
  jobTemplate:
    spec:
      backoffLimit: 1
      activeDeadlineSeconds: 540
      template:
        spec:
          restartPolicy: Never
          containers:
            - name: ho-report-scheduler
              image: ho-report-queue:latest
              args: ["scheduler"]
              envFrom:
                - configMapRef:
                    name: ho-report-queue-config
                - secretRef:
                    name: ho-report-queue-secret
```

### C. Tech Stack

| Stack | Detail |
|-------|--------|
| Language | Go |
| Database | PostgreSQL (new instance) |
| Message Queue | RabbitMQ (existing UPG) |
| Cache/Lock | Redis (existing UPG) |
| Scheduler | Kubernetes CronJob |
| Alert | GMS |
