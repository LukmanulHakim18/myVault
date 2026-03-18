---
title: Dynamic Platform Fee - Catatan
tags:
  - notes
  - mrg
  - dynamic-platform-fee
created: '2026-02-18'
updated: '2026-03-05'
---
# Dynamic Platform Fee — Catatan & Klarifikasi

**Updated:** 2026-03-05

---

## Key Decisions Log

| Date | Decision | Rationale |
|---|---|---|
| 2026-02-19 | Foreign flag irreversible | Business requirement — once foreign, always foreign |
| 2026-02-27 | RFC-004 merged ke RFC-001 | Consolidation — semua modifier dalam satu dokumen |
| 2026-03-02 | New user dari order counter, bukan User Service | Simplifikasi — tidak perlu endpoint baru |
| 2026-03-02 | Corporate detection per-transaksi | Flexibility — user sama bisa retail/corporate |
| 2026-03-02 | Rounding sebelum cap check | Business requirement — round up dulu baru cek cap |
| 2026-03-05 | **Loyalty tier dari header, bukan query** | Mengurangi latency & dependency; caller sudah punya data |
| 2026-03-05 | OO timeout → treat as `order=0` (new user) | Fail-safe: lebih baik gratis daripada error fleet list |
| 2026-03-05 | gRPC + HTTP/REST via gRPC-Gateway | Mengikuti Enera Plus pattern internal |

---

## Data Sources — Final

| Data | Source | Method |
|---|---|---|
| `bbid` | Header `User-Info` | `aphrodite/metadata.GetBBID()` |
| `is_foreign` | Header `User-Info` | `userInfo.IsForeign` |
| `loyalty_tier` | Header `User-Info` | `userInfo.LoyaltyTier` (sudah uppercase) |
| `total_completed_order` | Order Orchestrator | `GET /v1/order/counter/{bbid}` |
| `payment_method.*` | Request body | Dari caller |
| `urgency_type` | Request body | `immediate` / `advance` |
| Config & thresholds | DB `dpf_global_config` | Loaded once per request |
| Rules | DB `dpf_rules` | Lazy-loaded jika cache miss |

> **Note:** DPF **tidak lagi** query Loyalty Service. Semua data user context dari header.

---

## Nilai Final dari Implementasi

### Global Config

| config_key | value | keterangan |
|---|---|---|
| `feature_enabled` | true | Master switch |
| `new_user_order_threshold` | 1 | Order <= 1 → fee = 0 |
| `foreign_order_threshold` | 6 | Order >= 6 untuk kena foreign surcharge |
| `foreign_fee_increment` | 1000 | +Rp 1.000 |
| `corporate_surcharge` | 500 | +Rp 500 |
| `amex_surcharge` | 500 | +Rp 500 |
| `loyalty_discount_explorer` | 500 | -Rp 500 |
| `loyalty_discount_traveler` | 1000 | -Rp 1.000 |
| `loyalty_discount_voyager` | 1500 | -Rp 1.500 |
| `loyalty_discount_elite` | 2000 | -Rp 2.000 |
| `cache_ttl_seconds` | 3600 | 1 jam |
| `default_platform_fee` | 4000 | Fallback |

### Rules (dpf_rules)

| parameter | adjustment | keterangan |
|---|---|---|
| is_foreign | +1000 | Foreign user surcharge |
| payment_type (corporate) | +500 | ECV/trip_voucher/cc_cp |
| card_country (non-ID) | +1000 | Foreign card surcharge |
| card_principal (AMEX) | +500 | AMEX surcharge |
| urgency_type (immediate) | +250 | Booking segera |
| urgency_type (advance) | +300 | Booking di muka |

### Service Config (Phase 1)

| product | service | pricing | base | cap |
|---|---|---|---|---|
| BLUE | RIDE | ARGO | 4,000 | 50,000 |
| BLUE | RIDE | FIXED_RATE | 4,000 | 50,000 |
| SILVER | RIDE | ARGO | 7,000 | 70,000 |
| SILVER | RIDE | FIXED_RATE | 7,000 | 70,000 |
| GOLDEN_BIRD | RENT | TIME_BASED | 10,000 | 100,000 |
| CITITRANS | SHUTTLE | FIXED_RATE | 5,000 | 60,000 |

---

## Fase Implementasi

### Phase 1 — Core DPF ✅ Implemented

| Modifier | Source | Status |
|---|---|---|
| New user exemption | `total_completed_order <= 1` dari Order Orchestrator | ✅ Done |
| Foreign surcharge | Header `User-Info.is_foreign` + order threshold | ✅ Done |
| Corporate surcharge | Request body `payment_method.type` | ✅ Done |
| Loyalty tier discount | Header `User-Info.loyalty_tier` | ✅ Done |
| BIN country surcharge | Request body `payment_method.country_code` | ✅ Done |
| Card principal (AMEX) | Request body `payment_method.principal` | ✅ Done |
| Urgency type | Request body `urgency_type` | ✅ Done |
| Rounding | Round up to nearest 1000 | ✅ Done |

### Phase 2 — External Integration (Future)

| Modifier | Source | Status |
|---|---|---|
| Pickup zone (APSH, Event Area) | Polygon Service | ⏳ Pending |
| Surge condition (Extreme) | Demand Service | ⏳ Pending |

---

## Corporate Payment Types

| Payment Type | Category |
|---|---|
| `ECV` | Corporate |
| `trip_voucher` | Corporate |
| `cc_cp` | Corporate |
| `cc` | Retail |
| `debit` | Retail |
| `cash` | Retail |
| `ewallet` | Retail |

---

## Open Questions (Resolved)

| Question | Answer | Date |
|---|---|---|
| Rounding kelipatan berapa? | 1000 | 2026-03-02 |
| New user threshold? | `<= 1` order | 2026-03-02 |
| Foreign fee threshold? | `>= 6` order | 2026-03-02 |
| Double surcharge allowed? | Ya (foreign + AMEX) | 2026-03-02 |
| Service type fallback? | Return error 503 | 2026-03-02 |
| Loyalty tier source? | Header User-Info | 2026-03-05 |
| Query Loyalty Service? | Tidak, dari header | 2026-03-05 |

---

## References

- [[RFC-001-dynamic-platform-fee-service]]
- [[RFC-002-order-orchestrator-counter]]
- [[RFC-003-user-service-foreign-flag]]
- [[rule-engine-implementation]]
