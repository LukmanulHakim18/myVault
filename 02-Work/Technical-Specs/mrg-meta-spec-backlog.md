---
title: Meta-Spec Assessment Backlog — MRG Services
tags:
  - meta-spec
  - backlog
  - MRG
  - assessment
status: in-progress
owner: lukmanul.hakim
created: '2026-04-20T00:00:00.000Z'
updated: '2026-04-23T00:00:00.000Z'
sync-bluelink: synced
---

# Meta-Spec Assessment Backlog — MRG Services

Tracking pengerjaan meta-spec **bertahap** untuk microservice MRG yang belum terdokumentasi.
Satu service selesai, baru lanjut ke berikutnya.

> **Update 2026-04-23** — Rekonsiliasi ulang vs Bluelink live list.
> Promo Gateway dipindah dari backlog → sudah ada di Bluelink.
> 5 service ⏳ pending sync dikonfirmasi ✅.
> **Total MRG: 41 service** | ✅ Selesai: 21 | ❌ Backlog: 20

---

## Cara Kerja

1. Pilih satu service dari backlog di bawah
2. Buat folder di Bluelink: `services/MRG/{service-name}/meta-spec/`
3. Minimal buat 3 file: `README.md`, `api-reference.md`, `dependencies.md`
4. Centang item setelah selesai
5. Update field `updated` di frontmatter note ini

---

## ❌ Backlog — Belum Ada Meta-Spec (20 service)

### 🔴 Core / Gateway

- [ ] **GB Gateway** (`goldenbird-gateway`)
- [ ] **GB Order Processor** (`gb-order-processor`)
- [ ] **UPG Gateway** (`upg-gateway`)
- [ ] **AI Gateway** (`ai-gateway`)

### 🟠 Forwarder / Proxy

- [ ] **Stream Forwarder** (`stream-forwarder`)
- [ ] **UPG Forwarder** (`upg-forwarder`)
- [ ] **GB Partner Forwarder** (`gb-partner-forwarder`)
- [ ] **Taxi Partner Forwarder** (`taxi-partner-forwarder`)
- [ ] **Notification Forwarder** (`notification-forwarder`)

### 🟡 Pricing & Fee

- [ ] **Price Calculator** (`price-calculator`)
- [ ] **Dynamic Pricing** (`dynamic-pricing`)
- [ ] **Price Manager** (`price-manager`)
- [ ] **Dynamic Price** (`dynamic-price`)
- [ ] **Fare Adjustment Catalog** (`fare-adjustment-catalog`)

### 🟢 Order & Partner

- [ ] **Subscription Service** (`subscription-service`)
- [ ] **Gorooster** (`gorooster`)
- [ ] **Gorooster Forwarder** (`gorooster-forwarder`)

### 🔵 Shuttle / Cititrans

- [ ] **Cititrans Gateway** (`cititrans`)

### ⚪ Lainnya

- [ ] **Content Provider** (`content-provider`)
- [ ] **School Bus Gateway** (`school-bus-gateway`)

---

## ✅ Sudah Ada Meta-Spec (21 service)

| # | Service Name | POD Name | Bluelink Path | Status |
|---|-------------|----------|---------------|--------|
| 1 | Fraud Service | `fds-service` | `services/MRG/fds/meta-spec/` | ✅ Bluelink |
| 2 | Notification Center | `notification-center` | `services/MRG/notificationcenter/meta-spec/` | ✅ Bluelink |
| 3 | Dynamic Platform Fee | `dpf-service` | `services/MRG/dpf/meta-spec/` | ✅ Bluelink |
| 4 | Order Query | `order-query` | `services/MRG/orderquery/meta-spec/` | ✅ Bluelink |
| 5 | Taxi Partner Gateway | `taxi-partner-gateway` | `services/MRG/taxipartnergateway/meta-spec/` | ✅ Bluelink |
| 6 | Payment Processor | `payment-processor` | `services/MRG/paymentprocessor/meta-spec/` | ✅ Bluelink |
| 7 | Service Info | `service-info` | `services/MRG/serviceinfo/meta-spec/` | ✅ Bluelink |
| 8 | Tracker Service | `tracker-service` | `services/MRG/trackerservice/meta-spec/` | ✅ Bluelink |
| 9 | User Service | `user-service` | `services/MRG/userservice/meta-spec/` | ✅ Bluelink |
| 10 | Auth Service | `auth-service` | `services/MRG/authservice/meta-spec/` | ✅ Bluelink |
| 11 | Geo Service | `geo-service` | `services/MRG/geoservice/meta-spec/` | ✅ Bluelink |
| 12 | Session Manager | `session-manager` | `services/MRG/sessionmanager/meta-spec/` | ✅ Bluelink |
| 13 | Webhook Forwarder | `webhook-forwarder` | `services/MRG/webhookforwarder/meta-spec/` | ✅ Bluelink |
| 14 | Order Orchestrator | `order-orchestrator` | `services/MRG/orderorchestrator/meta-spec/` | ✅ Bluelink |
| 15 | N2C Service | `n2cservice` | `services/MRG/n2cservice/meta-spec/` | ✅ Bluelink |
| 16 | ConfigService | `config-service` | `services/MRG/configservice/meta-spec/` | ✅ Bluelink |
| 17 | Taxi Order Processor | `taxi-order-processor` | `services/MRG/top/` | ✅ Bluelink |
| 18 | Cititrans Order Processor | `cititrans-order-processor` | `services/MRG/cop/meta-spec/` | ✅ Bluelink |
| 19 | Routine Manager | `routine-manager` | `services/MRG/routinemanager/meta-spec/` | ✅ Bluelink |
| 20 | HO Gateway | `ho-gateway` | `services/MRG/mybb-ho-gateway/meta-spec/` | ✅ Bluelink |
| 21 | Promo Gateway | `promo-gateway` | `services/MRG/promogateway/meta-spec/` | ✅ Bluelink |

---

## 🚫 Dikeluarkan dari MRG (bukan service MRG)

| Service | POD Name | Alasan |
|---------|----------|--------|
| Order Receiver | `order-receiver` | Bukan MRG |
| BB Proxy | `bb-proxy-service` | Bukan MRG |

---

_Update terakhir: 2026-04-23 — rekonsiliasi live vs Bluelink list, Promo Gateway confirmed ✅, 5 pending sync dikonfirmasi ✅_
