---
title: QRIS Taximeter — Project Overview (UPG)
team: UPG
status: active
phase: implementation-kickoff
owner: Lukmanul Hakim (Architecture Engineer)
created: '2026-07-03'
updated: '2026-07-03'
tags:
  - upg
  - qris
  - qris-static
  - taximeter
  - design-doc
  - active
---

# QRIS Taximeter — Project Overview (UPG)

> **Status (2026-07-03):** Vendor **sudah diputuskan** — **BCA + BNI** dipilih sebagai partner
> untuk handling **seluruh proses QRIS Statis**. Fase saat ini = **implementation kick-off**.
> Kick-off BCA: **Jumat 3 Jul 2026, 15:30** (Ruang Meeting 1 Lt 7 Gedung Baru, organizer Pinashti Sakanti).

---

## 1. Ringkasan Project

Bluebird mengaktifkan pembayaran **QRIS Statis (MPM Static)** untuk pembayaran penumpang di taksi
("QRIS Taximeter"). Pola dasar: **1 QR statis per mobil** (sticker), penumpang scan → bayar,
dana **settle ke rekening settlement sesuai MID**.

**Keputusan vendor:** **BCA dan BNI** dua-duanya ditunjuk untuk menangani **seluruh proses QRIS Statis**
(multi-acquirer). Ini menjadikan QRIS Taximeter sebuah **project tim UPG** — UPG yang mengelola
integrasi payment, rekonsiliasi, dan reporting lintas kedua bank.

**Implikasi arsitektur (multi-acquirer):** karena ada 2 bank yang jalan bersamaan, UPG perlu
kontrak internal yang **acquirer-agnostic** (satu abstraksi, dua adapter: BCA & BNI), mirip pola
provider-interface pada Disbursement Gateway. Detail keputusan ini masih perlu di-lock (lihat §6).

---

## 2. Latar Belakang Vendor Assessment

Assessment teknikal 6 bank sudah dilakukan April 2026 (fokus awal ke MPM Dynamic Flow):

| Rank | Bank | Skor | Status |
|---|---|---|---|
| 🥇 1 | **BNI** | 79/100 | Existing partner — **DIPILIH** |
| 🥈 2 | **BCA** | 73/100 | **DIPILIH** |
| 3 | Mandiri / BRI / OCBC | 65/100 | Tidak dilanjut |
| 6 | BTN | 63/100 | Tidak dilanjut |

Sumber lengkap:
- [[2026-04-21-qris-vendor-assessment]] — matrix scoring 6 bank
- [[2026-04-22-qris-vendor-report]] — report final + head-to-head BNI vs BCA

> **Catatan:** Assessment April fokus ke **Dynamic** flow. Scope project yang diputuskan sekarang
> adalah **QRIS Statis**. Grey area Static dari assessment (BCA: fallback static hanya 2 komponen /
> tips tidak terpisah) menjadi titik yang harus diklarifikasi di implementasi — lihat point confirmation.

---

## 3. Scope of Work (QRIS Statis)

| Area | Ringkas |
|---|---|
| **Onboarding** | Add Pool & Add Fleet via file template (CSV/Excel); BCA klaim same-day, 24×7 |
| **Static QRIS** | 1 QR statis per mobil, settle ke rekening sesuai MID; notifikasi via callback API |
| **Reporting** | CMR D+1 (SFTP) + Report Intraday hourly + Dashboard QRMS |
| **MDR (fee)** | Pendebetan MDR harian, **terpisah** dari settlement |
| **Refund** | API Refund tersedia (usage/SLA/type perlu dikonfirmasi) |
| **Settlement** | T+1 (~05:00); 2 mutasi terpisah (settle per-MID + debet MDR bulk) |
| **Complaint** | Via WAG / Halo BCA |

Detail per-poin + posisi Bluebird → [[bca-point-confirmation]].

---

## 4. Material & Dokumen Sumber

### Dari BCA (ZIP: "QRIS Taximeter - Tech Docs & Working Docs BCA.zip", 2 Jul 2026)
Lokasi asli: `C:\Users\lukmanul.hakim\Downloads\`

**Working doc (inti review):**
- `Bluebird General flow and documentation 1.xlsx` — sheet **Point Confirmation** = daftar hal
  yang BCA minta dikonfirmasi Bluebird; + template Tambah Pool / Tambah Fleet / CMR / Intraday.

**Technical Documentation (SNAP BI-compliant):**
- OAuth & Signature v1.1
- Generate QR MPM v2.5
- Inquiry API v2.3
- Refund API v2.2
- Static Inquiry API v1.1
- Static Payment Notify v1.1
- SNAP QRIS Notification API v2.2

**Proposal & BRD (di luar ZIP, di Downloads):**
- `2. BCA BANKING SERVICES - QRIS - PROPOSAL for Bluebird.pdf`
- `2. PROVISION OF QRIS PAYMENT FOR BLUEBIRD.pdf`
- `BRD Qris Static.pdf` (3 Jul 2026)

### Dokumen BNI
- ⚠️ **TODO** — kumpulkan tech docs & working docs BNI (belum ada di material saat ini).

---

## 5. Jadwal / Milestone

| Tanggal | Event | Status |
|---|---|---|
| 2026-04-21/22 | Vendor assessment 6 bank | ✅ Selesai |
| 2026-06-30 → 07-02 | QRIS Taximeter internal kick-off & scope alignment | ✅ |
| 2026-07-02 | BCA kirim Tech Docs & Working Docs (ZIP) | ✅ |
| 2026-07-02 | ~~BNI Implementation Kick-off~~ | ❌ Canceled (reschedule?) |
| **2026-07-03 15:30** | **BCA Implementation Kick-off** | 🔜 Hari ini |

Sumber jadwal: [[2026-W27-weekly-meetings-report]]

---

## 6. Open Questions / Keputusan yang Belum Di-lock

- **OQ-1 (Arsitektur multi-acquirer):** Bagaimana UPG memisah/route BCA vs BNI? Per-pool, per-fleet,
  atau per-region? Satu service dengan 2 adapter (acquirer-agnostic contract) — perlu konfirmasi.
- **OQ-2 (Tips di Static):** Grey area assessment — pada Static, tips/extra tidak selalu terpisah
  (BCA: MDR hanya untuk Fare+Extra "karena keterbatasan field"). Bagaimana pemisahan uang
  perusahaan vs driver di skema static? Konsisten-kah antara BCA & BNI?
- **OQ-3 (Reconciliation lintas bank):** Format report beda (CMR/Intraday BCA vs SFTP BNI) —
  UPG perlu normalisasi ke satu model reporting internal.
- **OQ-4 (Field mapping → MDR):** BCA minta list field + mapping (unique ID, Fare, Extra, tips)
  untuk perhitungan MDR. Perlu disepakati struktur data Bluebird.
- **OQ-5 (Service ownership):** Apakah masuk service UPG existing atau service baru
  (mis. `qris-service` / `qris-acquirer-service`)? Perlu keputusan arsitektur.

---

## 7. Next Steps

1. **Hari ini (kick-off BCA):** bawa [[bca-point-confirmation]] — jawab/parkir tiap poin, catat komitmen.
2. Kumpulkan material BNI (tech docs + working docs) untuk paritas dengan BCA.
3. Lock OQ-1 (arsitektur multi-acquirer) & OQ-5 (service ownership) → lanjut ke RFC/design doc UPG.
4. Susun field mapping (OQ-4) sebagai input MDR ke kedua bank.

---

## Related

- [[bca-point-confirmation]]
- [[2026-04-21-qris-vendor-assessment]]
- [[2026-04-22-qris-vendor-report]]
- [[2026-W27-weekly-meetings-report]]
