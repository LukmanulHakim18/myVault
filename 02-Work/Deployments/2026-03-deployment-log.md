# 📅 March 2026 - Deployment Log

**Month:** March 2026  
**Total Hours:** 5 hours  
**Total Deployments:** 2  

---

## Summary by Team

| Team | Deployments | Hours | Success | Incident |
|------|------------|-------|---------|----------|
| MRG | 2 | ~9 hours | 0 | 2 |
| UPG | 1 | ~5 hours | 0 | 1 |
| BBD | 1 | ~5 hours | 0 | 1 |
| **TOTAL** | **2 events** | **~5 hours (personal)** | **0** | **1** |

---

## Deployment Details

### 2026-03-05: MyBB Legacy (MRG 6.23.7) ⏳ PLANNED
- **Team:** MRG
- **Feature:** Enhancement / Improvement
- **Type:** Enhancement
- **Scope:** TBD (Pre-deployment checklist pending)
- **Tech Lead:** Alfian Maulana Malik, Isfan
- **PM:** Aby Dharmala, Nurul Aisha Fitriyah, Fionna Benita
- **PIC Dev:** Kamal Firdaus
- **QA:** Nadila Putri Nurlitasari
- **Participating:** ✅ Yes
- **Scheduled Start:** 22:00
- **Scheduled End:** 03:59 (+1 day, 2026-03-06)
- **Estimated Duration:** ~6 hours
- **Outcome:** ⏳ PENDING
- **Notes:** Improvement release for MRG 6.23.7

---

### 2026-03-12: FDS Feature Deployment ⚠️ ROLLBACK
- **Teams:** MyBB (MRG) + BBD + UPG (joint deployment)
- **Feature:** FDS (Fraud Detection System)
- **Type:** Feature rollout — cross-team, 3 tim bersamaan
- **Participating:** ✅ Yes
- **Start Time:** 22:00 (2026-03-12)
- **End Time:** 03:00 (2026-03-13)
- **Duration:** 5 hours
- **Outcome:** ⚠️ ROLLBACK (semua tim)

#### Kronologi Incident

| # | Waktu | Event |
|---|-------|-------|
| 1 | ~22:xx | BBD deploy terlebih dahulu |
| 2 | ~22:xx | UPG deploy |
| 3 | ~22:xx | MRG deploy |
| 4 | ~22:xx | BBD pindah config v1 → v2 |
| 5 | ~22:xx | Massive error `GetWalletBalance` muncul di log |
| 6 | ~22:xx | Investigasi: v2 BBD tidak hit ke service MRG baru sama sekali |
| 7 | ~xx:xx | MyBB (MRG) rollback, BBD tetap di v2 |
| 8 | ~xx:xx | Isu CC berpindah ke Cash saat on-trip muncul |
| 9 | ~xx:xx | BBD pindah config v2 → v1 (mekanisme ewallet-to-cash lama, pre-FDS) |
| 10 | ~xx:xx | Isu CC → Cash **masih terjadi** meskipun BBD sudah di v1 |
| 11 | ~xx:xx | BBD rollback (full rollback binary) |
| 12 | ~03:00 | Isu CC → Cash **tidak terjadi lagi** setelah BBD rollback |

#### Root Cause Analysis (Preliminary)

**Issue 1: GetWalletBalance massive error**
- BBD v2 tidak terkonfigurasi dengan benar untuk memanggil service MRG versi baru
- Config switch v1→v2 di BBD tidak disertai dengan routing update ke endpoint MRG baru

**Issue 2: CC pindah ke Cash saat on-trip**
- Terjadi saat MyBB sudah rollback, BBD masih di v2
- Masih terjadi saat BBD pindah ke v1 (config switch saja, tanpa rollback binary)
- **Berhenti setelah BBD full rollback** → mengindikasikan bug ada di binary BBD, bukan hanya di config
- Config v1 ≠ state yang sama dengan binary BBD sebelum FDS

#### Impact
- Passenger on-trip mengalami payment method switching (CC → Cash) tanpa consent
- Potensi fraud exposure selama window incident

#### Action Items (Follow-up)
- [ ] Postmortem formal dengan semua tim (MRG, BBD, UPG)
- [ ] Investigate: kenapa BBD v2 tidak hit endpoint MRG baru
- [ ] Investigate: bug CC→Cash di binary BBD (apakah di logic FDS atau payment fallback)
- [ ] Review config management BBD: perbedaan config switch vs binary rollback
- [ ] Koordinasi deployment sequence yang lebih aman untuk joint deployment berikutnya

#### Lessons Learned (Preliminary)
- Config switch (v1↔v2) tidak cukup untuk rollback jika bug ada di level binary
- Joint deployment 3 tim memerlukan smoke test per tim sebelum config switch
- Perlu agreed rollback procedure yang terdefinisi sebelum deployment dimulai

---

## Monthly Statistics

| Metric | Value |
|--------|-------|
| **Working Days** | ~23 |
| **Deployments Worked** | 2 |
| **Total Overtime Hours** | ~5 hours |
| **Night Deployments (>17:00)** | 2 |
| **Weekend Deployments** | 1 (2026-03-12, Kamis malam) |
| **Success Rate** | 0% |
| **Incidents** | 1 |
| **Rollbacks** | 1 (full, semua tim) |

---

## 🔗 Related
- [[deployment-overtime-summary]] (Master Dashboard)
- [[monthly-deployment-overtime-2026]] (2026 Monthly Summary)
