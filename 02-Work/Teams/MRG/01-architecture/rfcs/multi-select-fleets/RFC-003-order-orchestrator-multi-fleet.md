---
title: 'RFC-003: Order Orchestrator Multi-fleet Support'
tags:
  - rfc
  - mrg
  - multi-select-fleets
  - order-orchestrator
created: '2026-03-16'
updated: '2026-03-18'
status: draft
author: Lukmanul Hakim
team: MRG
---
# RFC-003: Order Orchestrator Multi-fleet Support

**Status:** Draft  
**Author:** Lukmanul Hakim  
**Created:** 2026-03-16  
**Updated:** 2026-03-18  
**Related:** [[RFC-001-multi-select-fleets-overview]], [[RFC-002-session-manager-fleet-pricing]], [[RFC-004-payment-flow-multi-fleet]]

---

## 1. Overview

### 1.1. Problem Statement

Order Orchestrator saat ini hanya mendukung single fleet selection. Untuk Multi-select Fleets, perlu enhancement untuk:
- Menerima array `selected_fleets` dari client
- Copy fleet pricing dari Session Manager ke database (persistent storage)
- Forward ke Dispatcher dengan `item_id` + `price_code`
- Query database untuk winner fleet pricing setelah driver assigned

### 1.2. Proposed Solution

Extend Order Orchestrator untuk:
- Handle `selected_fleets[]` di create order request
- Copy fleet pricing ke table `order_fleet_pricing` saat order dibuat
- Forward `selected_fleets[]` dengan `item_id` + `price_code` ke Dispatcher
- Query `order_fleet_pricing` untuk winner price (bukan Session Manager)

### 1.3. Goals

- Harga fixed saat create order — tidak berubah selama fase retry 30 menit
- Persistent storage untuk audit trail
- Backward compatible dengan single fleet selection

### 1.4. Non-Goals

- Order Orchestrator tidak menghitung fleet eligibility
- Order Orchestrator tidak menentukan winner (Dispatcher domain)
- Order Orchestrator tidak translate `item_id` + `price_code` (BBD domain)

### 1.5. Key Change dari RFC Sebelumnya

> **PENTING:** Pricing data disimpan ke database saat create order, bukan di Session Manager. Ini memastikan data tidak hilang selama fase retry 30 menit.

---

## 2. Database Schema

### 2.1. New Table: order_fleet_pricing

```sql
CREATE TABLE order_fleet_pricing (
    id              BIGSERIAL PRIMARY KEY,
    order_id        VARCHAR(36) NOT NULL,
    fleet_code      VARCHAR(20) NOT NULL,   -- "BB", "BB_PRIME", "SB"
    price_type      VARCHAR(10) NOT NULL,   -- "ARGO", "FIXED"
    item_id         VARCHAR(50) NOT NULL,   -- representasi ke Price Engine
    price_code      VARCHAR(50) NOT NULL,   -- representasi ke Price Engine
    estimated_fare  BIGINT NOT NULL,
    platform_fee    BIGINT NOT NULL,
    total_price     BIGINT NOT NULL,
    created_at      TIMESTAMP DEFAULT NOW(),
    
    CONSTRAINT fk_order FOREIGN KEY (order_id) REFERENCES orders(id),
    CONSTRAINT uq_order_fleet UNIQUE (order_id, fleet_code)
);

CREATE INDEX idx_order_fleet_pricing_order_id ON order_fleet_pricing(order_id);
```

### 2.2. Data Model

```go
type OrderFleetPricing struct {
    ID            int64     `db:"id"`
    OrderID       string    `db:"order_id"`
    FleetCode     string    `db:"fleet_code"`
    PriceType     string    `db:"price_type"`
    ItemID        string    `db:"item_id"`
    PriceCode     string    `db:"price_code"`
    EstimatedFare int64     `db:"estimated_fare"`
    PlatformFee   int64     `db:"platform_fee"`
    TotalPrice    int64     `db:"total_price"`
    CreatedAt     time.Time `db:"created_at"`
}
```

### 2.3. Why Separate Table?

| Approach | Pros | Cons |
|----------|------|------|
| Column di order table | Simple | Tidak bisa store multiple fleets |
| JSON column | Flexible | Query kompleks, no FK constraint |
| **Separate table** | Clean, queryable, FK enforced | Extra join |

Winner di-representasikan oleh `fleet_code` di table `orders`. Table `order_fleet_pricing` hanya untuk historical lookup.

---

## 3. API Changes

### 3.1. Create Order Request

**Current:**

```go
type CreateOrderRequest struct {
    SessionID     string        `json:"session_id"`
    BBID          string        `json:"bbid"`
    FleetCode     string        `json:"fleet_code"`      // single fleet
    Pickup        Location      `json:"pickup"`
    Dropoff       Location      `json:"dropoff"`
    PaymentMethod PaymentMethod `json:"payment_method"`
    // ... other fields
}
```

**New:**

```go
type CreateOrderRequest struct {
    SessionID      string                   `json:"session_id"`
    BBID           string                   `json:"bbid"`
    FleetCode      string                   `json:"fleet_code"`      // deprecated, backward compat
    SelectedFleets []SelectedFleetRequest   `json:"selected_fleets"` // NEW
    Pickup         Location                 `json:"pickup"`
    Dropoff        Location                 `json:"dropoff"`
    PaymentMethod  PaymentMethod            `json:"payment_method"`
    // ... other fields
}

type SelectedFleetRequest struct {
    FleetCode string `json:"fleet_code"`
    ItemID    string `json:"item_id"`
    PriceCode string `json:"price_code"`
}
```

### 3.2. Backward Compatibility

```go
func (o *OrderOrchestrator) normalizeFleetSelection(req *CreateOrderRequest, sessionSnapshot *FleetPricingSnapshot) {
    if len(req.SelectedFleets) == 0 && req.FleetCode != "" {
        // Single fleet mode - ambil dari session snapshot
        fleetPrice := sessionSnapshot.Fleets[req.FleetCode]
        req.SelectedFleets = []SelectedFleetRequest{{
            FleetCode: req.FleetCode,
            ItemID:    fleetPrice.ItemID,
            PriceCode: fleetPrice.PriceCode,
        }}
    }
}
```

### 3.3. Create Order Response

```go
type CreateOrderResponse struct {
    OrderID      string `json:"order_id"`
    Status       string `json:"status"`
    WinnerFleet  string `json:"winner_fleet"`  // setelah driver assigned
    TotalPrice   int64  `json:"total_price"`
    PlatformFee  int64  `json:"platform_fee"`
    // ... other fields
}
```

---

## 4. Flow Changes

### 4.1. Create Order Flow

```
Client -> OO: CreateOrder(selected_fleets[])
    |
    v
OO: Validate selected_fleets not empty
OO: Get snapshot from Session Manager
OO: Normalize fleet selection (backward compat)
    |
    v
OO: Copy ALL selected fleet pricing ke order_fleet_pricing table
    |
    v
OO -> Dispatcher: Dispatch(order_id, selected_fleets[] with item_id + price_code)
    |
    v
Dispatcher: Calculate score per fleet
Dispatcher: Retry up to 30 minutes
Dispatcher: Select winner + Assign driver
    |
    v
Dispatcher -> OO: DispatchResponse(winner_fleet, driver_id, taxi_number)
    |
    v
OO: Query order_fleet_pricing WHERE order_id = X AND fleet_code = winner
OO: Apply winner price to order
OO: Update order table with winner fleet
    |
    v
OO -> Client: CreateOrderResponse(order_id, winner_fleet, total_price)
```

### 4.2. Code Implementation

```go
func (o *OrderOrchestrator) CreateOrder(ctx context.Context, req *CreateOrderRequest) (*CreateOrderResponse, error) {
    // 1. Get snapshot from Session Manager
    snapshotResp, err := o.sessionManager.GetFleetPricingSnapshot(ctx, &sm.GetFleetPricingSnapshotRequest{
        SessionId: req.SessionID,
    })
    if err != nil || !snapshotResp.Found {
        return nil, status.Error(codes.FailedPrecondition, "session expired, please search again")
    }
    
    // 2. Normalize fleet selection
    o.normalizeFleetSelection(req, snapshotResp.Snapshot)
    
    // 3. Validate
    if len(req.SelectedFleets) == 0 {
        return nil, status.Error(codes.InvalidArgument, "selected_fleets required")
    }
    
    // 4. Generate order ID
    orderID := o.generateOrderID()
    
    // 5. Copy ALL selected fleet pricing ke database
    for _, sf := range req.SelectedFleets {
        fleetPrice := snapshotResp.Snapshot.Fleets[sf.FleetCode]
        err := o.orderFleetPricingRepo.Insert(ctx, &OrderFleetPricing{
            OrderID:       orderID,
            FleetCode:     sf.FleetCode,
            PriceType:     fleetPrice.PriceType,
            ItemID:        sf.ItemID,
            PriceCode:     sf.PriceCode,
            EstimatedFare: fleetPrice.EstimatedFare,
            PlatformFee:   fleetPrice.PlatformFee,
            TotalPrice:    fleetPrice.TotalPrice,
        })
        if err != nil {
            return nil, status.Error(codes.Internal, "failed to save fleet pricing")
        }
    }
    
    // 6. Build dispatch request with item_id + price_code
    dispatchFleets := make([]DispatchFleet, len(req.SelectedFleets))
    for i, sf := range req.SelectedFleets {
        dispatchFleets[i] = DispatchFleet{
            FleetCode: sf.FleetCode,
            ItemID:    sf.ItemID,
            PriceCode: sf.PriceCode,
        }
    }
    
    // 7. Dispatch
    dispatchResp, err := o.dispatcher.Dispatch(ctx, &dispatcher.DispatchRequest{
        OrderId:        orderID,
        SelectedFleets: dispatchFleets,
        Pickup:         req.Pickup,
        Dropoff:        req.Dropoff,
    })
    if err != nil {
        return nil, status.Error(codes.Internal, "dispatch failed")
    }
    
    // 8. Get winner price from database (NOT Session Manager)
    winnerPricing, err := o.orderFleetPricingRepo.GetByOrderAndFleet(ctx, orderID, dispatchResp.WinnerFleet)
    if err != nil {
        return nil, status.Error(codes.Internal, "winner price not found")
    }
    
    // 9. Build order with winner price
    order := &Order{
        ID:          orderID,
        BBID:        req.BBID,
        FleetCode:   dispatchResp.WinnerFleet,
        DriverID:    dispatchResp.DriverId,
        TaxiNumber:  dispatchResp.TaxiNumber,
        BasePrice:   winnerPricing.EstimatedFare,
        PlatformFee: winnerPricing.PlatformFee,
        TotalPrice:  winnerPricing.TotalPrice,
        // ... other fields
    }
    
    // 10. Save to DB
    err = o.orderRepo.Create(ctx, order)
    if err != nil {
        return nil, status.Error(codes.Internal, "failed to save order")
    }
    
    return &CreateOrderResponse{
        OrderId:     orderID,
        Status:      "CREATED",
        WinnerFleet: order.FleetCode,
        TotalPrice:  order.TotalPrice,
        PlatformFee: order.PlatformFee,
    }, nil
}
```

---

## 5. Dispatcher Integration

### 5.1. Dispatch Request

```go
type DispatchRequest struct {
    OrderID        string         `json:"order_id"`
    SelectedFleets []DispatchFleet `json:"selected_fleets"`
    Pickup         Location       `json:"pickup"`
    Dropoff        Location       `json:"dropoff"`
}

type DispatchFleet struct {
    FleetCode string `json:"fleet_code"` // "BB", "BB_PRIME"
    ItemID    string `json:"item_id"`    // untuk BBD -> Price Engine
    PriceCode string `json:"price_code"` // untuk BBD -> Price Engine
}
```

### 5.2. Dispatch Response

```go
type DispatchResponse struct {
    OrderID     string `json:"order_id"`
    WinnerFleet string `json:"winner_fleet"` // "BB_PRIME"
    DriverID    string `json:"driver_id"`
    VehicleID   string `json:"vehicle_id"`
    TaxiNumber  string `json:"taxi_number"`  // "B 1234 XYZ"
    ETA         int    `json:"eta"`          // minutes
}
```

---

## 6. Repository Layer

### 6.1. OrderFleetPricingRepository

```go
type OrderFleetPricingRepository interface {
    Insert(ctx context.Context, pricing *OrderFleetPricing) error
    InsertBatch(ctx context.Context, pricings []*OrderFleetPricing) error
    GetByOrderAndFleet(ctx context.Context, orderID, fleetCode string) (*OrderFleetPricing, error)
    GetAllByOrder(ctx context.Context, orderID string) ([]*OrderFleetPricing, error)
}
```

### 6.2. Implementation

```go
func (r *orderFleetPricingRepo) Insert(ctx context.Context, pricing *OrderFleetPricing) error {
    query := `
        INSERT INTO order_fleet_pricing 
        (order_id, fleet_code, price_type, item_id, price_code, estimated_fare, platform_fee, total_price)
        VALUES ($1, $2, $3, $4, $5, $6, $7, $8)
    `
    _, err := r.db.ExecContext(ctx, query,
        pricing.OrderID, pricing.FleetCode, pricing.PriceType,
        pricing.ItemID, pricing.PriceCode,
        pricing.EstimatedFare, pricing.PlatformFee, pricing.TotalPrice,
    )
    return err
}

func (r *orderFleetPricingRepo) GetByOrderAndFleet(ctx context.Context, orderID, fleetCode string) (*OrderFleetPricing, error) {
    query := `
        SELECT id, order_id, fleet_code, price_type, item_id, price_code, 
               estimated_fare, platform_fee, total_price, created_at
        FROM order_fleet_pricing
        WHERE order_id = $1 AND fleet_code = $2
    `
    var pricing OrderFleetPricing
    err := r.db.GetContext(ctx, &pricing, query, orderID, fleetCode)
    return &pricing, err
}
```

---

## 7. Error Handling

| Scenario | Handling |
|----------|----------|
| Session expired | Return FailedPrecondition "session expired, please search again" |
| `selected_fleets` empty | Return InvalidArgument |
| Insert fleet pricing failed | Return Internal error |
| Dispatch failed | Return Internal error |
| Winner price not found | Return Internal error (seharusnya tidak terjadi) |
| DB save failed | Return Internal error |

---

## 8. Observability

### 8.1. Metrics

| Metric | Type | Description |
|--------|------|-------------|
| `order_orchestrator_multi_fleet_orders_total` | Counter | Orders dengan multi-fleet selection |
| `order_orchestrator_single_fleet_orders_total` | Counter | Orders dengan single fleet (backward compat) |
| `order_orchestrator_winner_fleet_distribution` | Counter | Distribution of winner fleets |
| `order_orchestrator_fleet_pricing_insert_latency_ms` | Histogram | Latency insert ke database |
| `order_orchestrator_fleet_pricing_query_latency_ms` | Histogram | Latency query winner price |

### 8.2. Logging

```go
// Order created
log.Info("order created with multi-fleet",
    "order_id", orderID,
    "selected_fleets", selectedFleets,
    "fleet_count", len(selectedFleets),
)

// Winner determined
log.Info("winner fleet determined",
    "order_id", orderID,
    "winner_fleet", winnerFleet,
    "total_price", totalPrice,
)
```

---

## 9. Feature Flag

```go
const FeatureMultiSelectFleets = "multi_select_fleets"

func (o *OrderOrchestrator) isMultiSelectEnabled(ctx context.Context, bbid string) bool {
    return o.featureFlag.IsEnabled(ctx, FeatureMultiSelectFleets, bbid)
}
```

Jika disabled, enforce single fleet:

```go
if !o.isMultiSelectEnabled(ctx, req.BBID) && len(req.SelectedFleets) > 1 {
    req.SelectedFleets = req.SelectedFleets[:1] // take first only
}
```

---

## 10. Migration Plan

### 10.1. Database Migration

```sql
-- Create order_fleet_pricing table
CREATE TABLE order_fleet_pricing (
    id              BIGSERIAL PRIMARY KEY,
    order_id        VARCHAR(36) NOT NULL,
    fleet_code      VARCHAR(20) NOT NULL,
    price_type      VARCHAR(10) NOT NULL,
    item_id         VARCHAR(50) NOT NULL,
    price_code      VARCHAR(50) NOT NULL,
    estimated_fare  BIGINT NOT NULL,
    platform_fee    BIGINT NOT NULL,
    total_price     BIGINT NOT NULL,
    created_at      TIMESTAMP DEFAULT NOW(),
    
    CONSTRAINT fk_order FOREIGN KEY (order_id) REFERENCES orders(id),
    CONSTRAINT uq_order_fleet UNIQUE (order_id, fleet_code)
);

CREATE INDEX idx_order_fleet_pricing_order_id ON order_fleet_pricing(order_id);
```

### 10.2. Deployment Sequence

1. Deploy DB migration
2. Deploy Session Manager (RFC-002)
3. Deploy Order Orchestrator with feature flag OFF
4. Enable feature flag for whitelist users
5. Gradual rollout
6. Full rollout

---

## 11. Open Items

| # | Item | Status |
|---|------|--------|
| 1 | Dispatcher API contract confirmation | Pending |
| 2 | Feature flag integration | Pending |

---

## 12. References

- [[RFC-001-multi-select-fleets-overview]]
- [[RFC-002-session-manager-fleet-pricing]]
- [[RFC-004-payment-flow-multi-fleet]]
