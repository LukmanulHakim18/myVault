---
title: QRIS Taximeter — BCA Point Confirmation & Posisi Bluebird
team: UPG
status: active
owner: Lukmanul Hakim (Architecture Engineer)
created: '2026-07-03'
updated: '2026-07-03'
source: 'Bluebird General flow and documentation 1.xlsx (sheet: Point Confirmation)'
tags:
  - upg
  - qris
  - qris-static
  - bca
  - working-doc
---

# QRIS Taximeter — BCA Point Confirmation & Posisi Bluebird

> Ekstrak dari working doc BCA (`Bluebird General flow and documentation 1.xlsx`, sheet **Point
> Confirmation**). Kolom **Posisi/Jawaban Bluebird** diisi untuk bahan kick-off 3 Jul 2026.
> Yang ditandai *(draft)* = rekomendasi awal berdasarkan assessment April, **belum final** —
> konfirmasi di meeting.

---

## 1. Onboarding

| # | Poin BCA | Mekanisme BCA | Pertanyaan ke Bluebird | Posisi/Jawaban Bluebird |
|---|---|---|---|---|
| 1 | Add Pool | Format file Bluebird → BCA (template CSV/Excel) | Tanggapan atas template? | ⬜ TBD |
| 2 | SLA Add Pool | Proses **same day** | Tanggapan atas SLA? | ⬜ TBD |
| 3 | Add Fleet (Static QR per mobil) | Template CSV/Excel + mandatory fields | Tanggapan atas template? | ⬜ TBD |
| 4 | SLA Add Fleet | Same day, **24×7** | Tanggapan atas SLA? | ⬜ TBD *(24×7 sesuai keunggulan onboarding — terima)* |
| 5 | Onboarding pool existing | — | — | ⬜ TBD |

## 2. Payment Flow — Static QRIS (scope utama)

| # | Poin BCA | Mekanisme BCA | Pertanyaan ke Bluebird | Posisi/Jawaban Bluebird |
|---|---|---|---|---|
| 6 | Generate QR per fleet | 1 QR statis per mobil, semua settle ke rekening sesuai MID | Ada pertanyaan? | ⬜ TBD |
| 7 | Notification (callback) | Static Payment Notify API | Ada pertanyaan soal notifikasi? | ⬜ TBD |
| 8 | Timeout & expired QR | Expiry configurable | Max waktu QR aktif? | ⬜ TBD |

## 3. Payment Flow — Dynamic (referensi/opsional)

| # | Poin BCA | Mekanisme BCA | Pertanyaan ke Bluebird | Posisi/Jawaban Bluebird |
|---|---|---|---|---|
| 9 | +6 field rekonsiliasi | Generate QRIS MPM | ⭐ **List field + mapping** (unique ID, Fare, Extra, lainnya) + format kirim — dipakai hitung MDR | ⬜ **PRIORITAS** — siapkan struktur data |
| 10 | Tips | via field `convenience fee` | Tanggapan penggunaan field ini untuk tips? | ⬜ TBD *(cek pemisahan uang perusahaan vs driver)* |
| 11 | Unique identifier | via `partnerReferenceNo` | Bisa input unique ID di field ini? | ⬜ TBD *(gunakan order/trip ID Bluebird)* |

## 4. Reporting

| # | Poin BCA | Mekanisme BCA | Pertanyaan ke Bluebird | Posisi/Jawaban Bluebird |
|---|---|---|---|---|
| 12 | CMR D+1 (Merchant Report) | Format CMR via SFTP | (a) 1 file semua MID atau pecah per-MID? (b) SFTP punya BCA atau Bluebird? | ⬜ TBD |
| 13 | Detail field CMR | Format CMR | Tanggapan atas format? | ⬜ TBD |
| 14 | Report Intraday | Hourly | Tanggapan atas format? | ⬜ TBD |

## 5. Dashboard

| # | Poin BCA | Mekanisme BCA | Pertanyaan ke Bluebird | Posisi/Jawaban Bluebird |
|---|---|---|---|---|
| 15 | Dashboard QRMS | Bisa download report dari dashboard | Masih perlu Intraday kalau sudah ada dashboard? | ⬜ TBD *(kemungkinan tetap perlu untuk otomasi/rekon internal)* |

## 6. Merchant Discount Rate (MDR)

| # | Poin BCA | Mekanisme BCA | Pertanyaan ke Bluebird | Posisi/Jawaban Bluebird |
|---|---|---|---|---|
| 16 | Fee deduction Dynamic | Debet **harian, terpisah** dari settlement | Tanggapan? | ⬜ TBD |
| 17 | Fee deduction Static QR | Debet harian; MDR untuk **Fare + Extra** (keterbatasan field); BCA beri info MDR di CMR | Tanggapan? | ⬜ **CEK** — implikasi tips tak kena MDR / pemisahan komponen |
| 18 | Retry fee deduction | Gagal → retry esok hari (2× debet) | Tanggapan? | ⬜ TBD |
| 19 | Blokir saldo untuk MDR | Blokir saldo | Bersedia? | ⬜ TBD *(butuh keputusan Finance)* |

## 7. Refund

| # | Poin BCA | Mekanisme BCA | Pertanyaan ke Bluebird | Posisi/Jawaban Bluebird |
|---|---|---|---|---|
| 20 | Refund usage | Refund QRIS MPM API | API atau manual? | ⬜ TBD |
| 21 | Refund SLA | — | Berapa SLA refund Bluebird? | ⬜ TBD |
| 22 | Refund type | — | Selalu full (Fare+Tips+Extra) atau ada partial? | ⬜ TBD |
| 23 | Refund rules | — | Case: (1) double bayar, (2) refund extra — ada case lain? | ⬜ TBD |

## 8. Settlement

| # | Poin BCA | Mekanisme BCA | Pertanyaan ke Bluebird | Posisi/Jawaban Bluebird |
|---|---|---|---|---|
| 24 | Settlement | T+1 (~05:00) | — | ⬜ TBD |
| 25 | Mutasi settlement | 2 mutasi terpisah: (a) settle dana per-MID, (b) debet MDR bulk | Tanggapan atas 2 mutasi? | ⬜ TBD |
| 26 | Rekonsiliasi rekening | — | Pakai rekening koran BCA atau report **MT940**? | ⬜ TBD |
| 27 | Debet MDR & rekening | 1× debet untuk semua MID | (a) Tanggapan? (b) Pakai 1 rekening sama dgn settlement untuk MDR? | ⬜ TBD |

## 9. Complaint Handling

| # | Poin BCA | Mekanisme BCA | Pertanyaan ke Bluebird | Posisi/Jawaban Bluebird |
|---|---|---|---|---|
| 28 | Metode complaint | Via WAG / Halo BCA (#) | — | ⬜ TBD |
| 29 | Jenis complaint & SLA | — | — | ⬜ TBD |
| 30 | Jumlah complaint | Dispute process | (data beberapa bulan terakhir) | ⬜ TBD |

---

## Poin Kritis untuk Ditekan di Kick-off

1. **Field mapping → MDR (#9):** BCA butuh ini untuk kalkulasi MDR — item paling urgent, siapkan struktur data.
2. **Pemisahan komponen di Static (#10, #17):** tips/extra vs Fare — grey area assessment April. Pastikan
   uang perusahaan vs driver tidak tercampur, dan konsisten dengan skema BNI.
3. **Settlement 2 mutasi + rekonsiliasi (#25–27):** pengaruh langsung ke proses rekon UPG & Finance.
4. **Multi-acquirer:** semua jawaban di sini harus dicek paritasnya dengan BNI agar model internal UPG seragam.

---

## Related

- [[02-Work/Teams/UPG/01-architecture/design-docs/qris-taximeter/README|README]] — Project overview QRIS Taximeter
- [[2026-04-22-qris-vendor-report]] — assessment (grey area static BCA)
