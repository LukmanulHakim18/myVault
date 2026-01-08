# Pickup & Dropoff Point Accuracy Improvement

> **Feature Name**: Auto-Snap Point Accuracy  
> **Team**: MRG (Meta Reservation Gateway)  
> **Service**: Booking Service  
> **Status**: Implemented  
> **Last Updated**: 2025-01-03

## 📋 Overview

Feature untuk meningkatkan akurasi pickup dan dropoff point dengan menggunakan auto-snap ke lokasi yang paling sering digunakan atau paling populer di area tersebut.

### Problem Statement

- User sering memasukkan koordinat yang tidak akurat
- Pickup/dropoff point tidak tepat di pinggir jalan
- Driver kesulitan menemukan lokasi yang tepat
- Waktu pickup lebih lama karena koordinat tidak presisi

### Solution

Implementasi auto-snap system yang secara otomatis menyesuaikan koordinat user ke:
1. **Frequency-based**: Lokasi yang paling sering digunakan di area tersebut
2. **Popularity-based**: Lokasi yang populer (berdasarkan agregasi data)

---

## 📚 Documentation

### Architecture & Design
- [C4 Design](c4-design.md) - Component architecture dan flow
- [Spike Analysis](spike-analysis.md) - Technical investigation dan findings

### Diagrams
- [Auto-Snap Sequence](diagrams/autosnap-sequence.plantuml) - Sequence diagram untuk auto-snap flow
- [Use Case: Frequency](diagrams/usecase-frequency.plantuml) - Frequency-based snap use case
- [Use Case: Popular](diagrams/usecase-popular.plantuml) - Popularity-based snap use case

---

## 🎯 Key Features

### 1. Frequency-Based Auto-Snap
Snap ke lokasi yang paling sering digunakan dalam radius tertentu.

**Use Case:**
- User set pickup point
- System cari koordinat dalam radius X meter
- Snap ke koordinat dengan frekuensi tertinggi
- User konfirmasi atau adjust manual

### 2. Popularity-Based Auto-Snap
Snap ke lokasi populer (POI, landmarks) di sekitar area.

**Use Case:**
- User set pickup/dropoff point
- System identifikasi POI terdekat
- Snap ke POI populer jika dalam threshold
- User konfirmasi atau adjust manual

---

## 🏗️ Architecture

### Components
1. **Booking Service** - Handle booking request
2. **Auto-Snap Engine** - Core logic untuk snap calculation
3. **Location Database** - Store historical location data
4. **POI Service** - Point of Interest data
5. **Map Service** - Geocoding & reverse geocoding

### Data Flow
```
User Input → Booking Service → Auto-Snap Engine
                                    ↓
                            Check Frequency DB
                            Check POI DB
                                    ↓
                            Calculate Best Snap
                                    ↓
                            Return Adjusted Coordinate
```

---

## 📊 Performance Metrics

### Target Metrics
- **Snap Accuracy**: > 95%
- **Response Time**: < 200ms
- **User Acceptance Rate**: > 80%

### Success Criteria
- ✅ Reduce incorrect pickup point by 50%
- ✅ Improve driver satisfaction score
- ✅ Reduce average pickup time by 2-3 minutes
- ✅ User acceptance rate > 80%

---

## 🚀 Implementation Status

### ✅ Completed
- [x] Spike analysis & feasibility study
- [x] C4 architecture design
- [x] Use case documentation
- [x] Sequence diagrams
- [x] Core algorithm implementation
- [x] Integration with Booking Service

### 🔄 In Progress
- [ ] Performance optimization
- [ ] A/B testing setup
- [ ] User feedback collection

### 📋 Planned
- [ ] Machine learning model for prediction
- [ ] Real-time traffic integration
- [ ] Advanced POI database

---

## 🔗 Related Services

- **Booking Service**: Primary service using auto-snap
- **Location Service**: Historical location data
- **Map Integration**: Geocoding & reverse geocoding
- **Analytics Service**: Usage metrics & performance tracking

---

## 📝 Notes

### Learnings
- Frequency-based snap works better in residential areas
- Popularity-based snap works better in commercial areas
- Threshold tuning is critical for user acceptance
- Need balance between automation and user control

### Future Improvements
- ML-based prediction for optimal snap point
- Context-aware snapping (time of day, day of week)
- Integration with real-time traffic data
- User preference learning

---

## 🏷️ Tags

#mrg #booking-service #auto-snap #location-accuracy #feature #architecture #design-docs

---

**Related Documents:**
- [Booking Service Overview](../../02-services/booking-service/README.md)
- [MRG Architecture](../README.md)
- [Map Integration Assessment](../../../../Shared/integration/map-services/00-assessment/README.md)