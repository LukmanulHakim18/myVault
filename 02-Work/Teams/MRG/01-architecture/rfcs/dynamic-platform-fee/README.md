---
title: Dynamic Platform Fee - RFC Index
tags:
  - rfc
  - mrg
  - dynamic-platform-fee
  - index
created: '2026-02-19'
updated: '2026-03-05'
status: implemented
---
# Dynamic Platform Fee — RFC Index

## Overview

Dynamic Platform Fee (DPF) adalah service untuk menghitung platform fee secara dinamis berdasarkan berbagai modifier: user type, payment method, loyalty tier, service type, dan urgency.

**Status:** ✅ Implemented (Phase 1)

---

## Documents

| Document | Description | Status |
|---|---|---|
| [[RFC-001-dynamic-platform-fee-service]] | Main architecture & design | ✅ Implemented |
| [[RFC-002-order-orchestrator-counter]] | Order counter for new user & foreign threshold | Final |
| [[RFC-003-user-service-foreign-flag]] | Foreign flag detection (add payment, change phone) | Final |
| [[rule-engine-implementation]] | Go implementation guide | ✅ Implemented |
| [[test-scenarios]] | Test cases & scenarios | Draft |
| [[c4-diagrams]] | C4 architecture diagrams | Final |
| [[catatan]] | Notes & clarifications | Reference |
| [[remaining-blockers]] | Remaining blockers & status | Active |

---

## Architecture Summary

```
service-info ──POST /v1/platform-fees──► DPF
                                          │
                 Header: User-Info        │
                 (bbid, is_foreign,       │
                  loyalty_tier)           │
                                          │
                          ┌───────────────┘
                          ▼
                  Order Orchestrator
                  GET /v1/order/counter/{bbid}
                          │
                          ▼
                  Check: new user? (order <= 1)
                          │
                 ┌────────┴────────┐
                 │ YES             │ NO
                 ▼                 ▼
            return fee=0      Evaluate rules
                              Calculate fee
                              return result
```

---

## Data Sources

| Data | Source |
|---|---|
| `bbid`, `is_foreign`, `loyalty_tier` | Header `User-Info` (via aphrodite/metadata) |
| `total_completed_order` | Order Orchestrator |
| `payment_method.*`, `urgency_type` | Request body |
| Config & rules | PostgreSQL |
| Cache | Redis |

> **Note:** DPF tidak query Loyalty Service — loyalty tier sudah ada di header.

---

## Phase 1 — Core DPF ✅

| Modifier | Value | Source |
|---|---|---|
| New user exemption | fee = 0 jika order <= 1 | Order Orchestrator |
| Foreign surcharge | +Rp 1.000 | Header + order >= 6 |
| Corporate surcharge | +Rp 500 | payment_type = ECV/trip_voucher/cc_cp |
| BIN country surcharge | +Rp 1.000 | card.country_code != ID |
| AMEX surcharge | +Rp 500 | card.principal = AMEX |
| Urgency immediate | +Rp 250 | urgency_type = immediate |
| Urgency advance | +Rp 300 | urgency_type = advance |
| Loyalty discount | -Rp 500 ~ 2.000 | Header loyalty_tier |
| Rounding | Round up to 1000 | Calculated |

## Phase 2 — External Integration (Future)

| Modifier | Source |
|---|---|
| Pickup zone (APSH, Event Area) | Polygon Service |
| Surge condition (Extreme) | Demand Service |

---

## Service Config (Phase 1)

| Fleet | Base Fee | Cap Fee |
|---|---|---|
| BLUE/RIDE/ARGO | 4,000 | 50,000 |
| BLUE/RIDE/FIXED_RATE | 4,000 | 50,000 |
| SILVER/RIDE/ARGO | 7,000 | 70,000 |
| SILVER/RIDE/FIXED_RATE | 7,000 | 70,000 |
| GOLDEN_BIRD/RENT/TIME_BASED | 10,000 | 100,000 |
| CITITRANS/SHUTTLE/FIXED_RATE | 5,000 | 60,000 |

---

## Fee Calculation Formula

```
1. Guard: if order <= 1 → return fee=0, STOP

2. raw = base_platform_fee
       + foreign_add        (jika is_foreign && order >= 6)
       + corporate_add      (jika payment_type corporate)
       + bin_country_add    (jika card.country != ID)
       + card_principal_add (jika AMEX)
       + urgency_add        (immediate=+250 / advance=+300)
       - loyalty_discount   (dari tier)

3. rounded = ceil(raw / 1000) * 1000

4. final = min(rounded, cap_platform_fee)  ← cap
5. final = max(final, base_platform_fee)   ← floor
```

---

## Quick Links

- **Architecture:** [[RFC-001-dynamic-platform-fee-service]]
- **Implementation:** [[rule-engine-implementation]]
- **Test Cases:** [[test-scenarios]]
- **Blockers:** [[remaining-blockers]]
