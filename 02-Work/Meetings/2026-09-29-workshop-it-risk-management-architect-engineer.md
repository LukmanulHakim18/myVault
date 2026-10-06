---
title: Workshop IT Risk Management - Upstream/Downstream & Impact Analysis
date: '2026-09-29'
type: workshop
tags:
  - workshop
  - it-risk-management
  - architecture
  - risk
  - data-flow
  - impact-analysis
status: in-progress
---
# Workshop IT Risk Management — Upstream/Downstream & Impact Analysis

## 📋 Info
| Item | Detail |
|------|--------|
| Tanggal | 2026-09-29 |
| Waktu | |
| Lokasi / Media | |
| Penyelenggara | |
| Pembicara / Fasilitator | |
| Peserta | |

## 🎯 Ekspektasi Workshop
Untuk setiap tim/layanan, siapkan:
1. **Upstream data** — dari mana data masuk
2. **Downstream data** — ke mana data keluar
3. **Analisis dampak** — jika bermasalah, tim mana saja yang terdampak
4. **Proses internal** — proses apa saja yang terjadi di dalam layanan
5. **Diskusi terbuka** — *drill-down* alur dan data

---

## 🗺️ Peta Dependensi Antar Tim
```mermaid
flowchart LR
    subgraph UP[Upstream]
        U1[Sumber 1]
        U2[Sumber 2]
    end

    subgraph SQ[Squad]
        PE[Price Engine]
        PR[Promo]
        UPG[UPG - Payment Gateway]
    end

    subgraph DOWN[Downstream]
        D1[Konsumen 1]
        D2[Konsumen 2]
    end

    U1 --> PE
    U2 --> PR
    PE --> PR
    PR --> UPG
    UPG --> D1
    UPG --> D2
```
> Sesuaikan node dan arah panah dengan kondisi aktual.

---

## 1️⃣ Price Engine

### Upstream Data
| Sumber (Tim/Service) | Data | Protokol (gRPC/REST/Kafka/RabbitMQ) | Pola (sync/async) | Frekuensi / Volume |
|----------------------|------|-------------------------------------|-------------------|--------------------|
| | | | | |

### Downstream Data
| Tujuan (Tim/Service) | Data | Protokol | Pola | Kritikalitas |
|----------------------|------|----------|------|--------------|
| | | | | |

### Proses Internal
```mermaid
flowchart TD
    A[Terima request] --> B[Validasi]
    B --> C[Proses 1]
    C --> D[Proses 2]
    D --> E[Response / Publish event]
```
- Data store (PostgreSQL/Redis): 
- Konfigurasi / rule: 
- Ketergantungan eksternal: 

### Analisis Dampak
| Skenario Gangguan | Tim Terdampak | Dampak Bisnis | Severity | Fallback / Mitigasi |
|-------------------|---------------|---------------|----------|---------------------|
| Service down | | | | |
| Latensi tinggi | | | | |
| Data tidak valid | | | | |
| Dependensi upstream gagal | | | | |

---

## 2️⃣ UPG (Universal Payment Gateway)

### Upstream Data
| Sumber (Tim/Service) | Data | Protokol (gRPC/REST/Kafka/RabbitMQ) | Pola (sync/async) | Frekuensi / Volume |
|----------------------|------|-------------------------------------|-------------------|--------------------|
| | | | | |

### Downstream Data
| Tujuan (Tim/Service/Vendor) | Data | Protokol | Pola | Kritikalitas |
|-----------------------------|------|----------|------|--------------|
| | | | | |

### Proses Internal
```mermaid
flowchart TD
    A[Terima request pembayaran] --> B[Validasi]
    B --> C[Proses ke vendor / PSP]
    C --> D[Callback / Webhook]
    D --> E[Update status & ledger]
    E --> F[Publish event]
```
- Data store (PostgreSQL/Redis): 
- Integrasi vendor/PSP: 
- Rekonsiliasi: 

### Analisis Dampak
| Skenario Gangguan | Tim Terdampak | Dampak Bisnis | Severity | Fallback / Mitigasi |
|-------------------|---------------|---------------|----------|---------------------|
| Service down | | | | |
| Vendor/PSP gagal | | | | |
| Webhook tertunda/hilang | | | | |
| Inkonsistensi ledger | | | | |

---

## 3️⃣ Promo

### Upstream Data
| Sumber (Tim/Service) | Data | Protokol (gRPC/REST/Kafka/RabbitMQ) | Pola (sync/async) | Frekuensi / Volume |
|----------------------|------|-------------------------------------|-------------------|--------------------|
| | | | | |

### Downstream Data
| Tujuan (Tim/Service) | Data | Protokol | Pola | Kritikalitas |
|----------------------|------|----------|------|--------------|
| | | | | |

### Proses Internal
```mermaid
flowchart TD
    A[Terima request] --> B[Validasi eligibilitas]
    B --> C[Kalkulasi potongan]
    C --> D[Reservasi / klaim kuota]
    D --> E[Response / Publish event]
```
- Data store (PostgreSQL/Redis): 
- Pengelolaan kuota & budget: 
- Ketergantungan eksternal: 

### Analisis Dampak
| Skenario Gangguan | Tim Terdampak | Dampak Bisnis | Severity | Fallback / Mitigasi |
|-------------------|---------------|---------------|----------|---------------------|
| Service down | | | | |
| Kuota tidak sinkron | | | | |
| Promo salah terapkan | | | | |
| Dependensi upstream gagal | | | | |

---

## 🔍 Diskusi Terbuka — Drill-down Alur & Data
### Pertanyaan Pemandu
- Titik tunggal kegagalan (*single point of failure*) ada di mana?
- Alur mana yang *sync* dan berpotensi *cascading failure*?
- Bagaimana penanganan *retry*, *idempotency*, dan *timeout*?
- Data apa yang bersifat sensitif (PII/finansial) dan bagaimana perlindungannya?
- Bagaimana *observability* (Grafana/Kibana) untuk mendeteksi gangguan?
- Siapa *owner* dan alur eskalasi saat insiden?

### Catatan Diskusi
- 

---

## ⚠️ Risk Register
| ID | Risk | Tim / Service | Likelihood | Impact | Level | Mitigasi | Owner |
|----|------|---------------|-----------|--------|-------|----------|-------|
| R-01 | | | | | | | |
| R-02 | | | | | | | |
| R-03 | | | | | | | |

## ✅ Action Items
- [ ] Lengkapi upstream/downstream Price Engine
- [ ] Lengkapi upstream/downstream UPG
- [ ] Lengkapi upstream/downstream Promo
- [ ] Validasi analisis dampak dengan masing-masing tim
- [ ] 

## 📎 Referensi
- Materi / slide: 
- Link terkait: 
