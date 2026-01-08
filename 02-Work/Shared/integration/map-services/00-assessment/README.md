# Map Services Vendor Assessment

> **Status**: In Progress  
> **Owner**: Lukmanul Hakim (Architecture Engineer)  
> **Last Updated**: 2025-01-03

## 📋 Assessment Overview

This assessment evaluates map service providers for Bluebird Group's transportation platform (MRG - Meta Reservation Gateway and related services).

### Quick Navigation

#### 📄 Core Documents
- [Requirements](requirements.md) - Business & technical requirements
- [Evaluation Criteria](evaluation-criteria.md) - Scoring matrix & methodology
- [Project Overview](project-overview.md) - Original project documentation

#### 🏢 Vendor Evaluations
- [Google Maps](vendors/google-maps/overview.md)
- [Mapbox](vendors/mapbox/overview.md)
- [NextBillion](vendors/nextbillion/overview.md)
- [TollGuru](vendors/tollguru/overview.md)
- [TomTom](vendors/tomtom/overview.md)

#### 📊 Analysis & Benchmarks
- [Vendor Comparison Matrix](analysis/vendor-comparison-matrix.md)
- [Technical Evaluation](technical-evaluation.md)
- [Benchmark Results](benchmarks/results/)

#### 🎯 Decisions
- [ADR-001: Map Provider Selection](decisions/adr-001-map-provider-selection.md)

#### 🔐 Administrative
- [Access Credentials](access-credentials.md)
- [Contract Details](contract-details.json)
- [Known Issues](known-issues.md)

---

## 🗂️ Previous Research Data

The following data from previous Map Integration work has been preserved:

### Key Metrics
- **Total Cities Covered:** 17
- **Total POI in GMO:** 11,434
- **Issues Identified:** 12
- **Issues DONE:** 6
- **Issues INPROGRESS:** 2
- **Issues TODO:** 3
- **Issues TBD:** 1

### Top 5 Cities by POI Count
1. **Bali:** 3,030 POI
2. **Surabaya:** 2,452 POI
3. **Jakarta:** 2,379 POI
4. **Bandung:** 2,026 POI
5. **Lombok:** 530 POI

### Vendors Evaluated
1. ✅ **Google Maps** - Primary (with ODRD Fleet Management)
2. 🔄 **ID-Tech** - Local provider (spike analysis completed)
3. 🔄 **Mapbox** - Alternative mapping solution
4. 🔄 **NextBillion** - Accurate routing & traffic
5. 🔄 **TomTom** - Historical data advantage
6. 🔄 **TollGuru** - Toll calculation specialist

## 📊 Key Findings Summary

### ID-Tech Analysis (Plus-code Issue)
| Dataset Type | Plus-code % |
|--------------|-------------|
| Actual Pickup | 21.94% |
| Request Pickup | 19.11% |
| Popular Dropoff | 17.00% |
| Popular Pickup | 18.00% |
| Popular Request Dropoff | 17.17% |
| Popular Request Pickup | 14.00% |

### Critical Issues Resolved
- ✅ SDK Upgrade (ODRD v5 → v6) - Fixed tracking
- ✅ Routing Standardization - Fixed pricing vs navigation mismatch
- ✅ Fake GPS Detection - Improved data integrity
- ✅ Traffic-Aware Routing - Better ETA accuracy

### Open Action Items
- 🔄 Waiting fee implementation (<15 km/jam condition)
- 📝 Internal POI correction layer
- 📝 Private road submission to GMCP
- 📝 NextBillion route comparison benchmark
- 📝 TollGuru integration analysis

## 🔧 Technical Stack
- **Backend:** Go (Golang)
- **Map Platform:** Google Maps Platform
- **Fleet Management:** ODRD v6
- **Analysis:** Python, Excel
- **Documentation:** Obsidian, Markdown

## 📞 Contacts & Resources

### APIs & Credentials
```bash
# Google Maps Geocoding
curl 'https://maps.googleapis.com/maps/api/geocode/json?latlng=LAT,LNG&key=KEY'

# ID-Tech POI
# Doc: https://apipoi.idtek.io/api/docs
# Token: 118ad5b6d0774c4fde7b80b1f8593513a6e39981

# GMO Internal API
curl 'mybb-mapsvc.internal.bluebird.id/api/v1/geolocation/autocomplete' \
  -H 'Token: 9muVTpOfSM41N8xJjYLWV04c4T6vdlHl'
```

### SharePoint Data
https://bluebirdgroup365.sharepoint.com/:f:/s/ArchitectandTW/EvynlC9BPNFKtNUtPgWV7q8BY9hBqmw8z0hNqgCmBBCKog?e=UCJUX7

## 🏷️ Tags
#map-integration #architecture #vendor-evaluation #google-maps #routing #ETA #transportation
