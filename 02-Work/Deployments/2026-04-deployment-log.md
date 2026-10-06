# 📅 April 2026 - Deployment Log

**Month:** April 2026  
**Total Hours:** 7.0 hours  
**Total Deployments:** 2  

---

## Summary by Team

| Team | Deployments | Hours | Success | Incident |
|------|------------|-------|---------|----------|
| MRG | 1 | ~7 hours | 1 | 0 |
| UPG | 2 | ~7 hours + TBD | 1 | 0 |
| BBD | 1 | ~7 hours | 1 | 0 |
| **TOTAL** | **2 events** | **~7 hours (personal) + TBD** | **1** | **0** |

---

## Deployment Details

### 2026-04-04: FDS Pre-auth — Deployment Simulation ✅ SUCCESS (SIMULATION)
- **Teams:** MRG + UPG + BBD (joint deployment, 3 squad)
- **Feature:** FDS Pre-auth (re-attempt setelah rollback Maret 2026)
- **Type:** Deployment Simulation — dry run sebelum production deployment
- **Commander:** Lukmanul Hakim
- **PIC per Squad:**
  - UPG: Eko (deploy), Nuur Rohman (indexing & whitelist CC)
  - MyBB/MRG: Alfian M
  - BBD: Aldhonie
  - QA: Abdul, Nadila, Dian, Eka, Bagus
- **Participating:** ✅ Yes
- **Start Time:** 20:00 (2026-04-04)
- **End Time:** 03:00 (2026-04-05)
- **Duration:** 7 hours
- **Outcome:** ✅ SIMULATION SUCCESS (dengan catatan)


#### Runbook Execution Summary

| Step | Deskripsi | PIC | Status |
|------|-----------|-----|--------|
| Pre-deploy | Konfirmasi PIC standby & pre-requisite | Lukmanul | ✅ Done |
| Step 1 | QA Test Preparation | Abdul, Nadila, Dian | ✅ Done |
| Step 2 | UPG: Indexing `charge_tracking` | Nuur Rohman | ✅ Done |
| Step 3 | UPG Deploy (card-payment, mrg-gateway, service-transaction) | Eko | ✅ Done |
| Step 4 | MyBB Deploy — tagging v6.24, whitelist BBID | Alfian M | ✅ Done |
| Step 5 | QA UPG — CC non-whitelist | Abdul | ✅ Pass |
| Step 6 | QA BBD Pre-switch — create order e-wallet | Bagus | ⏭️ **SKIPPED** |
| Step 7 | BBD Deploy — payment-watch, flagging V2, exclude GB | Aldhonie | ✅ Done |
| Step 8 | QA BBD Post-switch — end trip after switch | Abdul, Nadila, Eka | ⏭️ **SKIPPED** |
| Step 9 | BBD: Whitelist Payment CC | Nuur Rohman | ✅ Done |
| Step 10 | QA Full Flow (MyBB + BBD + UPG) | Abdul, Nadila, Dian, Eka | ✅ Pass* |
| Step 11 | Post-deploy monitoring (Kibana/Grafana) | Eko, Alfian M, Aldhonie | ✅ Done |

> *Full flow QA: semua checklist internal pass. Konfirmasi formal ke commander pending di catatan.

#### Catatan Khusus

- **Step 6 & 8 diskip** karena IOT tidak berfungsi saat simulasi — order e-wallet untuk end trip tidak bisa dibuat
- Simulasi ini adalah re-attempt dari deployment FDS yang rollback pada 2026-03-12
- Dependency order: UPG → MRG → BBD (enforced)
- Pre-requisite kritis: **Service MyBB bisa dihit dari BBD** — terverifikasi ✅

#### Follow-up Before Production Deployment
- [ ] Koordinasi dengan tim IOT untuk memastikan environment berfungsi saat deployment actual
- [ ] Konfirmasi ulang Step 6 & 8 bisa dieksekusi di production
- [ ] Review whitelist BBID sebelum go-live

---

### 2026-04-14: UPG / Payment — New Release 🕐 SCHEDULED
- **Teams:** UPG
- **Feature:** UPG Payment Release (3 changes)
- **Type:** New Release
- **Tech Lead:** Eko Nugroho
- **PM:** Rayan Nurbadi
- **Scheduled Time:** 21:30 (2026-04-14)
- **Target Environment:** Production
- **Participating:** ✅ Yes
- **Outcome:** 🕐 PENDING

#### Changes
1. `(provider-lib)` Add parameter `initiated_by` di endpoint `v2/charge`
2. Remove FF Get Voucher From CP & FF Push HO GB Now Ewallet
3. Pointing MPG1 dari VM ke Kube

#### Team Members
- Eko Nugroho (Tech Lead)
- Rayan Nurbadi (PM)
- Muhammad Marwan Faisal (BE)
- Sahrul Ramdoni (QA)
- Abdul Karman (QA)
- Adam Haniif (DevOps)

#### Risk / Impact Analysis
1. Midtrans dengan payment CC Error ketika charging
2. Impact tidak bisa di-switch ke code lama
3. Payment Cash datanya tidak tersimpan

#### Deployment Steps

| No | Service | Steps | PIC | Status |
|----|---------|-------|-----|--------|
| 1 | mrg-gateway-service | Create tag → CDI → build | Eko Nugroho | 🕐 |
| 2 | MPG2 | Create tag → CDI → build | Eko Nugroho | 🕐 |
| 3 | PVT (Test After Live) | Skenario [UPG] - UAT & Deploy.xlsx | Abdul Karman | 🕐 |

#### Pre-requisites
- [ ] Restart intermediate HO, pastikan topic `push.create.ho.order` terbuat di RabbitMQ dashboard
- [ ] Change MPG1 IP: `172.21.56.134:3000` (VM) → `upg-mpg1.internal.bluebird.id` (Kube)

#### Rollback Plan

| No | Actions | PIC | Status |
|----|---------|-----|--------|
| 1 | Rollback Image | DevOps (Adam Haniif) | 🕐 |
| 2 | Monitoring All services (KMS, GMS) | All member | 🕐 |

#### Quality Gate
| Service | Coverage | Duplications | Gate |
|---------|----------|-------------|------|
| card-payment | 27.1% | 6.2% | BB Level 0 |
| MPG2 | 80.2% | 4.0% | BB Level 2 |

---

## Monthly Statistics

| Metric | Value |
|--------|-------|
| **Working Days** | ~26 |
| **Deployments Worked** | 2 (1 simulation + 1 scheduled) |
| **Total Overtime Hours** | 7 hours (simulation) + TBD (14 Apr) |
| **Night Deployments (>17:00)** | 2 |
| **Weekend Deployments** | 1 (2026-04-04, Sabtu malam) |
| **Success Rate** | TBD |
| **Incidents** | 0 |
| **Rollbacks** | 0 |

---

## 🔗 Related
- [[deployment-overtime-summary]] (Master Dashboard)
- [[monthly-deployment-overtime-2026]] (2026 Monthly Summary)
- [[2026-04-04-deployment-simulation-fds-preauth]] (Meeting Notes)
