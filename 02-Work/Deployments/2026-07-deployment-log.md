# 📅 July 2026 - Deployment Log

**Month:** July 2026
**Total Hours:** 7.0 hours + TBD
**Total Deployments:** 3

---

## Summary by Team

| Team | Deployments | Hours | Success | Incident |
|------|------------|-------|---------|----------|
| UPG | 2 | 3.0 + TBD | 1 | 0 |
| MRG | 1 | 4.0 | 1 | 0 |
| **TOTAL** | **3** | **7.0 + TBD** | **2** | **0** |

---

## Deployment Details

### 2026-07-01: UPG Payment Release — [2026.06.05] 🕐 SCHEDULED

- **Teams:** UPG
- **Feature:** Nicepay inquiry-fail fix + timezone scheduler fix + tipping CC fallback to RECHARGE + routing-service & ecv-service updates
- **Type:** New Release
- **Tech Lead:** Eko Nugroho
- **PM:** Rayan Nurbadi
- **Scheduled Time:** 21:30 (2026-07-01)
- **Target Environment:** Production
- **Participating:** ✅ Yes
- **Outcome:** 🕐 SCHEDULED

#### Impact Analysis

| Service | Change | Risk | Catatan |
|---------|--------|------|---------|
| nicepay-service | Inquiry fail saat deadline hampir habis + fix timezone scheduler + set failed saat timeout | Rendah | Prevent order stuck pending. Scheduler query sebelumnya salah timezone → miss order expiry |
| mrg-gateway | Tipping CC: fallback ke RECHARGE data kalau CHARGE detail failed | Rendah | Fix bug existing — sebelumnya tipping gagal padahal recharge sudah sukses. Hanya affect CC preauth flow dengan capture gagal + recharge berhasil |
| routing-service | (belum didokumentasikan di Impact Analysis — gap) | — | — |
| ecv-service | (belum didokumentasikan di Impact Analysis — gap) | — | — |

#### Team Members
- Eko Nugroho (Tech Lead)
- Rayan Nurbadi (PM)
- Sahrul Ramdoni (QA)
- Abdul Karman (QA)
- Adam Haniif (DevOps)

#### Pre-Deployment Status

| Item | Status |
|------|--------|
| QA staging test | ⏳ Belum dikonfirmasi |
| PM stakeholder notif | ❌ Belum |

#### Deployment Steps

| No | Service | MR | Area | PIC | Status |
|----|---------|-----|------|-----|--------|
| 1 | mrg-gateway | [MR #1231](https://git.bluebird.id/upg/mrg-gateway/-/merge_requests/1231) | mrg-gateway | Eko Nugroho | 🕐 |
| 2 | nicepay-service | [MR #57](https://git.bluebird.id/upg/nicepay-service/-/merge_requests/57) | nicepay-service | Eko Nugroho | 🕐 |
| 3 | ecv-service | [MR #737](https://git.bluebird.id/argocd/upg/-/merge_requests/737) | ecv-service | Eko Nugroho | 🕐 |
| 4 | routing-service | [MR #162](https://git.bluebird.id/upg/routing-service/-/merge_requests/162) | routing-service | Eko Nugroho | 🕐 |
| 5 | PVT (Test After Live) | [UAT & Deploy.xlsx](https://bluebirdgroup365.sharepoint.com/:x:/s/UPG/ETTPqKbH4RRLuUO6CrKYTL0BB3gtjBwR6FCfymQtRtIcWA?e=zi9h8Y) | — | Abdul Karman | 🕐 |

#### Code Quality

| Service | Coverage | Duplication | Quality Gate |
|---------|----------|-------------|---------------|
| mrg-gateway | 28.0% | 6.2% | BB Level 0 |
| nicepay-service | 87.4% | 1.7% | BB Level Started |
| routing-service | 58.2% | 0.0% | BB Level Started |

#### Rollback Plan

| No | Actions | Detail | PIC | Status |
|----|---------|--------|-----|--------|
| 1 | Rollback Image | [mrg-gateway v1.22.3](https://git.bluebird.id/upg/mrg-gateway/-/tags/v1.22.3) · [nicepay-service v1.0.6](https://git.bluebird.id/upg/nicepay-service/-/tags/v1.0.6) · [routing-service v1.5.4](https://git.bluebird.id/upg/routing-service/-/tags/v1.5.4) | Adam Haniif (DevOps) | 🕐 |
| 2 | Monitoring All services (KMS, GMS) | — | All member | 🕐 |

#### Known Gaps (dari analisa pre-deployment)
- Impact Analysis belum mencakup `routing-service` dan `ecv-service`.
- Status di QA Notes, Deployment Steps, dan Rollback Plan masih kosong (belum dieksekusi).
- PM checklist notifikasi stakeholder belum dicentang.
- Operational Impact & Business Impact masih kosong di ClickUp doc.
- Risk level "Rendah" untuk timezone scheduler fix belum direview ulang.

---

### 2026-07-08: UPG Payment Release — [2026.07.01] Pre-Auth by User ID Level ✅ SUCCESS

- **Teams:** UPG
- **Feature:** FDS Pre-Authorization evaluasi risiko level user (BB ID) — user preauth override behind feature flag `FF_USER_PREAUTH` (guard AMEX/JCB) + trx type baru `gb_extra_overtime_daily` (remove scheduler retry & outstanding list untuk extra/OT GB Rent daily fail charge)
- **Type:** New Release
- **Tech Lead:** Eko Nugroho
- **PM:** Rayan Nurbadi
- **Deployment Time:** 22:00 (2026-07-08) – 01:00 (2026-07-09)
- **Target Environment:** Production
- **Participating:** ✅ Yes
- **Outcome:** ✅ SUCCESS
- **Related Tasks:** UPGN-3926, UPGN-3927, UPGN-3928, UPGN-3929, UPGN-3965

#### Impact Analysis

| Service | Change | Risk | Catatan |
|---------|--------|------|---------|
| routing-service | User preauth override sepenuhnya behind feature flag `FF_USER_PREAUTH` dengan guard AMEX/JCB. Read query pakai DB replica, fallback ke primary | Rendah | Tanpa flag ON, behavior 100% sama seperti sebelumnya |
| card-payment | Pass UserID ke routing backward compatible (tambah field saja). Save PaymentIdentifier non-fatal (UPGN-3907). Fix double charge — unblock CC gagal sekarang return success, bukan error (UPGN-3829) | Rendah | Menghilangkan customer retry akibat double charge |
| meta-payment-gateway (mpg2) | 1 commit cherry-pick: pass userID ke getCCRouteInfo agar routing-service bisa lookup user preauth override | Rendah | Perubahan minimal, backward compatible |
| mrg-gateway | Trx type baru `gb_extra_overtime_daily` untuk DirectPaymentV1.1 — flow charge normal tanpa retry & tanpa masuk outstanding, gagal langsung return error (UPGN-3965) | Rendah | Tidak mengubah flow existing `gb_extra_overtime` |

#### Team Members
- Eko Nugroho (Tech Lead)
- Rayan Nurbadi (PM)
- Sahrul Ramdoni (QA)
- Abdul Karman (QA)
- Adam Haniif (DevOps)

#### Deployment Steps

| No | Service | MR | Area | PIC | Status |
|----|---------|-----|------|-----|--------|
| 1 | card-payment | [MR #1069](https://git.bluebird.id/upg/card-payment/-/merge_requests/1069) | card-payment | Eko Nugroho | ✅ |
| 2 | routing-service | [MR #164](https://git.bluebird.id/upg/routing-service/-/merge_requests/164) | routing-service | Eko Nugroho | ✅ |
| 3 | meta-payment-gateway | [MR #2124](https://git.bluebird.id/mybb-backend/meta_payment_gateway_v2_go/-/merge_requests/2124) | MPG2 | Eko Nugroho | ✅ |
| 4 | mrg-gateway | [MR #1235](https://git.bluebird.id/upg/mrg-gateway/-/merge_requests/1235) | mrg-gateway | Eko Nugroho | ✅ |
| 5 | PVT (Test After Live) | [UAT & Deploy.xlsx](https://bluebirdgroup365.sharepoint.com/:x:/s/UPG/ETTPqKbH4RRLuUO6CrKYTL0BB3gtjBwR6FCfymQtRtIcWA?e=zi9h8Y) | — | Abdul Karman | 🕐 |

#### Code Quality

| Service | Coverage | Duplication | Quality Gate |
|---------|----------|-------------|---------------|
| routing-service | 72.9% | 3.6% | BB Level 2 |
| card-payment | 79.5% | 3.9% | BB Level 2 |
| mpg2 | 58.2% | 0.0% | BB Level 0 |
| mrg-gateway | 28.0% | 6.2% | BB Level 0 |

#### Rollback Plan

| No | Actions | Detail | PIC | Status |
|----|---------|--------|-----|--------|
| 1 | Rollback Image | [routing-service v1.5.6](https://git.bluebird.id/upg/routing-service/-/tags/v1.5.6) · [card-payment v1.18.7](https://git.bluebird.id/upg/card-payment/-/tags/v1.18.7) · [meta-payment-gateway v5.35.7](https://git.bluebird.id/mybb-backend/meta_payment_gateway_v2_go/-/tags/v5.35.7) · [mrg-gateway v1.22.4](https://git.bluebird.id/upg/mrg-gateway/-/tags/v1.22.4) | DevOps | Tidak diperlukan (deploy success) |
| 2 | Monitoring All services (KMS, GMS) | — | All member | ✅ |

---

### 2026-07-14: MyBB Release — GB AT P2P Fixes ✅ SUCCESS

- **Teams:** MRG
- **Feature:** GB AT P2P — handle final fare on order detail & final fare charging; handle schedule GB exclude from reschedule; handle missing airport transfer service type on order detail
- **Type:** New Release
- **Approvers:** Noverino, Ridlo
- **Deployment Time:** 22:00 (2026-07-14) – 02:00 (2026-07-15)
- **Target Environment:** Production
- **Participating:** ✅ Yes
- **Outcome:** ✅ SUCCESS
- **ClickUp:** [Task 80180058874768](https://bluebirdgroup.clickup.com/9018711461/v/cn/7-9018711461-8/t/80180058874768)

---

## Monthly Statistics

| Metric | Value |
|--------|-------|
| **Working Days** | TBD |
| **Deployments Worked** | 3 |
| **Total Overtime Hours** | 7.0 + TBD |
| **Night Deployments (>17:00)** | 3 |
| **Weekend Deployments** | 0 |
| **Success Rate** | TBD |
| **Incidents** | 0 |
| **Rollbacks** | 0 |

---

## 🔗 Related
- [[deployment-overtime-summary]] (Master Dashboard)
- [[monthly-deployment-overtime-2026]] (2026 Monthly Summary)
- [ClickUp Plan 2026.06.05](https://bluebirdgroup.clickup.com/9018711461/v/dc/8crx7d5-276098/8crx7d5-210238)
- [ClickUp Plan 2026.07.01 - Pre-Auth by User ID Level](https://bluebirdgroup.clickup.com/9018711461/v/dc/8crx7d5-276098/8crx7d5-212718)
