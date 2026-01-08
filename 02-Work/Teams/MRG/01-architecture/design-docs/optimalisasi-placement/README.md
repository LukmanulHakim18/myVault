---
title: Optimalisasi Placement untuk Tim MRG
project: MRG
status: completed
created: '2025-01-05'
tags:
  - mrg
  - placement
  - pickup
  - dropoff
  - autosnap
  - completed
  - design-doc
completed_date: '2025-01-05'
---
# Optimalisasi Placement untuk Tim MRG

**Project**: Drop-off and Pick-up Point Accuracy Improvement  
**Team**: MRG (Meta Reservation Gateway)  
**Status**: ✅ Completed  
**Owner**: Lukmanul Hakim  
**Completed**: 2025-01-05

---

## 📋 Overview

Project ini bertujuan untuk meningkatkan akurasi titik pickup dan drop-off berdasarkan:
- User behavior (frekuensi kunjungan)
- Landmark clustering
- Radius-based snapping

---

## 📁 Dokumentasi

| File | Deskripsi |
|------|-----------|
| [[C4-Autosnap-Design]] | Arsitektur C4 Model untuk AutoSnap system |
| [[Spike-Document]] | Spike document untuk flow dan concern |
| [[Diagrams/get-auto-snap]] | PlantUML diagram untuk Get Flow |
| [[Diagrams/setter-frequency]] | PlantUML diagram untuk Set Frequency Flow |
| [[Diagrams/setter-popular]] | PlantUML diagram untuk Set Popular Place Flow |

---

## 🎯 Goals

1. **Simpan data frekuensi pengguna** untuk optimasi snap
2. **Implementasi auto-snap** berdasarkan frekuensi (>=2 perjalanan)
3. **Prioritaskan radius-based snapping** (residential vs commercial)
4. **Clustering pickup/drop-off** untuk Landmark dengan subplace

---

## 🏗️ Architecture Highlights

### Prioritization Logic
1. **Favorite Place** - Private location manually selected by user
2. **Frequent Place** - Calculated from usage (if visited >= 3 times)

### Key Components
- **Kafka Consumer** - Listen to `endtrip` events
- **PostGIS** - Spatial queries dengan ST_DWithin
- **Redis** - Temporary counter store untuk location usage
- **Read Replica** - Distribute SELECT load

---

## 🔗 Related Links

- [[gmo-integration]] - GMO Landmark Integration
- [[ID-Generator-Index]] - ID Generator System

---

## 🏷️ Tags

#mrg #placement #pickup #dropoff #autosnap #optimization #postgis

---

*Created*: 2025-01-05  
*Last Updated*: 2025-01-05
