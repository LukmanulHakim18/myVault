---
date: '2026-04-22'
status: final
tags:
  - report
  - qris
  - vendor
  - assessment
title: QRIS Vendor Assessment Report — Technical Review
---

# QRIS Vendor Assessment Report — Technical Review

**Tanggal:** 22 April 2026  
**Prepared by:** Lukmanul Hakim — Architecture Engineer  
**Scope:** Technical assessment murni (MPM Dynamic Flow, Refund Capability, Availability & SLA, Status Check Policy)  
**Bank yang dinilai:** BNI, BCA, Mandiri, BRI, OCBC, BTN

---

## 1. Metodologi Penilaian

Setiap bank dinilai berdasarkan 4 kriteria teknikal dengan bobot sebagai berikut:

| Kriteria | Bobot | Alasan |
|---|---|---|
| MPM Dynamic Flow | 35% | Core requirement — generate, fallback, tipping, fare breakdown |
| Refund Capability | 25% | Operasional kritis — penanganan transaksi bermasalah |
| Availability & SLA | 25% | Reliability sistem untuk ride-hailing skala besar |
| Status Check Policy | 15% | Real-time visibility transaksi untuk driver & sistem |

Skala penilaian: 1–5 per kriteria. Total skor maksimal 100.

---

## 2. Ranking Final (Teknikal)

| Rank | Bank | Skor | Status |
|---|---|---|---|
| 🥇 1 | BNI | 79/100 | Existing Partner |
| 🥈 2 | BCA | 73/100 | New |
| 🥉 3 | Mandiri | 65/100 | New |
| 🥉 3 | BRI | 65/100 | New |
| 🥉 3 | OCBC | 65/100 | New |
| 6 | BTN | 63/100 | New |

---

## 3. Matrix Perbandingan 6 Bank

| Kriteria | Bobot | BNI | BCA | Mandiri | BRI | OCBC | BTN |
|---|---|---|---|---|---|---|---|
| MPM Dynamic Flow | 35% | 5 | 4 | 3 | 3 | 4 | 4 |
| Refund Capability | 25% | 2 | 3 | 3 | 3 | 3 | 3 |
| Availability & SLA | 25% | 5 | 4 | 4 | 4 | 2 | 3 |
| Status Check Policy | 15% | 3 | 5 | 3 | 3 | 4 | 4 |
| **Total** | **100%** | **79** | **73** | **65** | **65** | **65** | **63** |

---

## 4. Detail Penilaian Per Bank

### 4.1 Bank BNI ⭐ Existing Partner

| Kriteria | Bobot | Skor | Keterangan |
|---|---|---|---|
| MPM Dynamic Flow | 35% | 5 | Dynamic QRIS. Expiry up to 7 hari (configurable). Tipping ada (customer input). **Fare/extra/tipping 3 komponen terpisah** di dashboard & laporan (Trip Fare, Extra Fare, Tip) — berlaku di Dynamic maupun Static. Fallback Static ada + callback notif. API + SFTP onboarding. |
| Refund Capability | 25% | 2 | Refund API **masih dalam tahap development**. Good Faith D+2. Dispute handling tersedia. |
| Availability & SLA | 25% | 5 | Uptime **99.9% proven** — data aktual 2025 bersama Bluebird: 12.447 juta dari 12.451 juta transaksi sukses. Settlement **D+0 per batch** (tercepat). Onboarding 24/7 tanpa jam operasional. |
| Status Check Policy | 15% | 3 | Real-time callback (Dynamic & Static). 3 jenis report SFTP: Success, Pending, Failed. Dashboard real-time. Resend callback tersedia di dashboard. **Pending handling manual — driver diminta foto bukti transaksi.** |

**Keunggulan:**
- ✅ Existing partnership — proven data aktual 2025 bersama Bluebird
- ✅ Fare/extra/tipping 3 komponen terpisah di semua kondisi
- ✅ Settlement D+0 per batch — tercepat
- ✅ 3 report SFTP: Success, Pending, Failed
- ✅ Resend callback dari dashboard
- ✅ Onboarding 24/7 tanpa jam operasional

**Grey Areas:**
- 🔴 Refund API masih development
- ⚠ Pending handling manual — driver foto bukti
- ⚠ Latency generate QR diklaim cepat, belum ada angka eksplisit
- ⚠ Retry mechanism — detail tidak disebutkan
- ⚠ Rate limit callback/inquiry tidak disebutkan

---

### 4.2 Bank BCA

| Kriteria | Bobot | Skor | Keterangan |
|---|---|---|---|
| MPM Dynamic Flow | 35% | 4 | Tidak ada pending state — langsung error code jika gagal. Expiry 5–120 menit (configurable). Push notif + retry 2x/30s. Support tips. Dynamic 3 komponen, **Static fallback hanya 2 komponen**. |
| Refund Capability | 25% | 3 | **Refund API sudah ada**. Good Faith D+2. Issuer coverage tidak disebutkan eksplisit — abu-abu. |
| Availability & SLA | 25% | 4 | Latency **200ms eksplisit**. Uptime 99.97% (klaim). Intraday report per jam. **Onboarding Pool MID cut-off 16:00 WIB hari kerja.** |
| Status Check Policy | 15% | 5 | Push notif real-time + retry **2x/30s eksplisit**. Polling bebas tanpa rate limit. Callback notif Dynamic & Static. Intraday report per jam. |

**Keunggulan:**
- ✅ Latency 200ms — angka eksplisit
- ✅ Tidak ada pending state — langsung error code
- ✅ Refund API sudah ada
- ✅ Push notif retry 2x/30s — paling detail
- ✅ Intraday report per jam — granularitas terbaik
- ✅ Uptime 99.97% (klaim tertinggi)

**Grey Areas:**
- ⚠ Onboarding Pool MID cut-off 16:00 WIB hari kerja
- ⚠ Static QRIS fallback hanya 2 komponen (fare + extra, tip tidak terpisah)
- ⚠ Issuer coverage — abu-abu, tidak eksplisit

---

### 4.3 Bank Mandiri

| Kriteria | Bobot | Skor | Keterangan |
|---|---|---|---|
| MPM Dynamic Flow | 35% | 3 | Order ID via `partnerReferenceNo`. Expiry 5–15 menit. Pending dianggap gagal — customer harus repayment. |
| Refund Capability | 25% | 3 | Refund API ada. Ada issuer yang hold refund — tidak fully automated. |
| Availability & SLA | 25% | 4 | Uptime 99.98%. Generate QR ≤300ms. Status change ≤15 menit. |
| Status Check Policy | 15% | 3 | Polling min 10 detik. Pull only — tidak ada push notif. |

**Open Questions:**
- ⚠ List issuer yang hold refund & estimasi durasi
- ⚠ 300ms generate QR — P95/P99 atau average?
- ⚠ Dashboard Yokke — ada API/webhook export ke Grafana?

---

### 4.4 Bank BRI

| Kriteria | Bobot | Skor | Keterangan |
|---|---|---|---|
| MPM Dynamic Flow | 35% | 3 | Dynamic QRIS. Expiry up to 24 jam. Tipping & hybrid static fallback **masih in-development**. |
| Refund Capability | 25% | 3 | Refund API tersedia. Good Faith D+2. Third-party T+14. |
| Availability & SLA | 25% | 4 | Uptime 99.9%. PTEN terintegrasi — onboarding 5 menit, **24/7 tanpa jam operasional**. Bulk via upload + SFTP. |
| Status Check Policy | 15% | 3 | Callback success. Inquiry API 4 status. Unpaid hanya diketahui dari expired QR — tidak ada notif aktif. |

**Catatan:**
- ✅ PTEN 24/7 tanpa jam operasional
- ⚠ Tipping & hybrid fallback masih in-development

---

### 4.5 Bank OCBC

| Kriteria | Bobot | Skor | Keterangan |
|---|---|---|---|
| MPM Dynamic Flow | 35% | 4 | Latency **92ms** (tercepat). Retry 3x/15s. Expiry up to 24 jam. SNAP BI-compliant. Callback Dynamic & Static. |
| Refund Capability | 25% | 3 | Refund supported. Good Faith **30–90 hari kerja** — paling lama. Reserved fund Rp 10jt/bulan. |
| Availability & SLA | 25% | 2 | Uptime **99.8%** — **di bawah requirement minimum 99.9%**. OCBC sendiri menjawab NO. |
| Status Check Policy | 15% | 4 | Semua status dikirim via callback. Query status bebas tanpa rate limit. |

**Red Flags:**
- 🔴 Uptime 99.8% — tidak memenuhi requirement, dijawab NO sendiri
- 🔴 Good Faith 30–90 hari kerja — jauh di atas bank lain
- ⚠ Tips masuk total amount → potensi masalah pemisahan uang perusahaan vs driver

---

### 4.6 Bank BTN

| Kriteria | Bobot | Skor | Keterangan |
|---|---|---|---|
| MPM Dynamic Flow | 35% | 4 | SNAP BI-compliant. Expiry 5 menit (adjustable). Push notif real-time. BTN cover transaksi pending. |
| Refund Capability | 25% | 3 | Refund API tersedia. Tidak semua issuer support. SLA tidak eksplisit. |
| Availability & SLA | 25% | 3 | Uptime 99.9%. Generate QR **under 2s** — gap signifikan vs bank lain. |
| Status Check Policy | 15% | 4 | Push notif Dynamic & Static. Query API fallback. Polling 30 detik. |

**Open Questions:**
- ⚠ Rate limit Query API — belum dikonfirmasi

---

## 5. Komparasi Head-to-Head: BNI vs BCA

> Dua kandidat teratas dengan selisih skor 6 poin.

| Kriteria | Detail | BNI | BCA |
|---|---|---|---|
| **MPM Dynamic Flow** | **Skor** | **5** | **4** |
| | Latency generate QR | ⚠ Diklaim cepat, belum ada angka | ✅ 200ms eksplisit |
| | Expiry configurable | ✅ Up to 7 hari | ✅ 5–120 menit |
| | Tipping | ✅ Ada | ✅ Ada |
| | Fare/Extra/Tip — Dynamic | ✅ 3 komponen | ✅ 3 komponen |
| | Fare/Extra/Tip — Static | ✅ **3 komponen** | ⚠ 2 komponen |
| | Pending state | ⚠ Ada — driver foto bukti manual | ✅ Tidak ada pending state |
| | Static fallback + notif | ✅ Ada | ✅ Ada |
| | Retry mechanism | ⚠ Ada, tidak eksplisit | ✅ 2x/30s eksplisit |
| **Refund Capability** | **Skor** | **2** | **3** |
| | Refund API | 🔴 Masih development | ✅ Sudah ada |
| | Good Faith SLA | ✅ D+2 | ✅ D+2 |
| | Issuer coverage | ⚠ Tidak disebutkan | ⚠ Tidak disebutkan |
| **Availability & SLA** | **Skor** | **5** | **4** |
| | Uptime | ✅ **99.9% proven** (data aktual 2025) | ✅ 99.97% (klaim) |
| | Settlement | ✅ **D+0 per batch** | ⚠ D+1 |
| | Onboarding jam operasional | ✅ **24/7 tanpa batas** | ⚠ Pool MID cut-off 16:00 WIB |
| | Existing integration | ✅ **Sudah berjalan dengan Bluebird** | ❌ Baru |
| **Status Check** | **Skor** | **3** | **5** |
| | Push notif Dynamic | ✅ Real-time | ✅ Real-time + retry 2x/30s |
| | Push notif Static | ✅ Ada | ✅ Ada |
| | Notif retry detail | ⚠ Tidak disebutkan | ✅ 2x/30s eksplisit |
| | Pending handling | 🔴 **Manual — driver foto bukti** | ✅ Tidak ada pending state |
| | Intraday report | ✅ Real-time dashboard | ✅ **Per jam** (lebih granular) |
| | Polling rate limit | ⚠ Tidak disebutkan | ✅ Bebas tanpa rate limit |
| | Resend callback | ✅ **Ada di dashboard** | ❌ Tidak disebutkan |
| | SFTP report | ✅ **3 jenis** (Success/Pending/Failed) | ✅ 2 jenis (Intraday + CMR) |

### Head-to-Head Summary

| Faktor | Pemenang |
|---|---|
| Latency eksplisit | 🏆 BCA (200ms vs klaim) |
| Fare breakdown semua kondisi | 🏆 BNI (Dynamic & Static 3 komponen) |
| Refund API | 🏆 BCA (sudah ada, BNI masih dev) |
| Uptime proven | 🏆 BNI (data aktual Bluebird 2025) |
| Settlement speed | 🏆 BNI (D+0 vs D+1) |
| Pending handling | 🏆 BCA (tidak ada pending state) |
| Notif retry eksplisit | 🏆 BCA (2x/30s) |
| Intraday granularity | 🏆 BCA (per jam) |
| Onboarding 24/7 | 🏆 BNI |
| SFTP report lengkap | 🏆 BNI (3 jenis) |
| Resend callback | 🏆 BNI |
| Existing integration | 🏆 BNI |

**BNI menang 7-5** di head-to-head. BCA unggul di spesifikasi API yang terukur dan eksplisit. BNI unggul di reliability operasional dan proven track record bersama Bluebird.

---

## 6. Kesimpulan

**BNI direkomendasikan sebagai vendor utama** berdasarkan:
1. Satu-satunya bank dengan **data proven** bersama Bluebird (bukan klaim)
2. **Fare/extra/tipping 3 komponen** terpisah di semua kondisi — critical untuk settlement
3. **Settlement D+0** — paling cepat, mendukung cash flow operasional
4. **Onboarding 24/7** — tidak ada constraint jam operasional

**BCA sebagai alternatif kuat** jika BNI tidak terpilih, dengan catatan:
- Refund API sudah siap (keunggulan vs BNI)
- Status check paling robust (retry eksplisit, intraday per jam)
- Perlu konfirmasi latency P95/P99 dan issuer coverage

**Bank lain (Mandiri, BRI, OCBC, BTN)** belum memenuhi keunggulan kompetitif yang cukup untuk direkomendasikan pada tahap ini.

---

*Report ini bersifat technical assessment murni. Keputusan final mempertimbangkan faktor komersial dan operasional di luar scope dokumen ini.*

---

## Related

- **Project lanjutan:** [[02-Work/Teams/UPG/01-architecture/design-docs/qris-taximeter/README|QRIS Taximeter (UPG)]] — vendor DECIDED (BCA + BNI) untuk QRIS Statis
