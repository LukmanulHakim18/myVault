# Map Integration Project - Architecture Work Summary

> **Role:** Architect Engineer  
> **Domain:** Transportation Platform System  
> **Company:** Bluebird Group  
> **Last Updated:** 2025-01-03

---

## 📋 Table of Contents

1. [Executive Summary](#executive-summary)
2. [Vendor Evaluation & Comparison](#vendor-evaluation--comparison)
3. [Technical Issues & Solutions](#technical-issues--solutions)
4. [Data Analysis & Findings](#data-analysis--findings)
5. [API Access & Integration](#api-access--integration)
6. [Current Status & Next Steps](#current-status--next-steps)

---

## Executive Summary

Sebagai Architect Engineer di Bluebird Group, saya bertanggung jawab atas evaluasi, integrasi, dan optimasi berbagai vendor penyedia layanan peta (mapping services) untuk platform transportasi MyBluebird. Proyek ini mencakup:

- **Evaluasi 5+ vendor mapping** (Google Maps, ID-Tech, Mapbox, NextBillion, TomTom, TollGuru)
- **Analisis data anomali** dari 4,611 sample menjadi 744 data terkelompok
- **Identifikasi dan solusi 12+ issue kritis** terkait routing, ETA, dan tracking
- **Dokumentasi GMO (Google Maps Optimization)** dengan 17 kota dan 11,434 POI
- **Contract structure** untuk integrasi vendor mapping

---

## Vendor Evaluation & Comparison

### 1. Google Maps (Current Primary)

**Strengths:**
- Data POI terlengkap di Indonesia
- Fleet management via ODRD (On-Demand Ride & Delivery)
- Real-time traffic condition dan speed vehicle
- Update berkala via GMCP (Google Maps Contribution Platform)

**Weaknesses:**
- Issue POI tidak akurat (contoh: Gambir Station muncul di Manggarai)
- Reverse geocoding tidak sempurna ("Jalan Tanpa Nama", lokasi salah provinsi)
- Biaya tinggi untuk volume besar

**Access & API:**
```shell
curl --location 'https://maps.googleapis.com/maps/api/geocode/json?latlng=-6.3047054374615765%2C106.84963659275574&key=AIzaSyB_8CyWMjeYyBpczfRWR7Di401E-yjhbrw'
```

---

### 2. ID-Tech (Local Provider)

**Strengths:**
- Vendor lokal Indonesia
- Biaya potensial lebih kompetitif
- Data customizable untuk kebutuhan lokal

**Findings dari Spike Analysis:**
- Masih menemukan plus-code pada alamat (14-21% dari sample)
- Format alamat landmark mirip dengan GMO
- Perlu konfirmasi kenapa plus code masih muncul
- Perlu validasi formatting address untuk landmark dan subplace

**API Documentation:**
- Doc: https://apipoi.idtek.io/api/docs
- Token: `118ad5b6d0774c4fde7b80b1f8593513a6e39981`

**Data Analysis Results:**
| Dataset | Total Data | Plus Code Count | Percentage |
|---------|-----------|-----------------|------------|
| Grouping Sample Actual Pickup | 743 | 163 | 21.94% |
| Grouping Sample Request Pickup | 743 | 142 | 19.11% |
| Popular Dropoff | 100 | 17 | 17.00% |
| Popular Pickup | 100 | 18 | 18.00% |
| Popular Request Dropoff | 99 | 17 | 17.17% |
| Popular Request Pickup | 100 | 14 | 14.00% |

**Data Location:**
- Sharepoint: https://bluebirdgroup365.sharepoint.com/:f:/s/ArchitectandTW/EvynlC9BPNFKtNUtPgWV7q8BY9hBqmw8z0hNqgCmBBCKog?e=UCJUX7
- Datasets analyzed:
  - Data anomali sample Huawei (4,611 → 744 setelah grouping)
  - Popular request dropoff
  - Popular request pickup
  - Popular actual dropoff
  - Popular actual pickup

---

### 3. Mapbox

**Strengths:**
- Mandiri dalam populasi data rute privat
- Mampu menemukan rute baru
- Customizable mapping

**Weaknesses:**
- Tidak memiliki fleet management seperti ODRD
- Data traffic dari crawling driver (bukan real-time traffic feed)

**Evaluation Checklist:**
- ❌ Tools estimation untuk toll calculation (not eligible)
- ✅ Live tracking support
- ✅ Alternative route support
- ❌ Fleet engine

---

### 4. NextBillion

**Strengths:**
- Traffic dan routing akurat untuk pricing & assign driver
- Encoded polyline kompatibel dengan Google
- Customizable berdasarkan kebutuhan
- Alternative routes support
- Route types: fastest, shortest

**Technical Findings:**
- Polyline encoding mechanism sama dengan Google
- Support untuk custom routing requirements

**Open Questions:**
- ⏳ Toll calculation capability (on check)

**TODO:**
- Buat komparasi rute NextBillion vs Google
  - Waktu bersamaan
  - Lat/lon pickup dan dropoff yang sama

---

### 5. TomTom

**Strengths:**
- Feature set kurang lebih sama dengan Google
- **Unique advantage:** Data historical traffic
- Sudah ada kerjasama dengan pemerintah Jawa Barat dan Jawa Tengah

**Open Questions:**
- ❓ Apakah ada data live traffic?
- ❓ Routing custom support (melewati bangunan untuk potong jarak)?
  - Apakah route publish secara global?
  - Jika install on-prem, bagaimana mekanisme sync?
- ❓ Kalkulasi total biaya toll yang dilalui?

**Reference:** https://tollguru.com/toll-calculator-indonesia

---

### 6. TollGuru

**Purpose:**
- Specialized untuk toll calculation
- API return breakdown toll yang dilewati

**Research Items:**
- Dokumentasi ke PPT (PIC: Lukmanul Hakim)
- Compare dengan Google Maps toll calculation
- Cari kombinasi kompensasi untuk data tidak valid

---

## Technical Issues & Solutions

Sebagai architect, saya mengidentifikasi dan mendokumentasikan 12 issue kritis dalam implementasi map services:

### 1. ❌ Lokasi POI Tidak Benar

**Example:** "Gambir Train Station" muncul di Manggarai

**Root Cause:** Data POI Google Maps tidak akurat, koordinat pickup salah

**Impact:** 
- Driver salah jemput
- Fixed price tidak sesuai

**Solution:**
- ✅ **DONE:** Perbaikan via GMCP (Google Maps Contribution Platform)
- 🔄 **TODO:** Tambahkan internal POI correction layer (override DB internal) untuk update lebih cepat tanpa menunggu Google

---

### 2. ❌ Alamat POI Tidak Benar

**Example:** 
- Reverse geocoding: "Jalan Tanpa Nama"
- Lokasi Jakarta tapi hasil "Sumatera Selatan"

**Root Cause:** Google Maps reverse geocode tidak akurat

**Impact:** Customer menolak order karena alamat tidak sesuai

**Solution:**
- Cross-check dengan OpenStreetMap / HERE API
- Simpan alamat hasil validasi manual (mapping `place_id` → alamat benar)
- Database override untuk area bermasalah

---

### 3. ⏱️ ETA vs ATA Gap (Macet Parah)

**Example:** ETA 1 jam, actual travel time 1.5 jam

**Root Cause:** ETA tidak mempertimbangkan traffic ekstrem (jalan stuck total)

**Impact:** Real argo jauh > Fixed price

**Solution:**
- 🔄 **INPROGRESS:** Tambah waiting fee jika kecepatan rata-rata <15 km/jam
- Integrasi data traffic dari HERE / TomTom untuk perhitungan lebih realistis
- Terapkan dynamic pricing adjustment bila gap ETA vs ATA terlalu besar

---

### 4. 🛣️ Beda Rute: Pricing vs Navigasi


**Example:** Perhitungan harga via Kuningan, navigasi lewat Sudirman

**Root Cause:** Engine pricing dan routing menggunakan preferensi berbeda

**Impact:** Real argo ≠ Fixed price

**Solution:**
- ✅ **DONE:** Gunakan overlay routing yang sama di pricing & navigasi
- ✅ **DONE:** Standardisasi ke satu routing preference (fastest + traffic-aware)

---

### 5. 🚧 Rute Tidak Mengenali Private Road

**Example:** Navigasi taxi diarahkan memutar karena private road tidak dikenali

**Root Cause:** Google Maps tidak mengenali private access road

**Impact:** Rute memutar, argo tidak sesuai

**Solution:**
- 📝 **TODO:** Submit private road ke GMCP
- Tambahkan custom private road layer internal dengan data OSM/GraphHopper
- Override hasil Google jika private road terdeteksi di DB internal

---

### 6. 📍 Taksi Bergerak tapi Tracking Tidak

**Example:** Taxi bergerak, tracking di app diam/statis

**Root Cause:** Bug pada SDK ODRD v5

**Impact:** ETA pickup lama, order bisa batal

**Solution:**
- ✅ **DONE:** Upgrade SDK ke v6
- Tambahkan heartbeat GPS driver app → trigger re-sync jika lokasi tidak update

---

### 7. 🎯 Snap to Road Salah → ETA Lama

**Example:** Taxi di jalan biasa tapi di-snap ke jalan layang

**Root Cause:** Algoritma Snap-to-Road salah memilih jalur


**Impact:** ETA pickup lama → order berpotensi batal

**Solution:**
- 🔄 **TBD:** Nonaktifkan Snap-to-Road jika deviasi lokasi > 15m
- Validasi dengan multi-source positioning (GPS + sensor device)
- Prioritaskan jalan setingkat, bukan elevated road

---

### 8. 🔄 Navigasi Taksi Berputar (U-turn Logic Error)

**Example:** Taxi diarahkan u-turn resmi padahal jalan 2 arah bisa langsung balik

**Root Cause:** Routing engine ODRD hanya pakai u-turn resmi

**Impact:** ETA pickup lama → order batal

**Solution:**
- 📝 **TODO:** Ajukan u-turn ke GMCP
- Override rule: jika jalan 2 arah tanpa separator → izinkan u-turn manual

---

### 9. 🐌 Loading Tracking Navigasi Lambat

**Example:** Route muncul garis abu-abu statis, bukan indikator traffic real-time

**Root Cause:** Route ditampilkan statis, tidak real-time

**Impact:** User tidak lihat posisi taxi terbaru

**Solution:**
- 🔄 **INPROGRESS:** Tambah event analytics tracking
- Gunakan real-time streaming (WebSocket/GRPC)
- Render polyline dengan traffic indicator

---

### 10. 📊 Inkonsistensi ETA

**Example:** ETA berubah drastis dalam 2 menit (misal: 5 menit → 20 menit)

**Root Cause:** Fake GPS atau stop tracking driver

**Impact:** User bingung, trust turun

**Solution:**
- ✅ **DONE:** Fake GPS detection
- ✅ **DONE:** Stop tracking otomatis saat end shift
- Tambahkan anomaly detection bila ETA berubah ekstrem

---

### 11. 🗺️ Issue Routing Options

**Example:** Rute MyBB beda dengan Google Maps (hindari jalan HR Rasuna Said)

**Root Cause:** Routing engine tidak traffic-aware

**Impact:** User tidak diarahkan ke rute tercepat

**Solution:**
- ✅ **DONE:** Ganti routing option ke **TRAFFIC_AWARE_OPTIMAL**
- Uji coba A/B testing untuk validasi ETA vs perjalanan real

---

### 12. 🖼️ Halaman Map Blank

**Example:** Area navigasi kosong meski destinasi & driver info muncul

**Root Cause:** Bug rendering layer SDK Google Maps

**Impact:** User tidak bisa melihat tracking

**Solution:**
- ✅ **DONE:** Lapor ke Google
- Tambahkan fallback renderer (Mapbox/OSM)
- Implementasi retry + error logging

---

## Data Analysis & Findings

### GMO (Google Maps Optimization) - POI Data

Total coverage: **17 kota** dengan **11,434 POI**

| City | Mall | Office | Restaurant | Sports | Airport/Transport | Hospital | Hotel | Others | Residence | School | **Total** |
|------|------|--------|------------|--------|------------------|----------|-------|--------|-----------|--------|-----------|
| Jakarta | 228 | 515 | 101 | 70 | 122 | 150 | 248 | 152 | 649 | 144 | **2,379** |
| Bali | 45 | 281 | 421 | 68 | 9 | 51 | 1,660 | 265 | 180 | 50 | **3,030** |
| Surabaya | 36 | 655 | 349 | 46 | 28 | 153 | 207 | 42 | 21 | 915 | **2,452** |
| Bandung | 63 | 139 | 463 | 43 | 41 | 52 | 363 | 623 | 29 | 210 | **2,026** |
| Lombok | 15 | 108 | 37 | 28 | 8 | 42 | 187 | 22 | 6 | 77 | **530** |
| Pekanbaru | 15 | 95 | 43 | 13 | 1 | 22 | 65 | 9 | 75 | 51 | **389** |
| Semarang | 7 | 69 | 86 | 0 | 7 | 18 | 46 | 50 | 81 | 31 | **395** |
| Makassar | 30 | 38 | 48 | 15 | 2 | 33 | 59 | 32 | 1 | 43 | **301** |
| Batam | 14 | 35 | 10 | 22 | 8 | 11 | 51 | 14 | 7 | 15 | **187** |
| Padang | 3 | 53 | 1 | 2 | 1 | 19 | 58 | 0 | 1 | 32 | **170** |
| Yogyakarta | 15 | 4 | 17 | 5 | 9 | 11 | 50 | 31 | 2 | 10 | **154** |
| Manado | 4 | 17 | 19 | 13 | 5 | 6 | 33 | 31 | 0 | 0 | **128** |
| Palembang | 13 | 32 | 5 | 2 | 5 | 11 | 30 | 5 | 2 | 13 | **118** |
| Cilegon | 13 | 35 | 8 | 2 | 4 | 20 | 17 | 0 | 1 | 11 | **111** |
| Medan | 13 | 10 | 2 | 0 | 5 | 27 | 21 | 17 | 3 | 8 | **106** |
| Bangka Belitung | 4 | 8 | 3 | 0 | 3 | 4 | 18 | 6 | 3 | 2 | **51** |
| Balikpapan | 3 | 12 | 0 | 5 | 1 | 6 | 10 | 2 | 0 | 8 | **47** |

**Key Insights:**
- Bali memiliki POI terbanyak (3,030) terutama dari kategori Hotel (1,660)
- Jakarta memiliki residence/apartment terbanyak (649)
- Surabaya unggul di kategori School/College/University (915)
- Coverage merata di seluruh Indonesia untuk operasional Bluebird

---

## API Access & Integration

### GMO API

```shell
curl --location 'mybb-mapsvc.internal.bluebird.id/api/v1/geolocation/autocomplete?latlng=-6.2781473745487535%2C%20106.74667548088193&subplacetype=D&input=residence' \
--header 'Content-Type: application/json' \
--header 'Token: 9muVTpOfSM41N8xJjYLWV04c4T6vdlHl'
```

**Features:**
- Real-time traffic condition
- Vehicle speed data
- Autocomplete dengan filter subplace type

---

### Contract Structure

Data contract untuk integrasi vendor mapping:

```json
{
  "requested": {
    "pickup": {
      "coordinate": {
        "latitude": -6.3242,
        "longitude": 106.3242
      },
      "address": "Jl. jalan 14"
    },
    "dropoff": {
      "coordinate": {
        "latitude": -6.3242,
        "longitude": 106.3242
      },
      "address": "Jl. jalan 14"
    }
  },
  "actual": {
    "pickup": {
      "coordinate": {
        "latitude": -6.3242,
        "longitude": 106.3242
      },
      "address": "Jl. jalan 14"
    },
    "dropoff": {
      "coordinate": {
        "latitude": -6.3242,
        "longitude": 106.3242
      },
      "address": "Jl. jalan 14"
    }
  },
  "vehicle_info": {},
  "customer_info": {
    "internal_id": "BBID123456",
    "name": "erik",
    "phone": "34324322",
    "email": "34l2mlk2m3@mail.bb"
  },
  "state": 7
}
```

**Contract Fields:**
- `requested`: Koordinat & alamat yang diminta customer (dari input user)
- `actual`: Koordinat & alamat aktual perjalanan (dari GPS tracking)
- `vehicle_info`: Informasi kendaraan (type, plate, etc)
- `customer_info`: Data customer dengan internal_id Bluebird
- `state`: Status order (1-7, final state = 7)

---

## Current Status & Next Steps

### ✅ Completed (DONE)

1. **GMO Data Collection** - 17 kota, 11,434 POI
2. **ID-Tech Spike Analysis** - 4,611 sample data analyzed
3. **SDK Upgrade** - ODRD v5 → v6 untuk fix tracking issue
4. **Routing Standardization** - Unified pricing & navigation routing
5. **Fake GPS Detection** - Implemented and deployed
6. **Google Maps Bug Report** - Map blank issue reported
7. **Routing Option Optimization** - Changed to TRAFFIC_AWARE_OPTIMAL

### 🔄 In Progress (INPROGRESS)

1. **Waiting Fee Implementation** - Untuk kondisi traffic ekstrem (<15 km/jam)
2. **Event Analytics Tracking** - Untuk monitor loading performance
3. **ID-Tech Address Format Validation** - Konfirmasi plus-code dan landmark formatting

### 📝 Backlog (TODO)

1. **Internal POI Correction Layer** - Override database untuk update cepat
2. **Private Road Submission** - Submit ke GMCP
3. **Custom Private Road Layer** - Integrasi OSM/GraphHopper
4. **U-turn Rule Override** - Untuk jalan 2 arah tanpa separator
5. **Fallback Renderer** - Mapbox/OSM sebagai backup Google Maps
6. **NextBillion Route Comparison** - Benchmark dengan Google Maps
7. **TollGuru Integration Analysis** - Compare toll calculation
8. **TomTom Evaluation** - Live traffic, custom routing, on-prem sync

### 🔄 Decision Making (TBD)

1. **Snap-to-Road Deactivation** - Threshold 15m deviasi
2. **Multi-vendor Strategy** - Primary vs fallback vendor selection
3. **Cost-Benefit Analysis** - Google vs Local vendors (ID-Tech, etc)

---

## Architecture Principles Applied

### 1. **Vendor Agnostic Design**
- Contract-based integration memungkinkan switch vendor tanpa major refactoring
- Abstraction layer untuk multiple map providers

### 2. **Data Quality & Validation**
- Multi-source verification (Google, OSM, HERE)
- Internal override mechanism untuk quick fixes
- Statistical analysis untuk anomaly detection

### 3. **Performance Optimization**
- Real-time streaming vs polling
- SDK upgrade untuk better performance
- Traffic-aware routing untuk accurate ETA

### 4. **Monitoring & Analytics**
- Event tracking untuk identify bottlenecks
- Fake GPS detection untuk data integrity
- A/B testing untuk routing optimization

### 5. **Cost Optimization**
- Evaluasi multiple vendors untuk cost comparison
- Hybrid approach: Google primary + local backup
- Volume-based pricing strategy

---

## Technical Stack & Tools

**Primary Technologies:**
- **Backend:** Go (Golang) - Microservices architecture
- **Map Services:** Google Maps Platform, ODRD v6
- **Data Processing:** Python untuk data analysis
- **Documentation:** Markdown, Obsidian

**Integration Points:**
- GMO API (Internal Bluebird)
- Google Maps APIs (Geocoding, Directions, Places)
- ID-Tech POI API
- ODRD Fleet Management SDK

**Analysis Tools:**
- SharePoint untuk data sharing & collaboration
- Excel/CSV untuk data aggregation
- Postman/cURL untuk API testing

---

## Key Metrics & KPIs

### Accuracy Metrics
- **POI Accuracy:** Target 95%+ (Current: monitoring via GMCP)
- **Address Quality:** Target 90%+ valid addresses
- **Plus-code Reduction:** Current 14-21%, Target <10%

### Performance Metrics
- **ETA Accuracy:** Gap ATA vs ETA <20%
- **Routing Consistency:** Pricing route = Navigation route 100%
- **Tracking Update Latency:** <2 seconds

### Business Impact
- **Order Cancellation Rate:** Reduced by fixing ETA issues
- **Customer Satisfaction:** Improved with accurate pickup location
- **Driver Efficiency:** Optimized routing reduces fuel & time

---

## Lessons Learned

### 1. **Data Quality is Critical**
- POI accuracy directly impacts driver efficiency and customer satisfaction
- Regular GMCP contributions maintain data quality
- Internal override layer provides agility for quick fixes

### 2. **Multi-vendor Strategy is Essential**
- No single vendor perfect for all use cases
- Backup vendors prevent single point of failure
- Cost optimization through vendor comparison

### 3. **Real-world Testing Matters**
- Spike analysis revealed issues not visible in vendor demos
- Statistical analysis (4,611 samples) provided concrete evidence
- A/B testing validates architectural decisions

### 4. **SDK Versions Matter**
- ODRD v5 → v6 fixed critical tracking issues
- Regular updates prevent technical debt
- Version compatibility testing is mandatory

### 5. **Traffic-aware Routing is Non-negotiable**
- Indonesia traffic conditions highly variable
- Static routing causes pricing mismatches
- TRAFFIC_AWARE_OPTIMAL reduces ETA vs ATA gap

---

## Risk Management

### Technical Risks

| Risk | Impact | Probability | Mitigation |
|------|--------|-------------|------------|
| Google Maps API downtime | High | Low | Implement fallback to Mapbox/OSM |
| SDK breaking changes | Medium | Medium | Version pinning + thorough testing |
| Data quality degradation | High | Medium | Regular GMCP monitoring + internal override |
| Third-party API rate limits | Medium | Low | Caching + request optimization |

### Business Risks

| Risk | Impact | Probability | Mitigation |
|------|--------|-------------|------------|
| Vendor price increase | High | Medium | Multi-vendor strategy + negotiation |
| Competitor better routing | Medium | Medium | Continuous benchmarking |
| Customer trust loss | High | Low | Transparent communication + quick fixes |
| Regulatory compliance | Medium | Low | Regular audit + legal review |

---

## Collaboration & Stakeholders

### Internal Teams
- **Architecture Team:** System design & vendor evaluation
- **Data Team:** Sample data provision & analysis
- **Mobile Team:** SDK integration & UI/UX
- **Backend Team:** API integration & microservices
- **QA Team:** Testing routing accuracy & performance

### External Partners
- **Google Maps:** GMCP contributions, support tickets
- **ID-Tech:** API access, data validation
- **NextBillion:** Proof of concept, evaluation
- **TomTom:** Demo & capability assessment

### Documentation & Knowledge Sharing
- **SharePoint:** https://bluebirdgroup365.sharepoint.com/:f:/s/ArchitectandTW/EvynlC9BPNFKtNUtPgWV7q8BY9hBqmw8z0hNqgCmBBCKog?e=UCJUX7
- **Obsidian:** This documentation + technical notes
- **PPT Presentations:** Vendor comparisons & recommendations

---

## References & Resources

### Official Documentation
- **Google Maps Platform:** https://developers.google.com/maps
- **ID-Tech POI API:** https://apipoi.idtek.io/api/docs
- **ODRD Documentation:** (Internal Bluebird)
- **TollGuru Calculator:** https://tollguru.com/toll-calculator-indonesia

### Internal Resources
- Architecture folder: `/Desktop/document/mybluebird/Architech/map/`
- Data samples: SharePoint (link above)
- API tokens: Secured in team vault

### Related Work
- [[ID-Generator]] - ID generation system for booking & orders
- [[Notification-System]] - Multi-channel notification strategy
- [[Transportation-Domain]] - Domain knowledge & business logic

---

## Appendix

### Issue Summary Classification

**By Category:**
- **Data Quality Issues (1, 2, 5):** POI accuracy, address quality, private roads
- **Routing & ETA Issues (3, 4, 7, 8, 10, 11):** Traffic awareness, consistency, optimization
- **Technical Issues (6, 9, 12):** SDK bugs, rendering, performance

**By Priority:**
- **Critical (Impact > 80%):** Issues 1, 2, 3, 4, 10
- **High (Impact 50-80%):** Issues 5, 6, 11
- **Medium (Impact 20-50%):** Issues 7, 8, 9, 12

**By Status:**
- **DONE:** Issues 1, 4, 6, 10, 11, 12 (6 issues)
- **INPROGRESS:** Issues 3, 9 (2 issues)
- **TODO:** Issues 2, 5, 8 (3 issues)
- **TBD:** Issue 7 (1 issue)

---

## Contact & Ownership

**Architect Owner:** Lukmanul Hakim  
**Role:** Architect Engineer  
**Domain:** Transportation Platform Systems  
**Company:** Bluebird Group  
**Project:** Map Integration & Optimization  
**Last Updated:** January 3, 2025

---

**Tags:** #architecture #maps #transportation #bluebird #vendor-evaluation #routing #ETA #google-maps #id-tech #mapbox #nextbillion #tomtom

**Related Projects:**
- [[Web-Reservasi]]
- [[MyBB-Mobile-App]]
- [[Notification-System]]
- [[ID-Generator]]
- [[Microservices-Architecture]]

---

*This document serves as the single source of truth for Map Integration architectural decisions, vendor evaluations, issue tracking, and implementation status at Bluebird Group.*
