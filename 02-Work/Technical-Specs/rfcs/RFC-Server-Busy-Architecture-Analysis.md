---
title: 'Architecture Analysis: Server Busy Feature'
author: Lukmanul Hakim
type: analysis
status: draft
created: '2025-01-28'
tags:
  - architecture
  - analysis
  - server-busy
  - benchmark
  - ride-hailing
related:
  - RFC-Server-Busy-Dispatcher-Threshold
---
# Architecture Analysis: Server Busy Feature

**Type**: Supporting Document  
**Related RFC**: [[RFC-Server-Busy-Dispatcher-Threshold]]  
**Purpose**: Analisis arsitektur dan benchmark industri untuk mendukung keputusan desain

---

## 1. Industry Benchmark: Grab, Gojek, Uber

### 1.1 Bagaimana Pemain Besar Menangani High Demand?

Perusahaan ride-hailing besar **TIDAK langsung reject order**. Mereka menggunakan kombinasi strategi:

| Strategy | Bagaimana Kerja | Tujuan |
|----------|-----------------|--------|
| **Surge Pricing** | Harga naik saat demand tinggi | Reduce demand secara natural, attract more drivers |
| **Transparent ETA** | "Estimasi tunggu 15 menit, lanjutkan?" | Customer decide sendiri |
| **Driver Incentives** | Bonus untuk driver yang ke area busy | Increase supply |
| **Heat Maps** | Real-time demand visualization untuk driver | Proactive driver positioning |
| **Demand Prediction** | ML-based forecasting | Anticipate, bukan react |

### 1.2 Key Insight

> **Industri lebih ke market mechanism dan transparency, bukan hard rejection.**

Alasan:
- Hard rejection = customer pindah ke competitor
- Transparency = customer informed, bisa decide sendiri
- Supply-side intervention = solve root cause

---

## 2. Concern dengan Pendekatan Hard Rejection

| Concern | Masalah | Impact |
|---------|---------|--------|
| **Hard rejection** | Customer langsung ditolak | Bisa pindah ke competitor (Grab/Gojek) |
| **Threshold static** | Tidak adjust berdasarkan supply | Bisa reject padahal driver available |
| **Demand-side only** | Hanya block order masuk | Tidak solve root cause (kurang driver) |
| **Binary decision** | Ya/Tidak, tidak ada middle ground | Poor customer experience |

---

## 3. Alternative Approaches

### 3.1 Option A: Queue-Based dengan Transparency (RECOMMENDED)

**Konsep**: Alih-alih reject, beri pilihan ke customer dengan informasi lengkap.

**Customer UI:**
```
┌─────────────────────────────────────────────────────────────────┐
│                                                                 │
│   ⚠️ Area ini sedang ramai                                      │
│                                                                 │
│   📊 Antrian saat ini: 45 order                                │
│   ⏱️ Estimasi waktu tunggu: 10-15 menit                        │
│                                                                 │
│   ┌─────────────────┐  ┌─────────────────┐                     │
│   │  Tetap Pesan    │  │  Coba Nanti     │                     │
│   └─────────────────┘  └─────────────────┘                     │
│                                                                 │
│   * Member Elite/Voyager: Estimasi 5-8 menit (prioritas)       │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

**Keuntungan:**
- Customer informed, bisa decide sendiri
- Tidak kehilangan customer ke competitor
- Loyal customer tetap dapat prioritas
- Data-driven decision oleh customer

**API Response:**
```json
{
    "status": "AVAILABLE_WITH_WARNING",
    "area_id": "JKT-PUSAT",
    "current_waiting": 45,
    "estimated_wait_minutes": 15,
    "message": "Area ramai, estimasi tunggu 10-15 menit",
    "loyalty_benefit": "Member Elite: Estimasi 5-8 menit",
    "options": ["PROCEED", "CANCEL"]
}
```

### 3.2 Option B: Dynamic Threshold (Supply-Demand Ratio)

**Konsep**: Threshold bukan fixed number, tapi berdasarkan ratio waiting orders vs available drivers.

```go
type DynamicThreshold struct {
    AreaID              string
    
    // Ratio-based threshold
    MaxWaitingPerDriver float64  // e.g., 3.0 = max 3 waiting orders per available driver
    
    // Real-time data
    CurrentWaitingOrders int
    AvailableDrivers     int
    
    // Calculated
    CurrentRatio         float64  // waiting / drivers
    IsOverloaded         bool     // ratio > threshold
}

func (t *DynamicThreshold) Calculate() {
    if t.AvailableDrivers == 0 {
        t.IsOverloaded = true
        return
    }
    t.CurrentRatio = float64(t.CurrentWaitingOrders) / float64(t.AvailableDrivers)
    t.IsOverloaded = t.CurrentRatio > t.MaxWaitingPerDriver
}
```

**Contoh dengan Ratio 3.0:**

| Drivers Available | Max Waiting Orders | Alasan |
|-------------------|-------------------|--------|
| 10 drivers | 30 orders | 10 × 3.0 = 30 |
| 5 drivers | 15 orders | 5 × 3.0 = 15 |
| 20 drivers | 60 orders | 20 × 3.0 = 60 |

**Keuntungan:**
- Threshold otomatis adjust berdasarkan supply
- Lebih fair - tidak reject kalau driver sebenarnya cukup
- Self-balancing system

**Kebutuhan Tambahan:**
- Real-time driver availability data dari BBD
- Integration dengan driver tracking system

### 3.3 Option C: Zone-Based (Soft + Hard Limit)

**Konsep**: Multiple zone dengan behavior berbeda, bukan binary yes/no.

```
                    Waiting Orders
    0          50         100        150         200
    |__________|__________|__________|__________|
    
    [  GREEN   ][  YELLOW  ][   RED   ][ BLOCKED ]
```

| Zone | Waiting Count | Behavior |
|------|---------------|----------|
| **GREEN** | 0-50 | Normal, semua bisa order |
| **YELLOW** | 50-100 | Warning, show ETA, customer decide |
| **RED** | 100-150 | Priority only (Voyager/Elite) |
| **BLOCKED** | >150 | Hard stop, semua ditolak |

**Implementation:**
```go
type ZoneConfig struct {
    GreenMax   int  // 50
    YellowMax  int  // 100
    RedMax     int  // 150
    // Above RedMax = BLOCKED
}

func CheckAvailability(waitingCount int, loyaltyLevel string, config ZoneConfig) Response {
    switch {
    case waitingCount <= config.GreenMax:
        return Response{Status: "AVAILABLE", Zone: "GREEN"}
        
    case waitingCount <= config.YellowMax:
        return Response{
            Status: "AVAILABLE_WITH_WARNING",
            Zone: "YELLOW",
            EstimatedWait: calculateETA(waitingCount),
            Message: "Area ramai, estimasi tunggu 10-15 menit",
        }
        
    case waitingCount <= config.RedMax:
        if loyaltyLevel == "ELITE" || loyaltyLevel == "VOYAGER" {
            return Response{Status: "AVAILABLE_PRIORITY", Zone: "RED"}
        }
        return Response{
            Status: "SUGGEST_WAIT",
            Zone: "RED",
            Message: "Area sangat ramai, coba 15 menit lagi",
            RetryAfter: 900,
        }
        
    default:
        return Response{Status: "BLOCKED", Zone: "BLOCKED"}
    }
}
```

**Keuntungan:**
- Gradual degradation, bukan cliff edge
- Better customer experience
- Hard block hanya di kondisi extreme
- Loyal customer masih bisa order di RED zone

---

## 4. Supply-Side vs Demand-Side

### 4.1 Current RFC Focus

RFC saat ini **100% demand-side** (block incoming orders).

### 4.2 Complete Solution (Industry Standard)

```
┌─────────────────────────────────────────────────────────────────┐
│                    COMPLETE SOLUTION                            │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│   DEMAND SIDE (Current RFC)     │    SUPPLY SIDE (Future)      │
│   ─────────────────────────     │    ──────────────────────    │
│                                 │                               │
│   • Threshold check             │    • Driver notification      │
│   • Queue management            │      "Area X butuh driver!"   │
│   • Loyalty bypass              │                               │
│   • ETA transparency            │    • Incentive/Bonus          │
│                                 │      "Bonus Rp 20k ke Area X" │
│                                 │                               │
│                                 │    • Heat map untuk driver    │
│                                 │                               │
│                                 │    • Predictive positioning   │
│                                 │                               │
└─────────────────────────────────────────────────────────────────┘
```

### 4.3 Supply-Side Interventions (Future Enhancement)

| Intervention | Description | Impact |
|--------------|-------------|--------|
| **Driver Push Notification** | Alert driver: "Area Sudirman butuh 10 driver, bonus Rp 15k" | Attract drivers to busy area |
| **Heat Map** | Visual map showing demand hotspots untuk driver | Proactive positioning |
| **Dynamic Incentive** | Auto-increase bonus saat threshold approaching | Balance supply-demand |
| **Predictive Dispatch** | Pre-position drivers based on historical pattern | Prevent overload before happen |

---

## 5. Pertimbangan Khusus untuk Bluebird

### 5.1 Bluebird Context

| Aspect | Bluebird | Grab/Gojek |
|--------|----------|------------|
| **Pricing** | Fixed | Dynamic (surge) |
| **Brand** | Premium | Mass market |
| **Supply control** | Own fleet | Gig economy |
| **Customer expectation** | High service level | Price-sensitive |

### 5.2 Implikasi untuk Design

1. **Tidak bisa pakai surge pricing** → perlu mekanisme lain untuk manage demand
2. **Premium brand** → hard rejection bisa damage brand perception
3. **Own fleet** → lebih kontrol untuk supply-side intervention
4. **High expectation** → transparency dan communication sangat penting

### 5.3 Rekomendasi untuk Bluebird

| Aspect | Recommendation | Alasan |
|--------|----------------|--------|
| **Rejection approach** | Soft rejection dengan transparency | Jaga customer experience & brand |
| **Threshold type** | Zone-based (GREEN/YELLOW/RED/BLOCKED) | Gradual, bukan cliff edge |
| **Customer communication** | Show ETA, let them decide | Informed decision |
| **Loyalty benefit** | Priority di RED zone, bukan unlimited bypass | Sustainable |
| **Hard block** | Only at BLOCKED zone (>150% threshold) | Last resort |

---

## 6. Decision Framework

### 6.1 Key Questions untuk Stakeholder

| # | Question | Impact |
|---|----------|--------|
| 1 | Apakah Bluebird mau kehilangan customer ke Grab/Gojek karena hard rejection? | Jika tidak → soft rejection |
| 2 | Apakah ada mekanisme untuk increase supply saat busy? | Jika tidak → demand-side hanya "plester" |
| 3 | Berapa toleransi waiting time yang acceptable? | Menentukan threshold config |
| 4 | Apakah driver data (available count per area) accessible? | Menentukan static vs dynamic threshold |
| 5 | Budget untuk driver incentive saat high demand? | Supply-side feasibility |

### 6.2 Recommended Approach by Maturity

| Phase | Approach | Complexity |
|-------|----------|------------|
| **MVP (Now)** | Zone-based dengan transparency | Medium |
| **Phase 2** | Dynamic threshold (supply-demand ratio) | High |
| **Phase 3** | Supply-side intervention (driver notification) | High |
| **Future** | ML-based prediction & proactive positioning | Very High |

---

## 7. Comparison Matrix

| Criteria | Hard Rejection (Current) | Transparency + Choice | Zone-Based | Dynamic Ratio |
|----------|-------------------------|----------------------|------------|---------------|
| **Complexity** | Low | Medium | Medium | High |
| **Customer Experience** | Poor | Good | Good | Best |
| **Churn Risk** | High | Low | Low | Lowest |
| **Implementation Effort** | Low | Medium | Medium | High |
| **Data Requirement** | Waiting count only | + ETA calculation | + Zone config | + Driver availability |
| **Flexibility** | Low | Medium | High | Highest |

---

## 8. Conclusion

### 8.1 Short-term Recommendation

Implementasi **Zone-Based approach** dengan:
- GREEN: Normal flow
- YELLOW: Warning + ETA + customer choice
- RED: Loyalty priority only
- BLOCKED: Hard rejection (extreme cases)

### 8.2 Long-term Vision

Evolve ke **Dynamic Ratio-Based** dengan supply-side intervention:
- Real-time supply-demand balancing
- Driver notification & incentive
- Predictive positioning

### 8.3 Key Success Metrics

| Metric | Target | Purpose |
|--------|--------|---------|
| Customer rejection rate | < 5% | Minimize lost customers |
| Customer informed choice (proceed despite warning) | Track | Understand customer tolerance |
| Threshold breach duration | < 30 min avg | Quick recovery |
| Driver response to busy area notification | > 30% | Supply-side effectiveness |

---

## References

- [[RFC-Server-Busy-Dispatcher-Threshold]] - Main RFC document
- [[2025-01-12-Traffic-Spike-Analysis]] - Incident yang trigger initiative ini
- Uber Engineering Blog: Dynamic Pricing
- Grab Tech Blog: Demand Forecasting
- Gojek Engineering: Driver Allocation System

