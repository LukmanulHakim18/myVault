---
type: deployment-review
team: UPG
date: '2026-06-17'
status: pre-deployment
tags:
  - upg
  - payment
  - deployment
  - review
  - gap-analysis
---
# UPG / Payment Deployment Review - 2026-06-17

**Status:** 🔄 Pre-Deployment Review
**Deployment Time:** 21:30 (Production)
**Deployment Type:** New Release
**Source Plan:** [ClickUp - 2026.06.02 Deployment Plan](https://bluebirdgroup.clickup.com/9018711461/v/dc/8crx7d5-276098/8crx7d5-205438)

---

## 1. Ringkasan Scope

| Field | Detail |
|---|---|
| Project | UPG / Payment |
| Tech Lead | Eko Nugroho |
| Product Manager | Rayan Nurbadi |
| Target | Production |
| Tema Perubahan | Fixing pembayaran *extra overtime* pada feature GB Rent Revamp |

Service & MR yang dideploy:
- `mrg-gateway` → [MR !1218](https://git.bluebird.id/upg/mrg-gateway/-/merge_requests/1218)
- `card-payment` → [MR !1057](https://git.bluebird.id/upg/card-payment/-/merge_requests/1057)
- `routing-service` → [MR !157](https://git.bluebird.id/upg/routing-service/-/merge_requests/157)

Task terkait: UPGN-3887, UPGN-3890, UPGN-3858.

---

## 2. Temuan & Gap Analysis

### 2.1 Code Quality (Risiko Tertinggi)

| Service | Coverage | Duplications | Quality Gate | Catatan |
|---|---|---|---|---|
| mrg-gateway | 28.0% | 6.2% | **BB Level 0** | Coverage sangat rendah + duplikasi tinggi pada gateway utama di critical path payment. |
| card-payment | 79.5% | 3.9% | BB Level 2 | Sehat, acceptable. |
| routing-service | 58.2% | 0.0% | **BB Level Started** | Quality gate belum tuntas (status masih "Started"). |

> `mrg-gateway` adalah komponen kritikal namun punya coverage terendah (28%) dan hanya Level 0 → risiko regression signifikan.

### 2.2 Rollback Plan — Blocker

- Tag rollback `routing-service` tertulis `v1.` → **versi tidak lengkap/invalid**. Tidak ada tag valid untuk rollback routing-service jika terjadi insiden.
- Tag lain sudah jelas: `mrg-gateway` v1.22.0, `card-payment` v1.18.5.

### 2.3 Impact Analysis & Field Kosong

- Penomoran Impact Analysis mulai dari poin 2 → **poin 1 hilang** (kemungkinan ada impact belum tertulis).
- Field kosong yang relevan untuk Production release: PRD, Operational Impact, Business Impact, dan **Penetration Test Report** (idealnya wajib untuk service payment).

### 2.4 Checklist Belum Dikonfirmasi

- **QA Notes:** status "Testing on staging passed" masih kosong (belum di-checklist).
- **Pre-requisites:** tabel kosong total (tanpa task/PIC).
- **PM Checklist** poin 2 (notifikasi stakeholder via Email/WA) status kosong.
- **Deployment Step & PVT:** seluruh kolom Status kosong.

### 2.5 Deployment Step Minim Detail

- Kolom Details hanya berisi `cdi.bluebird.id` tanpa urutan langkah maupun dependency antar-service.
- Untuk 3 service yang saling terkait, urutan deploy & dependency sebaiknya eksplisit.
- Penomoran step melompat dari 3 ke 7.

---

## 3. Rekomendasi Prioritas Sebelum Go-Live

- [ ] **Lengkapi tag rollback `routing-service`** (blocker mutlak)
- [ ] **Konfirmasi QA staging passed** (checklist + PIC)
- [ ] **Justifikasi/mitigasi `mrg-gateway` Level 0** — sebutkan area perubahan ter-cover atau tambahkan extra monitoring/PVT
- [ ] **Pastikan `routing-service` quality gate tuntas** (bukan "Started")
- [ ] **Lengkapi poin 1 Impact Analysis** + field Operational/Business Impact
- [ ] **Tambahkan urutan & dependency deploy** di kolom Details + lengkapi Pre-requisites

---

## 4. Tim

| Nama | Role |
|---|---|
| Eko Nugroho | Tech Lead |
| Rayan Nurbadi | Product Manager |
| Sahrul Ramdoni | QA |
| Abdul Karman | QA |
| Adam Haniif | DevOps |

---

## Related Documents
- [[tech-debt]]
- [[deployment-overtime-tracking]]
