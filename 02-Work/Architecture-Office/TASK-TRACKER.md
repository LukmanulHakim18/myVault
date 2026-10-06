---
title: Active Tasks Tracker - Lukmanul Hakim
type: task-tracking
date: '2026-01-27'
status: active
updated: '2026-07-09'
---

# 📊 Task Tracker - Ongoing Work

**Last Updated**: 2026-07-09  
**Owner**: Lukmanul Hakim  

---

## 🔴 **HIGH PRIORITY - CRITICAL PATH**

### 1. **Disbursement Gateway (UPG/OCBC SNAP)**
**Status**: 🟢 IMPLEMENTASI  
**Owner**: Lukmanul (design) + UPG (Eko Nugroho, coding)  
**Docs**: `02-Work/Teams/UPG/01-architecture/design-docs/disbursement/`  
**Repo**: `disbursement-service` (`/mnt/d/code/go/upg/disbursement-service`)

- Requirements + design selesai (2026-07-03); semua open question (OQ-1..OQ-6, CONF-1..3) sudah resolved (2026-07-08)
- Coding jalan: scaffold+model+DB, entitlement (4 tabel), Admin API (8 RPC), Disburse usecase (12 subtest GREEN) — commit terakhir 2026-07-09
- Belum dikerjakan: batch disbursement, worker poller, adapter OLT/RTGS/SKN (nunggu bank-metadata)
- Next: RFC-UPG-00X sign-off

---

### 2. **QRIS Taximeter (UPG)**
**Status**: 🟡 VENDOR KICKOFF  
**Owner**: UPG  
**Docs**: `02-Work/Teams/UPG/01-architecture/design-docs/qris-taximeter/`

- Vendor DECIDED (2026-07-03): BCA + BNI dual-acquirer, handle seluruh proses QRIS Statis
- BCA implementation kick-off 2026-07-03; BNI kick-off reschedule
- Open: OQ-1 multi-acquirer routing arch, OQ-2 tips separation, OQ-4 field mapping MDR, OQ-5 service ownership (qris-service baru?)

---

### 3. **New Preauth FDS — Pre-Auth by User ID Level (CHASH → BB ID)**
**Status**: 🟢 PHASE 1 DEPLOYED (2026-07-08)  
**Owner**: MRG + UPG (Eko Nugroho, Tech Lead)  
**Docs**: `/mnt/d/project-bb/new-preauth-fds/` (architecture-brief.md, contract/fds.proto, laporan-vp-slide.md)

- Goal: evaluasi risiko pre-auth per **user (BB ID)** bukan per **kartu (CHASH)** — kurangi friksi user genuine yang pakai kartu baru/dorman
- Arsitektur 3 service: `card-payment` (UPG, first preauth), `n2cservice` (MRG, continuous preauth during trip), **FDS baru** (simpan & expose config risiko, dihitung Tim Data)
- Dilaporkan ke VP 2026-07-03
- **Deployed 2026-07-08** (release `[2026.07.01] Pre-Auth by User ID Level`, ✅ SUCCESS): user preauth override behind feature flag `FF_USER_PREAUTH` (guard AMEX/JCB) + trx type baru `gb_extra_overtime_daily`. Related tasks: UPGN-3926/3927/3928/3929/3965
- Next: rollout feature flag lebih luas, validasi 5 kategori klasifikasi risiko user

---

### 4. **CC Charging Flow (N2C) — Direct Charge & Force-to-Cash**
**Status**: 🟡 DRAFT (3 dari 4 doc)  
**Owner**: MRG + UPG Payment + BBD Dispatching  
**Docs**: `02-Work/Teams/MRG/01-architecture/design-docs/cc-charging-flow/`

- Goal: hapus 30 menit waiting setelah dropoff (direct charge saat Complete), pisah tips/extra dari fare utama
- N2C base flow (pre-auth/cancel/switch-to-cash): **Approved**
- Spike new charging flow, sequence diagrams, BBD force-to-cash: masih **Draft** sejak Feb–Mar 2026
- Related: deployment simulasi FDS Pre-auth (2026-04-04), bugfix tipping-CC-preauth (deployment log Juli 2026). Beririsan dengan item #3 di atas (sama-sama pre-auth flow) tapi scope berbeda: #3 = siapa yang di-preauth, #4 = kapan/bagaimana charge dieksekusi.

---

### 5. **ECV for GB Rent Revamp - Push HO Architecture**
**Target Release**: v6.24 (W4 Feb - Regression Start)  
**Status**: 🔴 NOT STARTED  
**Owner**: Team Discussion (MRG + UPG + CP)

#### Background
- GB charging di depan (beda dari BB/SB yang belakang)
- Rent Revamp v6.24 hanya support ECV + cc_cp untuk hourly
- Perlu definisikan push HO flow untuk payment, refund, tipping

#### Key Questions to Resolve
1. **Flow Payment ECV**: Apakah mengikuti Cititrans pattern?
2. **Budget Calculation**: Validasi untuk cross-month orders (Jan order untuk Feb)
3. **Pelaporan HO**: MRG yang handle (decision sudah confirmed)

#### Decision Needed
- [ ] **Architecture Decision**: Single vs Multiple posting untuk HO
  - Option A: Single posting (simpler)
  - Option B: Multiple posting dengan journal routing (overtime, extra, tipping, cancel)
  
#### Subtasks
- [ ] Define journal routing rules (jika pilih Option B)
- [ ] Design topic schema untuk general rent topic
- [ ] Error handling & retry mechanism untuk push HO failure
- [ ] Push CP failure handling & recovery
- [ ] Testing scenarios:
  - [ ] Tipping scenarios
  - [ ] Refund/void scenarios
  - [ ] Cross-month order scenarios
  - [ ] Budget calculation validation
- [ ] Monitoring & alerting untuk multi-charging flow

#### Implementation Owner
- **MRG**: Handle tipping case + push HO dengan parameter `is_push_ho`
- **UPG**: Optional push HO (based on `is_push_ho = true`)
- **CP**: Direct payment push, refund/void handling, tipping update

---

### 6. **Traffic Spike Analysis - Order Gantung Incident**
**Incident Date**: 2025-01-12 (77k concurrent users)  
**Status**: 🟡 INVESTIGATING  
**Owner**: Lukmanul Hakim  
**Severity**: HIGH

#### Investigation Phases

##### Phase 1: Data Collection (URGENT)
- [ ] Collect metrics dari Huawei Cloud (12 Jan full day)
- [ ] Export logs: MRG, UPG, Order Orchestrator
- [ ] Query database untuk list order gantung
- [ ] Collect payment transaction records
- [ ] Check HO records untuk missing payments
- [ ] Identify affected order count & financial impact

##### Phase 2: Analysis (HIGH)
- [ ] Analyze logs untuk error patterns
- [ ] Correlate order data across services
- [ ] Map timeline: spike → errors → degradation
- [ ] Identify root cause dari hypotheses

##### Phase 3-5: Root Cause, Remediation, Prevention
- [ ] Short-term fix untuk recover order gantung
- [ ] Payment reconciliation process
- [ ] Load testing dengan 100k+ users
- [ ] Auto-scaling configuration review

#### Success Criteria
- [x] Root cause identified
- [ ] All order gantung recovered
- [ ] Payment reconciliation completed
- [ ] Preventive measures implemented
- [ ] Load test successful (100k+ users)

---

### 7. **VP Initiative - Development Guidelines**
**Status**: 🟡 DRAFT  
**Priority**: MEDIUM-HIGH  
**Owner**: Lukmanul (with VP)

#### Deliverables
- [ ] Development guideline document
- [ ] Database access best practices guide
- [ ] Slow query detection & debugging guide
- [ ] Code examples & templates
- [ ] Monitoring & alerting setup guide

---

## 🟡 **MEDIUM PRIORITY - ONGOING**

### 8. **Technical Documentation Updates**
**Status**: 🟡 ONGOING  
**Owner**: Lukmanul

#### Pending Documentation
- [ ] GB Rent Revamp - Technical design docs
- [ ] ECV Payment Flow - Detailed diagrams & docs

---

### 9. **System-Wide Risk Assessment (MRG+UPG+BBD)**
**Status**: 🔴 TODO  
**Owner**: Lukmanul (coordinate per-team presentations)

- [ ] Koordinasi presentasi top 2-3 risk dari masing-masing team (MRG, UPG, BBD)

---

## ⚠️ **FOLLOW-UP DEPLOYMENT**

### FDS Deployment Follow-up (2026-03-12 Rollback)
- [ ] Postmortem formal dengan semua tim (MRG, BBD, UPG)
- [ ] Investigate kenapa BBD v2 tidak hit endpoint MRG baru
- [ ] Investigate bug CC→Cash di binary BBD
- [ ] Review config management BBD (config switch vs binary rollback)
- [ ] Koordinasi deployment sequence untuk joint deployment berikutnya

---

## ✅ **COMPLETED**

### ✓ FDS (Fraud Detection System) Deployment
**Completed**: 2026-04  
**Note**: Setelah rollback pada 2026-03-12, deployment FDS berhasil diselesaikan.

### ✓ Two-Level ID Generation System
**Completed**: 2026-04  
**Note**: Sudah selesai dan dipakai di production.

### ✓ SPOF - Database Separation
**Completed**: 2026-04  
**Note**: Database sudah dipisah per service, SPOF mitigated.

### ✓ MCP MRG Server (mcpmrg)
**Status**: CANCELLED  
**Note**: Project mcpmrg dibatalkan.

### ✓ Reserved Budget ECV Mechanism
**Completed**: 2026-01-27

### ✓ GCP to Huawei Cloud Migration
**Completed**: Aug 2025

### ✓ MRG SPOF Analysis
**Completed**: Documented

---

## 📊 **TASK METRICS**

| Category | Count | Status |
|----------|-------|--------|
| High Priority | 7 | 1 Implementasi, 1 Phase 1 Deployed, 1 Vendor Kickoff, 2 Draft, 1 Not Started, 1 Investigating |
| Medium Priority | 2 | 1 Ongoing, 1 Todo |
| Deployment Follow-up | 1 | Pending Postmortem |
| Completed | 7 | Done |

---

## 🔗 **Related Documents**

- [[2025-01-27 - ECV for GB Rent Revamp Discussion|ECV Meeting Notes]]
- [[2026-01-20 Meeting VP Architect - Development Guidelines|VP Meeting Notes]]
- [[2025-01-12-Traffic-Spike-Analysis|Traffic Spike Details]]
- [[02-Work/Deployments/2026-03-deployment-log|March 2026 Deployment Log]]
- [[02-Work/Deployments/2026-07-deployment-log|July 2026 Deployment Log]] — New Preauth FDS go-live
- [[02-Work/Teams/UPG/01-architecture/design-docs/disbursement/disbursement-gateway-design|Disbursement Gateway Design]]
- [[02-Work/Teams/UPG/01-architecture/design-docs/qris-taximeter/README|QRIS Taximeter]]
- [[02-Work/Teams/MRG/01-architecture/design-docs/cc-charging-flow/README|CC Charging Flow Docs]]

---

**Last Review**: 2026-07-09  
**Owner**: Lukmanul Hakim
