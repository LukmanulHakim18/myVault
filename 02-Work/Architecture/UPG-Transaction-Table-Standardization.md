---
title: UPG - Standarisasi Table Transaksi Payment Gateway
tags:
  - architecture
  - upg
  - payment-gateway
  - bluebird
  - database
  - microservices
date: '2026-04-28'
status: draft-proposal
author: lukmanul.hakim@bluebirdgroup.com
audience: Tim UPG
---
# UPG — Standarisasi Table Transaksi Payment Gateway

> **Tujuan dokumen ini:** Menjadi bahan diskusi dan saran untuk tim UPG tentang bagaimana sebaiknya arsitektur database transaksi dirancang agar mudah di-maintain, tidak membingungkan developer, dan tetap resilient (tahan terhadap kegagalan).

---

## Kondisi Saat Ini (Existing)

Saat ini setiap payment method memiliki tabel transaksinya masing-masing:

```
transaction_shopee
transaction_ovo
transaction_dana
transaction_gopay
... dst
```

Masing-masing service juga punya database sendiri (microservices pattern, DB per service).

### Apa yang sudah benar ✅

- **Fault isolation** — kalau Shopee DB mati, OVO dan DANA tetap bisa memproses transaksi. Ini prinsip yang benar.
- **Independensi per service** — tiap tim bisa manage schema mereka sendiri tanpa khawatir mempengaruhi service lain.

### Apa yang bermasalah ❌

| Masalah | Dampak |
|---|---|
| Tidak ada Transaction ID yang global dan unik | ID transaksi bisa collision antar payment method (misalnya OVO dan Shopee sama-sama punya `txn_id = 1`) |
| Tidak ada titik query terpusat | Untuk laporan atau fitur lintas payment, developer harus query ke semua service/tabel satu per satu |
| Fitur cross-payment sangat sulit | Contoh: menampilkan semua riwayat transaksi user dari semua metode pembayaran dalam satu halaman |
| Developer experience buruk | Developer baru bingung karena tidak ada standar — tiap tabel beda struktur, beda cara aksesnya |

---

## Akar Masalahnya Apa?

Arsitektur DB per service itu **prinsipnya benar**. Yang salah bukan konsepnya, tapi **implementasinya tidak lengkap**. Ada tiga hal yang hilang:

1. **Global Unique Transaction ID** — setiap transaksi perlu punya ID yang unik di seluruh sistem UPG, bukan hanya unik di dalam satu service
2. **Central Transaction Registry** — sebuah "buku besar" yang mencatat ringkasan semua transaksi dari semua payment method, untuk kebutuhan query dan reporting
3. **Event-driven mechanism** — cara agar setiap payment service bisa "memberitahu" sistem lain ketika ada transaksi baru tanpa harus saling memanggil langsung

---

## Solusi yang Direkomendasikan

### Gambaran Besar Arsitektur

```
WRITE PATH — Siapa yang handle request pembayaran:
──────────────────────────────────────────────────────────
Client → API Gateway → Shopee Service → DB Shopee ─┐
                     → OVO Service   → DB OVO    ──┼──→ Kafka (Message Broker)
                     → DANA Service  → DB DANA  ───┘

READ PATH — Untuk query dan reporting:
──────────────────────────────────────────────────────────
Kafka ──→ Core Service → DB Core (aggregated transactions)
                ↓
     Melayani query seperti:
     - Semua transaksi user X dari semua payment method
     - Total revenue per hari per payment method
     - Status transaksi berdasarkan upg_transaction_id
```

---

### Komponen 1: Global Transaction ID

Setiap transaksi yang masuk ke UPG harus mendapatkan satu ID unik yang berlaku di seluruh sistem. ID ini di-generate menggunakan **UUID v4** — formatnya seperti ini:

```
f47ac10b-58cc-4372-a567-0e02b2c3d479
```

**Kenapa UUID v4?**
- Bisa di-generate oleh service mana pun tanpa perlu koordinasi ke service lain
- Probabilitas collision hampir nol (122-bit random)
- Kalau Core Service mati pun, payment service tetap bisa generate ID sendiri

Setiap payment service menyimpan `upg_transaction_id` ini di tabelnya masing-masing sebagai referensi:

```sql
-- Di DB OVO:
CREATE TABLE transaction_ovo (
    id              BIGINT PRIMARY KEY AUTO_INCREMENT,
    upg_transaction_id  UUID NOT NULL UNIQUE,  -- ← global ID dari UPG
    ovo_reference   VARCHAR(100),              -- ← ID dari OVO
    amount          DECIMAL(15,2),
    status          VARCHAR(20),
    created_at      TIMESTAMP,
    ...
);
```

---

### Komponen 2: Central Transaction Registry (Core Service)

Core Service adalah service khusus yang menyimpan **ringkasan semua transaksi** dari semua payment method. Tujuannya satu: **melayani query lintas payment method**.

**PENTING — Core Service BUKAN proxy atau forwarder request:**

```
❌ SALAH — Core jadi perantara (anti-pattern):
Client → Core → OVO Service → DB OVO

✅ BENAR — Core hanya read model:
Client → OVO Service → DB OVO (write path, tidak melibatkan Core)
                  ↓ publish event
              Kafka → Core Service → DB Core (async, tidak blocking)
```

Schema minimal di DB Core:

```sql
CREATE TABLE transactions (
    upg_transaction_id  UUID PRIMARY KEY,
    payment_method      VARCHAR(50),     -- 'ovo', 'shopee', 'dana'
    status              VARCHAR(20),     -- 'INITIATED', 'SUCCESS', 'FAILED'
    amount              DECIMAL(15,2),
    currency            VARCHAR(10) DEFAULT 'IDR',
    merchant_id         VARCHAR(100),
    customer_id         VARCHAR(100),
    created_at          TIMESTAMP,
    updated_at          TIMESTAMP
);
```

Core Service tidak menerima request pembayaran. Core hanya:
- Mendengarkan event dari Kafka
- Menyimpan ringkasan transaksi ke DB-nya sendiri
- Melayani query dari dashboard, reporting, atau service lain yang butuh data lintas payment

---

### Komponen 3: Event-Driven via Kafka

Setiap payment service publish event ke Kafka setiap kali ada perubahan status transaksi:

```
Topic: upg.transaction.created
Topic: upg.transaction.updated
Topic: upg.transaction.failed
```

Contoh payload event:

```json
{
  "upg_transaction_id": "f47ac10b-58cc-4372-a567-0e02b2c3d479",
  "payment_method": "ovo",
  "event_type": "TRANSACTION_SUCCESS",
  "amount": 150000,
  "customer_id": "usr_123",
  "timestamp": "2026-04-28T10:30:00Z"
}
```

Untuk memastikan event tidak hilang walau Kafka sempat down sesaat, gunakan **Outbox Pattern**:

```sql
-- Di DB masing-masing payment service, tambahkan tabel outbox:
CREATE TABLE outbox (
    id          BIGINT PRIMARY KEY AUTO_INCREMENT,
    event_type  VARCHAR(100),
    payload     JSON,
    published   BOOLEAN DEFAULT FALSE,
    created_at  TIMESTAMP
);
```

Cara kerjanya:
1. Saat transaksi dibuat, INSERT ke tabel transaksi **dan** INSERT ke outbox dalam satu database transaction yang sama
2. Background process membaca outbox → publish ke Kafka → tandai sebagai published
3. Jika Kafka mati sesaat, data aman di outbox, akan di-retry otomatis

---

## "Kalau Core Mati, Gimana?"

Ini pertanyaan yang wajar. Jawabannya: **transaksi tetap berjalan normal**.

### Core tidak di write path

Bayar via OVO? Requestnya langsung ke OVO Service, bukan lewat Core. Core tidak tahu ada request masuk.

### Hierarki komponen berdasarkan kritikalitas

```
🔴 HARUS SELALU UP (langsung pengaruhi pembayaran):
   → Shopee Service + DB Shopee
   → OVO Service + DB OVO
   → DANA Service + DB DANA
   → Kafka

🟡 KRITIS TAPI BISA DEGRADED SESAAT:
   → Redis (untuk routing table — lihat di bawah)

🟢 BOLEH DOWN SESAAT (hanya pengaruhi reporting):
   → Core Service + DB Core
```

### Strategi fallback saat Core down

**Untuk query status transaksi spesifik (real-time, critical):**

Simpan routing table di Redis — sangat ringan dan mudah dibuat highly available:

```
upg_transaction_id "abc-123" → { payment_method: "ovo" }
upg_transaction_id "xyz-456" → { payment_method: "shopee" }
```

Alurnya:
```
Client tanya status "abc-123"
     ↓
Cek Redis → payment_method = "ovo"
     ↓
Tanya OVO Service langsung → dapat status real-time
(Core tidak terlibat sama sekali)
```

**Untuk reporting dan rekonsiliasi (non-critical):**
- Serve dari cache (data boleh stale 5 menit — masih acceptable)
- Atau tampilkan pesan: *"Data laporan mungkin tidak lengkap, sedang dalam proses update"*

**Saat Core hidup kembali:**
- Core otomatis catch-up dari Kafka (Kafka menyimpan semua events)
- Tidak ada data yang hilang karena semua sudah tersimpan di DB masing-masing payment service

---

## Perbandingan Resiliensi

| Skenario | Arsitektur Sekarang | Arsitektur Baru |
|---|---|---|
| Shopee DB mati | OVO/DANA tetap jalan ✅ | OVO/DANA tetap jalan ✅ |
| Core DB mati | *(tidak ada core)* | Write tetap jalan ✅, reporting sementara stale ⚠️ |
| Kafka mati | *(tidak ada Kafka)* | Write tetap jalan ✅, events di-buffer di outbox ✅ |
| Developer buat fitur cross-payment | Sangat susah ❌ | Query satu tabel di Core ✅ |
| ID collision antar payment method | Bisa terjadi ❌ | Tidak mungkin terjadi (UUID global) ✅ |
| Developer baru onboarding | Bingung karena tidak ada standar ❌ | Ada satu ID standar dan satu titik query ✅ |

---

## Rencana Implementasi (Incremental — Tidak Perlu Rewrite Total)

Tidak perlu ubah semua sekaligus. Bisa dilakukan bertahap:

### Tahap 1 — Paling Cepat, Paling Berdampak
> Target: selesai dalam 1-2 sprint

- [ ] Tentukan format `upg_transaction_id` (rekomendasi: UUID v4)
- [ ] Tambahkan kolom `upg_transaction_id` di semua tabel transaksi yang sudah ada
- [ ] Semua transaksi baru wajib menyertakan `upg_transaction_id`
- [ ] Simpan routing table `upg_transaction_id → payment_method` di Redis

**Dampak langsung:** ID collision hilang, status check bisa dilakukan tanpa Core.

### Tahap 2 — Core Service sebagai Read Model
> Target: selesai dalam 1 sprint setelah Tahap 1

- [ ] Buat Core Service dengan schema minimal (lihat di atas)
- [ ] Setup Kafka topic `upg.transaction.*`
- [ ] Setiap payment service publish event ke Kafka saat transaksi dibuat/diupdate
- [ ] Core konsumsi Kafka dan populate DB Core

**Dampak langsung:** Cross-payment query bisa dilakukan, dashboard terpusat bisa dibangun.

### Tahap 3 — Hardening & Outbox Pattern
> Target: setelah Tahap 2 stabil

- [ ] Implementasi Outbox Pattern di setiap payment service
- [ ] Setup Core Service dengan multiple instance (HA)
- [ ] Setup read replica untuk DB Core

**Dampak langsung:** Sistem lebih resilient terhadap Kafka downtime sesaat.

---

## Ringkasan untuk Diskusi Tim

> **Masalah inti:** Bukan karena pakai DB per service, tapi karena tidak ada Global Transaction ID, tidak ada Central Read Model, dan tidak ada Event-Driven bridge antar service.
>
> **Solusi:** Tambahkan tiga komponen — UUID sebagai Global ID, Core Service sebagai Read Model, dan Kafka sebagai Event Bridge. Ketiga komponen ini tidak mengganggu fault isolation yang sudah ada.
>
> **Argumen arsitek sebelumnya soal fault isolation tetap valid.** Kita tidak membuang prinsip itu, kita melengkapinya.

---

## Referensi & Bacaan Lanjutan

- [Database per Service Pattern — microservices.io](https://microservices.io/patterns/data/database-per-service.html)
- [Outbox Pattern — microservices.io](https://microservices.io/patterns/data/transactional-outbox.html)
- [CQRS Pattern — Martin Fowler](https://martinfowler.com/bliki/CQRS.html)
- [Saga Pattern untuk Distributed Transactions](https://microservices.io/patterns/data/saga.html)
