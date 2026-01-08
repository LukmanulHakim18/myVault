---
title: Spike Document - Optimalisasi Placement
parent: Optimalisasi Placement MRG
type: spike
tags:
  - spike
  - research
  - autosnap
---
# Spike Document: Optimalisasi Placement

**Parent**: [[README|Optimalisasi Placement untuk Tim MRG]]

---

## 🎯 Spike: Drop off and Pick up Point Accuracy Improvement `FLOW`

### Simpan data frekuensi pengguna untuk optimasi snap

Untuk menyimpan data bersih dan siap pakai, setiap order complete melakukan langkah-langkah berikut:

1. **Mendengarkan event** order complete dari BBD (via Kafka)
2. **Cek ke `frequency place`** user table dan `popular public place`
3. **Jika tidak ada**, cek apakah point tersebut masuk ke dalam landmark
4. **Simpan data** ke table `frequency place` jika memenuhi syarat ketentuan
5. **Simpan data** ke table `popular public place` jika memenuhi syarat ketentuan. Jika sudah ada, hanya tambahkan increment counter pada kolom `pickedUp` atau `droppedOff`

### Sequence Diagram

Lihat: [[Diagrams/setter-frequency]] dan [[Diagrams/setter-popular]]

---

## 🔄 Implementasi Auto-Snap Berdasarkan Frekuensi

**Threshold**: >= 2 perjalanan

### Endpoint Baru

Buat endpoint baru untuk rekomendasi frequency place yang mengembalikan dua data:
- `private_frequency_place` - Lokasi frekuensi personal user
- `popular_public_place` - Lokasi populer global

**Tujuan**: Agar tidak mengganggu current flow

### Response Handling

Mobile akan menerima data dan langsung menerapkan berdasarkan prioritas:
1. Favorite Place
2. Frequent Place (Personal)
3. Popular Place (Global)

Lihat diagram: [[Diagrams/get-auto-snap]]

---

## 📍 Prioritaskan Radius-Based Snapping

| Zone Type | Radius | Use Case |
|-----------|--------|----------|
| Residential | 50m | Rumah, apartemen |
| Commercial | 100m | Mall, perkantoran |
| Landmark | Varies | Airport, stasiun |

---

## 🏢 Clustering Pickup/Drop-off untuk Landmark dengan Subplace

Contoh:
- **Landmark**: Grand Indonesia
  - **Subplace**: East Mall Entrance
  - **Subplace**: West Mall Entrance
  - **Subplace**: Basement Parking

---

# Spike: Drop off and Pick up Point Accuracy Improvement `CONCERN`

## ⚠️ Concerns

### 1. Penambahan Beban Query SELECT

**Problem**: Penambahan beban query SELECT ke table `order` dengan penambahan setup query

**Solusi**: 
- Ambil data dari **read replica** database
- Gunakan **caching layer** untuk frequent queries
- Implementasi **pagination** untuk large dataset

### 2. PostGIS Query Performance

**Problem**: ST_DWithin queries bisa expensive

**Solusi**:
- Pastikan **GIST index** pada geometry fields
- Gunakan **bounding box** filter sebelum spatial query
- Monitor **query execution plan**

### 3. Redis Memory Growth

**Problem**: Counter dan geospatial data bisa grow tanpa batas

**Solusi**:
- Implementasi **TTL** untuk temporary data
- Gunakan **capped counters** dengan max limit
- Periodic **cleanup job** untuk expired data

---

## 🏷️ Tags

#spike #research #autosnap #placement #concern

---

*Last Updated*: 2025-01-05
