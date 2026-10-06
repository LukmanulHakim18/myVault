# 📅 May 2026 - Deployment Log

**Month:** May 2026  
**Total Hours:** TBD  
**Total Deployments:** 2  

---

## Summary by Team

| Team | Deployments | Hours | Success | Incident |
|------|------------|-------|---------|----------|
| UPG | 2 | TBD | TBD | TBD |
| **TOTAL** | **2** | **TBD** | **TBD** | **TBD** |

---

## Deployment Details

### 2026-05-25: UPG Payment Release — [2026.05.04] 🕐 SCHEDULED

- **Teams:** UPG
- **Feature:** GB Rent Revamp — Recharge komponen extra/overtime yang gagal charge
- **Type:** New Release
- **Tech Lead:** Eko Nugroho
- **PM:** Rayan Nurbadi
- **Scheduled Time:** 21:30 (2026-05-25)
- **Target Environment:** Production
- **Participating:** ✅ Yes
- **Outcome:** 🕐 SCHEDULED

#### Business Impact
- Support fitur GB Rent Revamp: ketika komponen extra/overtime gagal charge, dapat dilakukan recharge

#### Pre-Deployment Status
| Item | Status |
|------|--------|
| Rollback tag (v1.21.7) | ✅ Siap |
| QA staging test | ⏳ Sedang test (konfirmasi nanti) |
| PM stakeholder notif | ❌ Belum |

#### Team Members
- Eko Nugroho (Tech Lead)
- Rayan Nurbadi (PM)
- Erwin Primadani (BE)
- Muhammad Marwan Faisal (BE)
- Sahrul Ramdoni (QA)
- Abdul Karman (QA)
- Adam Haniif (DevOps)

#### Deployment Steps

| No | Service | MR | Area | PIC | Status |
|----|---------|-----|------|-----|--------|
| 1 | mrg-gateway | [MR #1209](https://git.bluebird.id/upg/mrg-gateway/-/merge_requests/1209) | mrg-gateway | Eko Nugroho | 🕐 |
| 2 | PVT (Test After Live) | [UAT & Deploy.xlsx](https://bluebirdgroup365.sharepoint.com/:x:/s/UPG/ETTPqKbH4RRLuUO6CrKYTL0BB3gtjBwR6FCfymQtRtIcWA?e=zi9h8Y) | — | Abdul Karman | 🕐 |

#### Code Quality

| Service | Coverage | Duplication | Quality Gate |
|---------|----------|-------------|--------------|
| mrg-gateway | 27.9% | 6.3% | BB Level 0 |

#### Rollback Plan

| No | Actions | Detail | PIC | Status |
|----|---------|--------|-----|--------|
| 1 | Rollback Image | [v1.21.7](https://git.bluebird.id/upg/mrg-gateway/-/tags/v1.21.7) | Adam Haniif (DevOps) | 🕐 |
| 2 | Monitoring All services (KMS, GMS) | — | All member | 🕐 |

---

### 2026-05-20: UPG Payment Release — [2026.05.03] 🕐 SCHEDULED
- **Teams:** UPG
- **Feature:** Business Metric Logging + GoPay/ShopeePay Cross Payment Fix
- **Type:** New Release
- **Tech Lead:** Eko Nugroho
- **PM:** Rayan Nurbadi
- **Scheduled Time:** 22:00 (2026-05-20)
- **Target Environment:** Production
- **Participating:** ✅ Yes
- **Outcome:** 🕐 SCHEDULED

#### Business Impact
1. Penambahan log business metric → monitoring real-time status & performa business flow UPG
2. Perbaikan payment gateway GoPay & ShopeePay di fitur cross payment → clearing transaksi dari provider

#### Team Members
- Eko Nugroho (Tech Lead)
- Rayan Nurbadi (PM)
- Erwin Primadani (BE)
- Muhammad Marwan Faisal (BE)
- Sahrul Ramdoni (QA)
- Abdul Karman (QA)
- Adam Haniif (DevOps)

#### Deployment Steps

| No | Service | MR | Area | PIC | Status |
|----|---------|-----|------|-----|--------|
| 1 | meta-payment-gateway | [MR #2093](https://git.bluebird.id/mybb-backend/meta_payment_gateway_v2_go/-/merge_requests/2093) | mpg-2 | Eko Nugroho | 🕐 |
| 2 | webhook-fwd | [MR #212](https://git.bluebird.id/upg/webhook-fwd/-/merge_requests/212) | webhook-fwd | Eko Nugroho | 🕐 |
| 3 | intermediate-ho | [MR #809](https://git.bluebird.id/upg/intermediate-ho/-/merge_requests/809) | intermediate-ho | Eko Nugroho | 🕐 |
| 4 | card-payment | [MR #1041](https://git.bluebird.id/upg/card-payment/-/merge_requests/1041) | card-payment | Eko Nugroho | 🕐 |
| 5 | routing-service | [MR #153](https://git.bluebird.id/upg/routing-service/-/merge_requests/153) | routing-service | Eko Nugroho | 🕐 |
| 6 | PVT (Test After Live) | [UAT & Deploy.xlsx](https://bluebirdgroup365.sharepoint.com/:x:/s/UPG/ETTPqKbH4RRLuUO6CrKYTL0BB3gtjBwR6FCfymQtRtIcWA?e=zi9h8Y) | — | Abdul Karman | 🕐 |

#### Code Quality

| Service | Coverage | Duplication | Quality Gate | Notes |
|---------|----------|-------------|--------------|-------|
| webhook-fwd | 86.4% | 2.0% | BB Level 2 | ✅ |
| intermediate-ho | 84.5% | 6.6% | BB Level 3 | ✅ |
| card-payment | 79.5% | 3.9% | BB Level Started | ⚠️ |
| routing-service | 69.7% | 2.3% | BB Level 2 | ✅ |
| meta-payment-gateway | 23.9% | 7.0% | BB Level 0 | ✅ Accepted — Legacy system |
| mrg-gateway | 27.9% | 6.2% | BB Level 0 | ✅ Accepted — Legacy system |

#### Rollback Plan

| No | Actions | PIC | Status |
|----|---------|-----|--------|
| 1 | Rollback Image | DevOps (Adam Haniif) | 🕐 |
| 2 | Monitoring All services (KMS, GMS) | All member | 🕐 |

---

## Monthly Statistics

| Metric | Value |
|--------|-------|
| **Working Days** | ~22 |
| **Deployments Worked** | 1 |
| **Total Overtime Hours** | TBD |
| **Night Deployments (>17:00)** | 1 |
| **Weekend Deployments** | 0 |
| **Success Rate** | TBD |
| **Incidents** | TBD |
| **Rollbacks** | TBD |

---

## 🔗 Related
- [[deployment-overtime-summary]] (Master Dashboard)
- [[monthly-deployment-overtime-2026]] (2026 Monthly Summary)
- [ClickUp Plan 05.04 — Malam Ini](https://bluebirdgroup.clickup.com/9018711461/v/dc/8crx7d5-276098/8crx7d5-198818)
- [ClickUp Plan 05.03](https://bluebirdgroup.clickup.com/9018711461/v/dc/8crx7d5-276098/8crx7d5-196798)
