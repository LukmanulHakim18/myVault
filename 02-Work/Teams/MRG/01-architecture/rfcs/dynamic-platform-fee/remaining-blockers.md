---
title: Dynamic Platform Fee - Remaining Blockers
tags:
  - design-doc
  - mrg
  - dynamic-platform-fee
  - blockers
created: '2026-02-13'
status: active
team: MRG
updated: '2026-03-05'
---
# Dynamic Platform Fee - Remaining Blockers

**Updated:** 2026-03-05  
**Status:** Active

---

## ✅ RESOLVED ITEMS

| Item | Resolution | Date |
|---|---|---|
| Foreign fee amount | `foreign_fee_increment = 1000` | 2026-03-05 |
| Corporate surcharge | `corporate_surcharge = 500` | 2026-03-05 |
| Urgency immediate | `+250` via dpf_rules | 2026-03-05 |
| Urgency advance | `+300` via dpf_rules | 2026-03-05 |
| Loyalty tier source | Header `User-Info.loyalty_tier` (bukan query) | 2026-03-05 |
| Rule stacking | Yes, semua bisa stack | 2026-03-02 |
| User Profile Service | Existing service + `is_foreign` flag | 2026-03-02 |
| Cap mechanism | Per service type | 2026-03-02 |
| Corporate detection | Payment method type per-transaksi | 2026-03-02 |
| New user detection | Order counter based (`<= 1`) | 2026-03-02 |
| Service type format | `product/service/pricing` | 2026-03-02 |
| Rounding | Round up to 1000 | 2026-03-02 |
| Double surcharge | Confirmed (foreign + AMEX) | 2026-03-02 |
| OO timeout handling | Treat as `order=0` (new user) | 2026-03-05 |
| Protocol | gRPC + HTTP/REST via gRPC-Gateway | 2026-03-05 |

---

## 🔴 CRITICAL BLOCKERS (0 items)

All critical blockers resolved. ✅

---

## ⚠️ PENDING COORDINATION

### 1. DBA Archive Exclusion (RFC-002)
**Status:** ⏳ Pending  
**Owner:** DBA Team  
**Note:** Tabel `order_completed_counter` dan `order_completed_processed` harus di-exclude dari archive cycle.

### 2. User Service Development (RFC-003)
**Status:** ⏳ Pending  
**Owner:** EM  
**Note:** Timeline untuk implementasi `is_foreign` flag di User Service.

---

## 📋 PHASE 2 ITEMS

| Item | Service | Status |
|---|---|---|
| Pickup zone (APSH, Event Area) | Polygon Service | ⏳ Contract TBD |
| Surge condition (demand = Extreme) | Demand Service | ⏳ Contract TBD |

---

## 📊 BLOCKERS SUMMARY

- **Critical (P0):** 0 items ✅
- **Pending Coordination (P1):** 2 items — DBA archive, User Service timeline
- **Phase 2 (P2):** 2 items — pickup zone, surge condition
- **Resolved:** 15 items ✅

**DPF Service implementation complete.** Remaining items are external dependencies.

---

## References

- [[RFC-001-dynamic-platform-fee-service]]
- [[RFC-002-order-orchestrator-counter]]
- [[RFC-003-user-service-foreign-flag]]
