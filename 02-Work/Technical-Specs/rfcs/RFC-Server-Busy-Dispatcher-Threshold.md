---
title: 'RFC: Server Busy - Dispatcher Area Threshold'
status: draft
author: Lukmanul Hakim
created: '2025-01-27'
tags:
  - rfc
  - mrg
  - dispatcher
  - backpressure
  - reliability
related:
  - traffic-spike-analysis
  - order-orchestrator
---
# RFC: Server Busy - Dispatcher Area Threshold

**Status**: Draft  
**Author**: Lukmanul Hakim  
**Created**: 2025-01-27  
**Related**: Traffic Spike Analysis 2025-01-12

---

## 1. Problem Statement

Ketika terjadi traffic spike atau kekurangan driver di area tertentu, order terus masuk tanpa mekanisme backpressure. Ini menyebabkan:

1. **Waiting dispatch meningkat** - Order menumpuk tanpa dapat driver
2. **Customer experience buruk** - Customer menunggu lama tanpa kepastian
3. **System overload** - Dispatcher terus melakukan pencarian driver untuk order yang kemungkinan besar tidak akan terpenuhi
4. **Cascading failure** - Seperti incident 12 Jan 2025 (77k concurrent users)

## 2. Proposed Solution

Ada **DUA mekanisme terpisah** yang saling melengkapi:

### 2.1 Mechanism A: Server Busy Check (PRE-ORDER)

Validasi **SEBELUM** order dibuat. Jika area sudah penuh, order ditolak.

```
┌──────────┐     ┌──────────┐     ┌──────────┐     ┌──────────────────┐
│  MyBB    │────▶│   MRG    │────▶│   BBD    │────▶│ Check Waiting    │
│  App     │     │ Pre-Order│     │ (Legacy) │     │ Count vs         │
│          │     │ Validate │     │          │     │ Threshold        │
└──────────┘     └──────────┘     └──────────┘     └──────────────────┘
                      │                                     │
                      │◀────────────────────────────────────┘
                      │     
                 ┌────┴────┐
                 ▼         ▼
           ┌─────────┐ ┌──────────────┐
           │AVAILABLE│ │ SERVER_BUSY  │
           │ → Lanjut│ │ → Tolak      │
           │ create  │ │ order, show  │
           │ order   │ │ error ke user│
           └─────────┘ └──────────────┘
```

**Tujuan**: Mencegah order baru masuk ke area yang sudah overloaded.

### 2.2 Mechanism B: Dispatch Retry (POST-ORDER)

**SETELAH** order dibuat, dispatcher mencari driver dengan mekanisme retry.

```
┌──────────────────────────────────────────────────────────────────────┐
│              DISPATCH LIFECYCLE (setelah order dibuat)               │
├──────────────────────────────────────────────────────────────────────┤
│                                                                      │
│  Order Created                                                       │
│       │                                                              │
│       ▼                                                              │
│  ┌─────────────┐                                                     │
│  │ DISPATCH    │◀──────────────────────────┐                         │
│  │ to BBD      │                           │                         │
│  │ (2 min max) │                           │                         │
│  └──────┬──────┘                           │                         │
│         │                                  │                         │
│   ┌─────┴─────┐                            │                         │
│   ▼           ▼                            │                         │
│ DRIVER     TIMEOUT                         │                         │
│ FOUND      (2 min, no driver)              │                         │
│   │           │                            │                         │
│   │           ▼                            │                         │
│   │    ┌─────────────┐    < 30 min    ┌────┴────┐                    │
│   │    │ Check Total │───────────────▶│ RETRY   │                    │
│   │    │ Lifetime    │                │ Dispatch│                    │
│   │    └──────┬──────┘                └─────────┘                    │
│   │           │ >= 30 min                                            │
│   │           ▼                                                      │
│   │    ┌─────────────┐                                               │
│   │    │ AUTO CANCEL │                                               │
│   │    │ by System   │                                               │
│   ▼    └─────────────┘                                               │
│ ORDER                                                                │
│ ASSIGNED ────────────────────────────────────────────────────────    │
│                                                                      │
│  USER CAN CANCEL ANYTIME DURING THIS 30 MIN WINDOW                   │
└──────────────────────────────────────────────────────────────────────┘
```

**Tujuan**: Memberikan waktu cukup untuk mencari driver, dengan batas waktu jelas.

### 2.3 How They Work Together

```
┌─────────────────────────────────────────────────────────────────────┐
│                        COMPLETE FLOW                                │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│  User Request Order                                                 │
│         │                                                           │
│         ▼                                                           │
│  ┌─────────────────┐                                                │
│  │ MECHANISM A:    │                                                │
│  │ Server Busy     │                                                │
│  │ Check           │                                                │
│  └────────┬────────┘                                                │
│           │                                                         │
│     ┌─────┴─────┐                                                   │
│     ▼           ▼                                                   │
│  AVAILABLE   SERVER_BUSY                                            │
│     │           │                                                   │
│     │           ▼                                                   │
│     │     ┌───────────────┐                                         │
│     │     │ REJECT        │                                         │
│     │     │ "Area sibuk,  │                                         │
│     │     │ coba lagi     │                                         │
│     │     │ nanti"        │                                         │
│     │     └───────────────┘                                         │
│     ▼                                                               │
│  ┌─────────────────┐                                                │
│  │ CREATE ORDER    │                                                │
│  │ (order masuk ke │                                                │
│  │ waiting count)  │                                                │
│  └────────┬────────┘                                                │
│           │                                                         │
│           ▼                                                         │
│  ┌─────────────────┐                                                │
│  │ MECHANISM B:    │                                                │
│  │ Dispatch Retry  │                                                │
│  │ (2min timeout,  │                                                │
│  │ 30min lifetime) │                                                │
│  └────────┬────────┘                                                │
│           │                                                         │
│     ┌─────┴─────┐                                                   │
│     ▼           ▼                                                   │
│  ASSIGNED    CANCELLED                                              │
│  (driver     (30min timeout                                         │
│   found)     atau user cancel)                                      │
│     │           │                                                   │
│     ▼           ▼                                                   │
│  Remove from waiting count                                          │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

### 2.4 Components Affected

| Component | Mechanism A (Server Busy) | Mechanism B (Dispatch Retry) |
|-----------|---------------------------|------------------------------|
| BBD (Legacy) | New threshold check endpoint | Existing dispatch + timeout handling |
| Order Orchestrator | New threshold check (Rent, GB) | Existing dispatch + timeout handling |
| MRG | Call threshold check on pre-order | Pass through dispatch response |
| MyBB | Display "server busy" message | Handle retry loop, show searching UI |

## 3. Technical Design

### 3.1 Scope Clarification

| Aspect | Mechanism A: Server Busy | Mechanism B: Dispatch Retry |
|--------|--------------------------|----------------------------|
| **When** | Pre-order (sebelum order dibuat) | Post-order (setelah order dibuat) |
| **Purpose** | Prevent new orders ke area overloaded | Retry cari driver sampai timeout |
| **Trigger** | Waiting count >= threshold | Dispatch timeout (2 min no driver) |
| **Response** | Reject order, show "area sibuk" | Retry dispatch atau cancel |
| **Scope RFC ini** | ✅ **Focus utama** | ⚠️ Existing behavior, perlu alignment |

> **Note**: Mechanism B (Dispatch Retry dengan 2 min timeout, 30 min lifetime) kemungkinan sudah ada di BBD. RFC ini fokus pada **Mechanism A** dan memastikan kedua mekanisme terintegrasi dengan baik.

---

## 4. Mechanism A: Server Busy Check (Detail)

### 4.1 Area Definition

Area akan didefinisikan berdasarkan **geohash** atau **predefined zones**:

```go
type DispatcherArea struct {
    AreaID      string    `json:"area_id"`      // e.g., "JKT-PUSAT", "CGK-T3"
    Name        string    `json:"name"`
    Geofence    []LatLng  `json:"geofence"`     // polygon coordinates
    Geohash     string    `json:"geohash"`      // alternative: geohash prefix
    ServiceType []string  `json:"service_type"` // ["RIDE", "RENT", "DELIVERY"]
}
```

**Options untuk area identification:**
1. **Geohash-based** - Automatic, scalable, tapi kurang precise di boundary
2. **Predefined zones** - Manual setup, tapi lebih kontrol (recommended untuk fase awal)
3. **Hybrid** - Geohash dengan override untuk area khusus (airport, mall, dll)

### 3.2 Waiting Order Tracking

```go
type WaitingOrderTracker struct {
    AreaID          string        `json:"area_id"`
    ServiceType     string        `json:"service_type"`
    WaitingCount    int64         `json:"waiting_count"`
    Threshold       int64         `json:"threshold"`
    OldestWaitingSince time.Time  `json:"oldest_waiting_since"`
    LastUpdated     time.Time     `json:"last_updated"`
}

// Redis key structure
// waiting:order:{area_id}:{service_type} -> sorted set (order_id, timestamp)
// threshold:config:{area_id}:{service_type} -> hash (threshold, timeout_seconds)
```

**Storage Options:**
| Option | Pros | Cons |
|--------|------|------|
| Redis Sorted Set | Fast, built-in TTL, atomic ops | Memory usage |
| PostgreSQL | Persistence, complex queries | Latency |
| In-memory + Redis | Fastest read | Consistency across instances |

**Recommendation**: Redis Sorted Set dengan periodic sync ke PostgreSQL untuk analytics.

### 3.3 Threshold & Timeout Configuration

```go
type DispatchConfig struct {
    // Area Threshold Settings
    AreaID              string        `json:"area_id"`
    ServiceType         string        `json:"service_type"`
    MaxWaitingOrders    int64         `json:"max_waiting_orders"`    // threshold count
    
    // Timeout Settings
    DispatchTimeoutSec  int64         `json:"dispatch_timeout_sec"`  // 2 minutes = 120
    OrderLifetimeSec    int64         `json:"order_lifetime_sec"`    // 30 minutes = 1800
    RetryIntervalSec    int64         `json:"retry_interval_sec"`    // 15-30 seconds
    BusyRetryIntervalSec int64        `json:"busy_retry_interval_sec"` // 60 seconds (when threshold reached)
    
    // Control Settings  
    CooldownPeriodSec   int64         `json:"cooldown_period_sec"`   // prevent oscillation
    Enabled             bool          `json:"enabled"`
    Priority            int           `json:"priority"`              // for override rules
}
```

**Default Configuration:**
```json
{
    "area_id": "JKT-PUSAT",
    "service_type": "RIDE",
    "max_waiting_orders": 100,
    "dispatch_timeout_sec": 120,
    "order_lifetime_sec": 1800,
    "retry_interval_sec": 15,
    "busy_retry_interval_sec": 60,
    "cooldown_period_sec": 60,
    "enabled": true
}
```

**Timeline Visualization:**
```
Order Created                                              30 min lifetime
    │                                                            │
    ▼                                                            ▼
    ├────┬────┬────┬────┬────┬────┬────┬────┬────┬── ... ──┬────┤
    │ 2m │ 2m │ 2m │ 2m │ 2m │ 2m │ 2m │ 2m │ 2m │        │    │
    │    │    │    │    │    │    │    │    │    │        │    │
    └────┴────┴────┴────┴────┴────┴────┴────┴────┴── ... ──┴────┘
      ▲    ▲    ▲    ▲    ▲    ▲    ▲    ▲    ▲              ▲
      │    │    │    │    │    │    │    │    │              │
    Attempt 1  2    3    4    5    6    7    8    9    ...   ~15 attempts max
              (with 15s retry interval between each)
              
    If any attempt SUCCESS → Order Assigned, exit loop
    If 30 min passed → Auto Cancel (LIFETIME_TIMEOUT)
    If User Cancel → Cancel immediately
```

### 3.4 API Design

#### 3.4.1 Pre-Order Check API (BBD/Order Orchestrator)

**Request:**
```http
POST /api/v1/dispatcher/availability-check
Content-Type: application/json

{
    "pickup_location": {
        "latitude": -6.2088,
        "longitude": 106.8456
    },
    "service_type": "RIDE",
    "channel": "MYBB"
}
```

**Response - Available:**
```json
{
    "status": "AVAILABLE",
    "area_id": "JKT-PUSAT",
    "current_waiting": 45,
    "threshold": 100,
    "estimated_wait_time_sec": 120
}
```

**Response - Server Busy:**
```json
{
    "status": "SERVER_BUSY",
    "error_code": "DISPATCHER_AREA_THRESHOLD_REACHED",
    "area_id": "JKT-PUSAT",
    "current_waiting": 105,
    "threshold": 100,
    "retry_after_sec": 60,
    "message": "Area sedang sibuk, silakan coba beberapa saat lagi"
}
```

#### 3.4.2 Admin API - Threshold Management

```http
# Get current status all areas
GET /api/v1/admin/dispatcher/areas/status

# Update threshold config
PUT /api/v1/admin/dispatcher/areas/{area_id}/threshold
{
    "service_type": "RIDE",
    "max_waiting_orders": 150,
    "waiting_timeout_sec": 300
}

# Emergency override - force busy/available
POST /api/v1/admin/dispatcher/areas/{area_id}/override
{
    "status": "FORCE_BUSY",  // or "FORCE_AVAILABLE", "AUTO"
    "reason": "Banjir di area",
    "expires_at": "2025-01-27T18:00:00Z"
}
```

### 4.4 Waiting Count Logic

Order dihitung sebagai "waiting" selama dalam dispatch retry loop (existing BBD behavior).

```go
// Order masuk waiting count saat:
// - Order created dan mulai dispatch

// Order keluar dari waiting count saat:
// - Driver assigned (SUCCESS)
// - User cancel
// - Lifetime exceeded (30 min timeout)

func (t *ThresholdChecker) GetCurrentWaitingCount(ctx context.Context, areaID string) (int64, error) {
    return t.redis.ZCard(ctx, fmt.Sprintf("waiting:order:%s", areaID))
}

// Saat order created - called by BBD
func (t *ThresholdChecker) AddToWaiting(ctx context.Context, order *Order) error {
    key := fmt.Sprintf("waiting:order:%s", order.AreaID)
    lifetimeExpiry := order.CreatedAt.Add(30 * time.Minute).Unix()
    
    return t.redis.ZAdd(ctx, key, redis.Z{
        Score:  float64(lifetimeExpiry),
        Member: order.OrderID,
    })
}

// Saat order selesai (assigned/cancelled) - called by BBD
func (t *ThresholdChecker) RemoveFromWaiting(ctx context.Context, areaID, orderID string) error {
    key := fmt.Sprintf("waiting:order:%s", areaID)
    return t.redis.ZRem(ctx, key, orderID)
}

// Background cleanup - remove expired orders
func (t *ThresholdChecker) CleanupExpiredOrders(ctx context.Context) error {
    now := time.Now().Unix()
    areas := t.getAllAreas(ctx)
    for _, area := range areas {
        key := fmt.Sprintf("waiting:order:%s", area.AreaID)
        t.redis.ZRemRangeByScore(ctx, key, "-inf", fmt.Sprintf("%d", now))
    }
    return nil
}
```

### 4.5 User Loyalty Bypass

Pengguna dengan loyalitas tinggi dapat **bypass threshold** walaupun area sedang busy.

#### 4.5.1 Loyalty Level Definition

Data loyalitas didapat dari **MRG** (source: Tim Marketing).

```go
type UserLoyalty struct {
    UserID       string `json:"user_id"`
    LoyaltyLevel string `json:"loyalty_level"` // EXPLORER, TRAVELER, VOYAGER, ELITE
    CanBypass    bool   `json:"can_bypass"`    // derived from level
}

// Loyalty levels (dari terendah ke tertinggi)
// 1. EXPLORER  - entry level
// 2. TRAVELER  - mid level
// 3. VOYAGER   - high level
// 4. ELITE     - highest level

// Loyalty level yang bisa bypass threshold (configurable)
var BypassableLevels = map[string]bool{
    "ELITE":    true,
    "VOYAGER":  true,
    "TRAVELER": false,
    "EXPLORER": false,
}
```

#### 4.5.2 Bypass Configuration

```go
type ThresholdBypassConfig struct {
    AreaID              string   `json:"area_id"`
    ServiceType         string   `json:"service_type"`
    BypassEnabled       bool     `json:"bypass_enabled"`
    BypassLoyaltyLevels []string `json:"bypass_loyalty_levels"` // ["ELITE", "VOYAGER"]
    MaxBypassPercentage int      `json:"max_bypass_percentage"` // e.g., 20% extra capacity for loyal users
}
```

**Example Config:**
```json
{
    "area_id": "JKT-PUSAT",
    "service_type": "RIDE",
    "max_waiting_orders": 100,
    "bypass_enabled": true,
    "bypass_loyalty_levels": ["ELITE", "VOYAGER"],
    "max_bypass_percentage": 20
}
```

Artinya:
- Threshold normal: 100 orders
- User ELITE/VOYAGER masih bisa order sampai 120 orders (100 + 20%)
- Setelah 120, semua user (termasuk loyal) akan ditolak

#### 4.5.3 Updated Flow dengan Loyalty Check

```
┌─────────────────────────────────────────────────────────────────────────┐
│                SERVER BUSY CHECK WITH LOYALTY BYPASS                    │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                         │
│  ┌─────────────────┐                                                    │
│  │ User Request    │                                                    │
│  │ NEW Order       │                                                    │
│  └────────┬────────┘                                                    │
│           │                                                             │
│           ▼                                                             │
│  ┌─────────────────┐                                                    │
│  │ Get User        │◀── dari MRG (source: Marketing)                    │
│  │ Loyalty Level   │                                                    │
│  └────────┬────────┘                                                    │
│           │                                                             │
│           ▼                                                             │
│  ┌─────────────────┐     NO     ┌─────────────────┐                     │
│  │ Threshold       │───────────▶│ PROCEED TO      │                     │
│  │ Reached?        │            │ CREATE ORDER    │                     │
│  └────────┬────────┘            └─────────────────┘                     │
│           │ YES                                                         │
│           ▼                                                             │
│  ┌─────────────────┐     YES    ┌─────────────────┐                     │
│  │ User Loyalty    │───────────▶│ Check: Under    │                     │
│  │ = VOYAGER/ELITE?│            │ Max Bypass %?   │                     │
│  └────────┬────────┘            └────────┬────────┘                     │
│           │ NO                           │                              │
│           ▼                        YES   │   NO                         │
│  ┌─────────────────┐            ┌────────┴────────┐                     │
│  │ REJECT ORDER    │            ▼                 ▼                     │
│  │ SERVER_BUSY     │     ┌──────────┐      ┌──────────┐                 │
│  │                 │     │ PROCEED  │      │ REJECT   │                 │
│  │ "Area sibuk"    │     │ (bypass) │      │ (full)   │                 │
│  └─────────────────┘     └──────────┘      └──────────┘                 │
│                                                                         │
└─────────────────────────────────────────────────────────────────────────┘
```

#### 4.5.4 Updated API Request

**Request (dengan loyalty info dari MRG):**
```http
POST /api/v1/dispatcher/availability-check
Content-Type: application/json

{
    "pickup_location": {
        "latitude": -6.2088,
        "longitude": 106.8456
    },
    "service_type": "RIDE",
    "channel": "MYBB",
    "user_id": "USR-123456",
    "loyalty_level": "VOYAGER"
}
```

**Response - Busy but Bypassed (Loyal User):**
```json
{
    "status": "AVAILABLE",
    "area_id": "JKT-PUSAT",
    "current_waiting": 105,
    "threshold": 100,
    "bypassed": true,
    "bypass_reason": "LOYALTY_VOYAGER",
    "message": "Area sedang sibuk, tapi Anda mendapat prioritas sebagai member Voyager"
}
```

**Response - Busy, Bypass Limit Reached:**
```json
{
    "status": "SERVER_BUSY",
    "error_code": "DISPATCHER_AREA_THRESHOLD_REACHED",
    "area_id": "JKT-PUSAT",
    "current_waiting": 125,
    "threshold": 100,
    "max_with_bypass": 120,
    "loyalty_level": "VOYAGER",
    "bypassed": false,
    "message": "Area sedang sangat sibuk, silakan coba beberapa saat lagi"
}
```

#### 4.5.5 Threshold Check Logic dengan Loyalty

```go
func (t *ThresholdChecker) CheckAvailability(ctx context.Context, req *AvailabilityRequest) (*AvailabilityResponse, error) {
    area := t.getAreaByLocation(req.PickupLocation)
    config := t.getThresholdConfig(ctx, area.AreaID, req.ServiceType)
    currentWaiting := t.GetCurrentWaitingCount(ctx, area.AreaID)
    
    // Check 1: Under normal threshold - allow everyone
    if currentWaiting < config.MaxWaitingOrders {
        return &AvailabilityResponse{
            Status: "AVAILABLE",
            AreaID: area.AreaID,
            CurrentWaiting: currentWaiting,
            Threshold: config.MaxWaitingOrders,
        }, nil
    }
    
    // Check 2: Over threshold - check loyalty bypass
    if config.BypassEnabled && t.canBypass(req.LoyaltyLevel, config.BypassLoyaltyLevels) {
        maxWithBypass := config.MaxWaitingOrders + (config.MaxWaitingOrders * config.MaxBypassPercentage / 100)
        
        if currentWaiting < maxWithBypass {
            return &AvailabilityResponse{
                Status:       "AVAILABLE",
                AreaID:       area.AreaID,
                CurrentWaiting: currentWaiting,
                Threshold:    config.MaxWaitingOrders,
                Bypassed:     true,
                BypassReason: fmt.Sprintf("LOYALTY_%s", req.LoyaltyLevel),
            }, nil
        }
    }
    
    // Check 3: Over threshold and no bypass - reject
    return &AvailabilityResponse{
        Status:    "SERVER_BUSY",
        ErrorCode: "DISPATCHER_AREA_THRESHOLD_REACHED",
        AreaID:    area.AreaID,
        CurrentWaiting: currentWaiting,
        Threshold: config.MaxWaitingOrders,
        RetryAfterSec: 60,
    }, nil
}

func (t *ThresholdChecker) canBypass(userLevel string, allowedLevels []string) bool {
    for _, level := range allowedLevels {
        if userLevel == level {
            return true
        }
    }
    return false
}
```

### 4.6 Server Busy Check Flow

**PENTING**: Server Busy Check **HANYA** dilakukan saat **PRE-ORDER** (sebelum order dibuat).

```
┌─────────────────────────────────────────────────────────────────────────┐
│                    SERVER BUSY CHECK (PRE-ORDER ONLY)                   │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                         │
│  ┌─────────────────┐                                                    │
│  │ User Request    │                                                    │
│  │ NEW Order       │                                                    │
│  └────────┬────────┘                                                    │
│           │                                                             │
│           ▼                                                             │
│  ┌─────────────────┐     YES    ┌─────────────────┐                     │
│  │ Check: Area     │───────────▶│ REJECT ORDER    │                     │
│  │ Threshold       │            │ SERVER_BUSY     │                     │
│  │ Reached?        │            │                 │                     │
│  └────────┬────────┘            │ "Area sedang    │                     │
│           │ NO                  │ sibuk, coba     │                     │
│           │                     │ beberapa saat   │                     │
│           │                     │ lagi"           │                     │
│           │                     └─────────────────┘                     │
│           ▼                                                             │
│  ┌─────────────────┐                                                    │
│  │ PROCEED TO      │                                                    │
│  │ CREATE ORDER    │                                                    │
│  │                 │                                                    │
│  │ (masuk ke       │                                                    │
│  │  waiting count) │                                                    │
│  └────────┬────────┘                                                    │
│           │                                                             │
│           ▼                                                             │
│  ┌─────────────────┐                                                    │
│  │ MECHANISM B:    │  ◀── Existing BBD behavior                         │
│  │ Dispatch Retry  │      (2min timeout, 30min lifetime)                │
│  │ ...             │                                                    │
│  └─────────────────┘                                                    │
│                                                                         │
└─────────────────────────────────────────────────────────────────────────┘
```

**Tidak ada Server Busy check saat retry** - Setelah order dibuat, order sudah masuk ke waiting count dan akan terus retry dispatch sampai:
- Driver found (SUCCESS)
- 30 menit lifetime habis (AUTO CANCEL)
- User cancel

---

## 5. Mechanism B: Dispatch Retry (Existing - Out of Scope)

> **Note**: Retry mechanism (2 min timeout, 30 min lifetime) **sudah ada** di BBD. Ini hanya context untuk menjelaskan mengapa ada waiting orders.

**Context**:
- Order yang dibuat akan masuk ke dispatch queue
- BBD akan retry cari driver dengan timeout 2 menit per attempt
- Total lifetime order adalah 30 menit
- Selama retry loop, order dihitung sebagai "waiting"

**Implikasi untuk Server Busy**:
- Waiting count = jumlah order yang sedang dalam retry loop
- Order masuk waiting saat: created
- Order keluar waiting saat: assigned, cancelled (user/timeout)

---

## 6. Implementation Plan

### Phase 1: Foundation (Week 1-2)
- [ ] Define area zones (start dengan major areas: Jakarta zones, Airport, dll)
- [ ] Setup Redis data structure untuk waiting tracking
- [ ] Implement threshold configuration CRUD API
- [ ] Create admin dashboard untuk monitoring

### Phase 2: BBD Integration (Week 2-3)
- [ ] Implement availability check endpoint di BBD
- [ ] Add waiting order tracking (ZADD on order created, ZREM on dispatched/cancelled)
- [ ] Implement auto-cancel worker
- [ ] Integration testing dengan BBD

### Phase 3: MRG Integration (Week 3-4)
- [ ] Add pre-order check call di MRG
- [ ] Handle SERVER_BUSY response
- [ ] Update MyBB API contract
- [ ] E2E testing

### Phase 4: Order Orchestrator (Week 4-5)
- [ ] Implement same logic untuk Rent & Golden Bird
- [ ] Share Redis instance atau separate?
- [ ] Unified monitoring dashboard

### Phase 5: Monitoring & Tuning (Week 5-6)
- [ ] Setup Grafana dashboard untuk threshold metrics
- [ ] Alert configuration (threshold approaching, auto-cancel rate)
- [ ] Load testing dengan simulated busy scenario
- [ ] Fine-tune threshold values based on data

## 7. Monitoring & Alerting

### 5.1 Key Metrics

```
# Prometheus metrics - Threshold
dispatcher_waiting_orders_total{area_id, service_type}
dispatcher_threshold_reached_total{area_id, service_type}
dispatcher_threshold_utilization_ratio{area_id}  # current/max

# Prometheus metrics - Dispatch & Retry
dispatcher_attempt_total{area_id, service_type, result}  # result: assigned, timeout
dispatcher_attempt_duration_seconds{area_id, service_type}
dispatcher_retry_count_total{area_id, service_type}
dispatcher_order_lifetime_seconds{area_id, service_type, result}  # result: assigned, cancelled_timeout, cancelled_user

# Prometheus metrics - Cancel
dispatcher_cancel_total{area_id, service_type, reason}  # reason: user, lifetime_timeout, server_busy
```

### 5.2 Alerts

| Alert | Condition | Severity |
|-------|-----------|----------|
| ThresholdApproaching | utilization > 80% for 5min | Warning |
| ThresholdReached | utilization >= 100% | Critical |
| HighRetryRate | retry_rate > 50% for 10min | Warning |
| HighLifetimeCancel | lifetime_cancel_rate > 10% for 15min | Critical |
| LowAssignmentRate | assignment_rate < 30% for 15min | Critical |
| DispatchLatencyHigh | p95_dispatch_duration > 90s | Warning |

### 5.3 Dashboard Panels

**Real-time Overview:**
- Current waiting orders per area (gauge)
- Threshold utilization % per area (gauge with color coding)
- Orders in retry loop (gauge)

**Dispatch Performance:**
- Dispatch attempts per minute (counter)
- Assignment rate % (calculated: assigned / total attempts)
- Average dispatch duration (histogram)
- Retry distribution (histogram: attempts per order)

**Cancellation Analysis:**
- Cancels by reason (pie chart)
- Lifetime timeout trend (time series)
- User cancel rate by area (bar chart)

**Area Health:**
- Top 5 busiest areas
- Areas currently at threshold
- Average order lifetime per area

## 8. Rollout Strategy

### 6.1 Feature Flag

```go
type ServerBusyFeatureFlag struct {
    Enabled         bool     `json:"enabled"`
    EnabledAreas    []string `json:"enabled_areas"`    // gradual rollout
    EnabledChannels []string `json:"enabled_channels"` // MYBB, WEB, etc
    ShadowMode      bool     `json:"shadow_mode"`      // log only, don't block
}
```

### 6.2 Rollout Phases

1. **Shadow Mode** - Enable tracking & logging, tapi tidak block order
2. **Pilot Areas** - Enable untuk 2-3 area dengan traffic rendah
3. **Gradual Expansion** - Tambah area per minggu
4. **Full Rollout** - Enable untuk semua area

## 9. Open Questions

1. **Area granularity** - Seberapa detail area definition? Per kecamatan? Per kelurahan?

2. **Cross-area handling** - Jika area A busy tapi area B (nearby) available, redirect atau tetap reject?

3. **VIP/Priority handling** - Apakah ada exception untuk customer tertentu yang bisa bypass threshold?

4. **Dynamic threshold** - Apakah threshold perlu adjust otomatis berdasarkan time of day / historical pattern?

5. **Legacy BBD effort** - Seberapa besar effort untuk modify legacy system? Apakah perlu dedicated epic?

6. **Retry interval strategy** - Fixed 15s atau exponential backoff (15s → 30s → 60s)?

7. **UI/UX saat retry** - Bagaimana tampilan di MyBB saat dalam retry loop? Progress bar? Countdown? Estimated time?

8. **Notification** - Apakah perlu push notification jika order di-cancel karena lifetime timeout?

9. **Analytics** - Data apa yang perlu di-track untuk business intelligence? (peak hours, cancel patterns, dll)

## 10. References

- [[2025-01-12-Traffic-Spike-Analysis]] - Incident yang trigger requirement ini
- [[RFC-Server-Busy-Architecture-Analysis]] - Analisis arsitektur dan benchmark industri (Grab, Gojek, Uber)
- [[SPOF Analysis MRG]] - Order Orchestrator sebagai critical component
- [[Exponential Backoff Retry Pattern]] - Related pattern untuk BBD callback

---

## Appendix A: Redis Commands Reference

```redis
# Add order to waiting set
ZADD waiting:order:JKT-PUSAT:RIDE {timestamp} {order_id}

# Get waiting count
ZCARD waiting:order:JKT-PUSAT:RIDE

# Get expired orders (older than 300 seconds)
ZRANGEBYSCORE waiting:order:JKT-PUSAT:RIDE -inf {now - 300}

# Remove order (on dispatch success or cancel)
ZREM waiting:order:JKT-PUSAT:RIDE {order_id}

# Get threshold config
HGETALL threshold:config:JKT-PUSAT:RIDE
```

## Appendix B: Error Codes

| Code | HTTP Status | Description |
|------|-------------|-------------|
| DISPATCHER_AREA_THRESHOLD_REACHED | 503 | Area sudah mencapai max waiting orders |
| DISPATCHER_AREA_DISABLED | 503 | Area sedang di-disable (maintenance/emergency) |
| DISPATCHER_AREA_NOT_FOUND | 400 | Koordinat tidak masuk area manapun |
| DISPATCHER_SERVICE_UNAVAILABLE | 503 | Service dispatcher tidak available |
