---
date: '2026-04-21'
status: in-progress
tags:
  - meeting
  - qris
  - vendor
  - assessment
title: QRIS Vendor Assessment — Technical Presentation
updated: '2026-04-22'
---

# QRIS Vendor Assessment — Technical Presentation

**Date:** 2026-04-21  
**Updated:** 2026-04-22  
**Context:** Pitching QRIS — Technical Presentation  
**Scope:** Teknikal murni (MPM Dynamic Flow, Refund, Availability & SLA, Status Check)

---

## Ranking Final (Teknikal)

| Rank | Bank | Skor |
|---|---|---|
| 🥇 1 | BNI | 79/100 |
| 🥈 2 | BCA | 73/100 |
| 🥉 3 | Mandiri | 65/100 |
| 🥉 3 | BRI | 65/100 |
| 🥉 3 | OCBC | 65/100 |
| 6 | BTN | 63/100 |

---

## Perbandingan 6 Bank

| Kriteria | Bobot | BNI | BCA | BTN | Mandiri | BRI | OCBC |
|---|---|---|---|---|---|---|---|
| MPM Dynamic Flow | 35% | 5 | 4 | 4 | 3 | 3 | 4 |
| Refund Capability | 25% | 2 | 3 | 3 | 3 | 3 | 3 |
| Availability & SLA | 25% | 5 | 4 | 3 | 4 | 4 | 2 |
| Status Check Policy | 15% | 3 | 5 | 4 | 3 | 3 | 4 |
| **Total** | **100%** | **79** | **73** | **63** | **65** | **65** | **65** |

---

## Bank BNI ⭐ EXISTING PARTNER

### Matrix Penilaian

| Kriteria | Bobot | Skor | Keterangan |
|---|---|---|---|
| MPM Dynamic Flow | 35% | 5 | Dynamic QRIS. Expiry up to 7 hari (configurable). Tipping ada (customer input). **Fare/extra/tipping 3 komponen terpisah** di dashboard & laporan (Trip Fare, Extra Fare, Tip — jelas dan eksplisit). Static fallback ada. API + SFTP onboarding. Retry mechanism ada. |
| Refund Capability | 25% | 2 | Refund API **masih dalam tahap development**. Good Faith D+2. Scope lengkap: duplicate, incorrect amount, complaint. Issuer coverage tidak disebutkan masalah. |
| Availability & SLA | 25% | 5 | Uptime **99.9% proven** — data aktual 2025 bersama Bluebird: 12.447 juta dari 12.451 juta transaksi sukses. Settlement **D+0 per batch** (terbaik). Onboarding 24/7 tanpa jam operasional. |
| Status Check Policy | 15% | 3 | Real-time callback untuk success. **Callback notif tersedia untuk Static QRIS juga.** 3 jenis report SFTP: Success, Pending, Failed (T+1). Dashboard real-time. Resend callback tersedia di dashboard. **Pending handling manual — driver diminta foto bukti transaksi**, bukan otomatis. |

### Keunggulan BNI

- ✅ **Existing partnership** — data aktual 2025 bersama Bluebird, bukan klaim
- ✅ **Fare/extra/tipping 3 komponen terpisah** di semua kondisi (Dynamic & Static)
- ✅ **Settlement D+0 per batch** — tercepat dari semua bank
- ✅ **Good Faith D+2**
- ✅ **3 report SFTP**: Success, Pending, Failed — paling lengkap
- ✅ **Resend callback** tersedia dari dashboard
- ✅ Onboarding 24/7 tanpa jam operasional
- ✅ 3,7 juta merchant QRIS — ekosistem terbesar

### Grey Areas / Open Questions

- 🔴 **Refund API masih development** — belum production-ready
- ⚠ Retry mechanism detail — berapa kali, interval berapa detik
- ⚠ Expiry 7 hari — perlu konfirmasi bisa dikonfigurasi ke menit
- ⚠ **Pending handling manual** — driver harus foto bukti transaksi, bukan notif otomatis
- ⚠ Rate limit callback/inquiry — tidak disebutkan
- ⚠ SLA onboarding fleet baru — tidak ada angka eksplisit
- ⚠ Resend callback — hanya manual dari dashboard atau ada via API juga?

---

## Bank BCA

### Matrix Penilaian

| Kriteria | Bobot | Skor | Keterangan |
|---|---|---|---|
| MPM Dynamic Flow | 35% | 4 | Error code eksplisit jika gagal (tidak ada pending state). Expiry 5–120 menit (configurable). Push notif + retry 2x/30s. Order ID dedicated field. Support tips. Dynamic 3 komponen, **Static fallback hanya 2 komponen**. |
| Refund Capability | 25% | 3 | **Refund API sudah ada**. Good Faith D+2. Issuer coverage tidak disebutkan — abu-abu, tidak berarti aman. |
| Availability & SLA | 25% | 4 | Latency 200ms eksplisit. Uptime 99.97% (klaim). Notif retry otomatis. Intraday report per jam. **Onboarding Pool MID cut-off 16:00 WIB hari kerja**. |
| Status Check Policy | 15% | 5 | Push notif real-time + retry 2x/30s eksplisit. Polling bebas tanpa rate limit. Fallback ke inquiry API. Intraday report per jam. |

### Keunggulan BCA

- ✅ Latency **200ms eksplisit**
- ✅ Tidak ada pending state — langsung error code jika gagal
- ✅ Expiry fleksibel: 5–120 menit
- ✅ Push notif + retry otomatis **2x/30s** — paling detail & robust
- ✅ Intraday report **per jam** — granularitas terbaik
- ✅ Uptime 99.97% — tertinggi dari semua bank (klaim)
- ✅ Proposal paling detail memahami pain point Bluebird

### Grey Areas (Teknikal)

- 🔴 **Refund API belum ada** di grey areas BCA dihapus — **Refund API sudah ada** ✅
- ⚠ **Onboarding Pool MID cut-off jam 16:00 WIB hari kerja** — constraint jam operasional
- ⚠ **Static QRIS fallback hanya 2 komponen** — saat fallback, breakdown 3 komponen tidak tersedia
- ⚠ **Issuer coverage** — tidak disebutkan, status abu-abu (tidak berarti aman)

---

## Bank Mandiri

### Matrix Penilaian (Teknikal)

| Kriteria | Bobot | Skor | Keterangan |
|---|---|---|---|
| MPM Dynamic Flow | 35% | 3 | Order ID via `partnerReferenceNo`. Expiry 5–15 menit. Pending = dianggap gagal, customer harus repayment |
| Refund Capability | 25% | 3 | Pending & success bisa refund via API. Ada issuer yang hold refund — tidak fully automated |
| Availability & SLA | 25% | 4 | Uptime 99.98%. Generate QR ≤300ms. Status change ≤15 menit |
| Status Check Policy | 15% | 3 | Polling min 10 detik. Pull only — tidak ada push notif |

### Open Questions

- ⚠ Issuer hold refund — butuh list issuer mana yang hold dan estimasi durasi hold
- ⚠ 300ms generate QR — konfirmasi P95/P99 atau average
- ⚠ Dashboard Yokke — apakah ada API/webhook export untuk integrasi ke Grafana/internal monitoring?

---

## Bank BTN

### Matrix Penilaian (Teknikal)

| Kriteria | Bobot | Skor | Keterangan |
|---|---|---|---|
| MPM Dynamic Flow | 35% | 4 | Order ID custom unique ID (SNAP BI-compliant). Expiry 5 menit (adjustable). Real-time push notif. Fallback ke Query API. BTN cover transaksi pending |
| Refund Capability | 25% | 3 | Refund API tersedia. Tidak semua issuer support. SLA tidak eksplisit |
| Availability & SLA | 25% | 3 | API success rate 99.9%. Generate dynamic QRIS under 2s — gap signifikan vs bank lain |
| Status Check Policy | 15% | 4 | Push notif real-time QR dinamis & statis. Query API fallback. Polling 30 detik |

### Open Questions

- ⚠ Rate limit Query API — belum dikonfirmasi

---

## Bank BRI

### Matrix Penilaian (Teknikal)

| Kriteria | Bobot | Skor | Keterangan |
|---|---|---|---|
| MPM Dynamic Flow | 35% | 3 | Dynamic QRIS supported. Expiry default 5 menit, configurable up to 24 jam. Tipping & static fallback ada di proposal tapi **belum production-ready**. Retry mekanisme ada. |
| Refund Capability | 25% | 3 | Refund API tersedia. Overpayment ada endpoint refund. Good Faith D+2, third-party T+14. |
| Availability & SLA | 25% | 4 | Uptime 99.9%. PTEN terintegrasi — onboarding 5 menit, 24/7 tanpa jam operasional. Bulk via upload + SFTP. |
| Status Check Policy | 15% | 3 | Callback notif untuk success. Inquiry API: 4 status (Success, Unpaid, Expired, Invalid). Status unpaid/gantung hanya diketahui dari expired QR — tidak ada notif explicit. |

### Catatan

- ✅ PTEN 24/7 tanpa jam operasional
- ✅ Onboarding bulk via upload & SFTP
- ⚠ Tipping & hybrid QRIS fallback masih in-development
- ⚠ Status unpaid bergantung pada expired QR, bukan notif aktif

---

## Bank OCBC

### Matrix Penilaian (Teknikal)

| Kriteria | Bobot | Skor | Keterangan |
|---|---|---|---|
| MPM Dynamic Flow | 35% | 4 | Latency generate QRIS **92ms** (tercepat). Retry 3x/15s setelah timeout 60 detik. Expiry up to 24 jam. SNAP BI-compliant. Static QRIS fallback ada, notif callback juga. Query status bebas tanpa rate limit. |
| Refund Capability | 25% | 3 | Refund & dispute supported. Good Faith SLA **30–90 hari kerja** — paling lama. Reserved fund Rp 10jt/bulan untuk pending transaction. |
| Availability & SLA | 25% | 2 | Uptime **99.8%** — di bawah requirement minimum 99.9%. OCBC sendiri menjawab NO untuk poin ini. |
| Status Check Policy | 15% | 4 | Semua status dikirim (success, expired, pending on-us/off-us diasumsikan success). Query status bebas tanpa rate limit. Static QRIS notif ada. |

### Catatan

- ✅ Latency 92ms — tercepat dari semua bank
- ✅ Semua status dikirim via callback
- ✅ Query status bebas tanpa rate limit
- 🔴 Uptime 99.8% — tidak memenuhi requirement minimum, dijawab NO sendiri
- 🔴 Good Faith SLA 30–90 hari kerja — jauh di atas bank lain
- ⚠ Tips masuk ke total amount → potensi masalah pemisahan uang perusahaan vs driver
- ⚠ Client list terbatas (5 client), belum ada referensi ride-hailing

### Open Questions

- Timeline mencapai uptime 99.9%?
- Tips dalam total amount — bagaimana mekanisme settlement pemisahan uang perusahaan vs driver?

---

## Related

- **Project lanjutan:** [[02-Work/Teams/UPG/01-architecture/design-docs/qris-taximeter/README|QRIS Taximeter (UPG)]] — vendor DECIDED (BCA + BNI) untuk QRIS Statis
