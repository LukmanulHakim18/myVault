---
title: CC Charging Flow - Sequence Diagrams
type: design-doc
status: draft
team: MRG
created: '2026-02-12'
updated: '2026-02-12'
tags:
  - sequence-diagram
  - charging
  - cc
  - payment
  - mrg
  - upg
  - bbd
  - n2c
---
# CC Charging Flow - Sequence Diagrams

**Created:** 2026-02-12  
**Updated:** 2026-02-12  
**Status:** Draft  
**Related:** [[02-spike-new-charging-flow]], [[01-n2c-flow-documentation]]

---

## Overview

Sequence diagram untuk flow charging CC yang baru. Flow terbagi menjadi 3 fase:
1. **Selama Trip** - Preauth polling (penjagaan)
2. **Stop Argo** - Direct charge main payment
3. **Endtrip** - Extended payment untuk extra

---

## Participants

| Service | Layer | Role |
|---------|-------|------|
| IoT | Device | Driver device (stop argo, input extra, endtrip) |
| BBD | External | Forward event dari IoT ke MRG |
| N2C | MRG | Validate n2c, orchestrate preauth/charge |
| payment_processor | MRG | Handle charging logic |
| Redis | Storage | Order state management |
| PGW | MRG | Payment gateway internal |
| UPG | Payment | Payment middleware |
| PayProv | External | Payment provider (bank/CC) |
| TOP | MRG | Taxi order processor |
| DBOrder | Storage | Order database |
| taxi-partner-gw | MRG | Gateway ke BBD services |
| bb-trip | BBD | Handle driver notification |
| notif_center | Notification | Notification orchestrator |
| bbone_notif | Notification | Bluebird One notification |
| Firebase | External | Push notification delivery |

---

## N2C Request Flag

| Field | Value | Behavior |
|-------|-------|----------|
| `trip_status` | `ON_TRIP` | Preauth polling (existing flow) |
| `trip_status` | `END_TRIP` | Direct charge (new flow) |

---

## Sequence Diagram 1: Selama Trip (Preauth Polling)

```mermaid
sequenceDiagram
    autonumber
    participant BBD as BBD
    participant N2C as N2C Service
    participant Redis as Redis
    participant PGW as Payment GW
    participant UPG as UPG
    participant PayProv as Payment Provider
    participant TOP as TOP
    participant DBOrder as Order DB
    participant NC as notif_center
    participant BBOne as bbone_notif
    participant Firebase as Firebase

    loop Every 20 seconds
        BBD->>N2C: validate n2c (order_id, actual_argo, trip_status="ON_TRIP")
        N2C->>Redis: get order detail (payment_method, last_reserve_value)
        
        alt payment_method == CASH
            N2C-->>BBD: need_n2c = true (already cash)
        else actual_argo < last_reserve_value
            N2C-->>BBD: need_n2c = false (preauth sufficient)
        else actual_argo >= last_reserve_value
            N2C->>PGW: cancel pre-auth
            PGW->>UPG: cancel pre-auth
            UPG->>PayProv: cancel pre-auth
            PayProv-->>UPG: success
            UPG-->>PGW: success
            PGW-->>N2C: success
            
            N2C->>PGW: new pre-auth (actual_argo + markup)
            PGW->>UPG: pre-auth request
            UPG->>PayProv: pre-auth request
            
            alt Pre-auth Success
                PayProv-->>UPG: success
                UPG-->>PGW: success
                PGW-->>N2C: success
                N2C->>Redis: set last_reserve_value
                N2C-->>BBD: need_n2c = false
            else Pre-auth Failed
                PayProv-->>UPG: failed
                UPG-->>PGW: failed
                PGW-->>N2C: failed
                N2C->>TOP: update payment to CASH
                TOP->>DBOrder: UPDATE payment_method = CASH
                N2C->>Redis: set payment_method = CASH
                N2C-->>BBD: need_n2c = true
                N2C->>NC: publish payment_switched event
                NC->>BBOne: send notification
                BBOne->>Firebase: push to user
            end
        end
    end
```

---

## Sequence Diagram 2: Stop Argo (Direct Charge)

```mermaid
sequenceDiagram
    autonumber
    participant IoT as IoT Device
    participant BBD as BBD
    participant N2C as N2C Service
    participant PP as payment_processor
    participant Redis as Redis
    participant PGW as Payment GW
    participant UPG as UPG
    participant PayProv as Payment Provider
    participant TOP as TOP
    participant DBOrder as Order DB
    participant TPG as taxi-partner-gw
    participant BBTrip as bb-trip
    participant NC as notif_center
    participant BBOne as bbone_notif
    participant Firebase as Firebase

    Note over IoT: Driver tap "Stop Argo"
    IoT->>BBD: stop_argo event
    Note over BBD: Stop n2c polling
    
    BBD->>N2C: validate n2c (order_id, actual_argo, trip_status="END_TRIP")
    N2C->>Redis: get order detail (payment_method, last_reserve_value)
    
    alt payment_method == CASH
        N2C-->>BBD: need_n2c = true (already cash)
        Note over BBD: Driver already knows to collect cash
    else payment_method == CC
        Note over N2C: Calculate total (argo + platform_fee - discount)
        N2C->>PP: request direct charge
        PP->>PGW: charge request (total_amount)
        PGW->>UPG: charge request
        UPG->>PayProv: charge request
        
        alt Charge Success (authorize >= capture)
            PayProv-->>UPG: success
            Note over UPG: If authorize > capture, refund difference
            UPG-->>PGW: success
            PGW-->>PP: success
            PP->>Redis: update status = PAID_MAIN
            PP-->>N2C: success
            N2C-->>BBD: need_n2c = false
            Note over IoT: ✅ Driver can input extra
            
        else Charge Partial (authorize < capture)
            PayProv-->>UPG: partial (charged: authorize_amount)
            Note over UPG: Sisa jadi outstanding
            UPG-->>PGW: partial
            PGW-->>PP: partial (outstanding: remaining)
            PP->>Redis: update status = PARTIAL_PAID, set outstanding
            PP-->>N2C: partial success
            N2C-->>BBD: need_n2c = false (partial)
            Note over UPG: Scheduler retry outstanding
            Note over IoT: ✅ Driver can input extra
            
        else Charge Failed
            PayProv-->>UPG: failed
            UPG-->>PGW: failed
            PGW-->>PP: failed
            PP->>TOP: update payment to CASH
            TOP->>DBOrder: UPDATE payment_method = CASH
            PP->>Redis: set payment_method = CASH
            PP-->>N2C: failed
            N2C-->>BBD: need_n2c = true
            
            par Notify User
                N2C->>NC: publish payment_switched event
                NC->>BBOne: send notification
                BBOne->>Firebase: push "Pembayaran diubah ke tunai"
            and Notify Driver
                N2C->>TPG: POST /notify-payment-switch
                TPG->>BBTrip: forward request
                Note over BBTrip: Eskalasi ke DA + IoT
            end
            Note over IoT: ⚠️ Driver: "Tagih Tamu"
        end
    end
```

---

## Sequence Diagram 3: Endtrip (Extended Payment for Extra)

```mermaid
sequenceDiagram
    autonumber
    participant IoT as IoT Device
    participant BBD as BBD
    participant MRG as MRG (REST)
    participant Kafka as Kafka
    participant PP as payment_processor
    participant Redis as Redis
    participant PGW as Payment GW
    participant UPG as UPG
    participant PayProv as Payment Provider
    participant DBOrder as Order DB

    Note over IoT: Driver input extra → tap "Complete"
    IoT->>BBD: endtrip event (extra, actual_dropoff, polyline, etc)
    BBD->>MRG: POST /endtrip (order_id, extra, complete_data)
    MRG->>Kafka: publish endtrip event
    MRG-->>BBD: 200 OK
    
    Kafka->>PP: consume endtrip event
    PP->>Redis: get order detail (payment_method, status)
    
    alt payment_method == CASH
        PP->>DBOrder: update order complete data
        PP->>Redis: update status = COMPLETED
        Note over PP: ✅ Order Complete (cash)
        
    else payment_method == CC AND extra == 0
        PP->>DBOrder: update order complete data
        PP->>Redis: update status = COMPLETED
        Note over PP: ✅ Order Complete (no extra charge)
        
    else payment_method == CC AND extra > 0
        PP->>PGW: charge extra (extended payment)
        PGW->>UPG: charge request (extra_amount)
        UPG->>PayProv: charge request
        
        alt Extra Charge Success
            PayProv-->>UPG: success
            UPG-->>PGW: success
            PGW-->>PP: success
            PP->>DBOrder: update order complete data
            PP->>Redis: update status = COMPLETED
            Note over PP: ✅ Order Complete
            
        else Extra Charge Failed
            PayProv-->>UPG: failed
            UPG-->>PGW: failed
            PGW-->>PP: failed
            PP->>Redis: update status = PARTIALLY_PAID
            PP->>Redis: set outstanding_amount = extra
            PP->>DBOrder: update order (partially_paid)
            Note over PP: ⚠️ Order Partially Paid
            Note over UPG: Scheduler retry for extra
        end
    end
```

---

## Flow Summary

```
┌─────────────────────────────────────────────────────────────────┐
│                        SELAMA TRIP                               │
│  BBD polling N2C setiap 20s (trip_status = "ON_TRIP")           │
│  → Preauth validation & refresh                                  │
│  → Jika preauth gagal → switch to CASH                          │
└─────────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────────┐
│                        STOP ARGO                                 │
│  IoT → BBD → N2C (trip_status = "END_TRIP")                     │
│  → Direct charge (argo + platform_fee - discount)               │
│  → UPG handle selisih preauth vs charge otomatis                │
│  → Jika gagal → switch to CASH → notify user + driver           │
└─────────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────────┐
│                     INPUT EXTRA (IoT)                            │
│  Driver input extra amount (bisa 0)                             │
└─────────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────────┐
│                        ENDTRIP                                   │
│  IoT → BBD → MRG (complete data + extra)                        │
│  → Jika extra > 0 → charge extra (extended payment)             │
│  → Jika gagal → partially paid (outstanding)                    │
│  → Order complete                                                │
└─────────────────────────────────────────────────────────────────┘
```

---

## Charge Scenarios Summary

| Phase | Scenario | Result | Action |
|-------|----------|--------|--------|
| Stop Argo | authorize >= capture | Success | Main payment complete, refund sisa |
| Stop Argo | authorize < capture | Partial | Main captured, sisa outstanding |
| Stop Argo | Failed | Switch CASH | Notify user + driver |
| Endtrip | extra = 0 | Success | Order complete |
| Endtrip | extra > 0, success | Success | Order complete |
| Endtrip | extra > 0, failed | Partial | Extra jadi outstanding |

---

## Contract: N2C Payment Eligibility

### Endpoint
```
POST /n2c-service/v1/payment/eligibility
Host: dev-mybb-api.internal.bluebird.id
```

### Request
```json
{
  "order_id": "string",
  "actual_argo": 250000,
  "trip_status": "ON_TRIP | END_TRIP",
  "payment_method": "E_WALLET | CREDIT_CARD"
}
```

| Field | Type | Description |
|-------|------|-------------|
| `order_id` | string | Order ID |
| `actual_argo` | number | Actual argo amount |
| `trip_status` | string | `ON_TRIP` (polling) atau `END_TRIP` (direct charge) |
| `payment_method` | string | `E_WALLET` atau `CREDIT_CARD` |

### Response

**Case 1: Balance sufficient / CC preauth valid**
```json
HTTP 200 OK
{
  "success": true,
  "data": {
    "order_id": "string",
    "is_sufficient": true,
    "force_to_cash": false
  }
}
```

**Case 2: Balance insufficient, user notified for top up**
```json
HTTP 200 OK
{
  "success": true,
  "data": {
    "order_id": "string",
    "is_sufficient": false,
    "force_to_cash": false
  }
}
```

**Case 3: Balance insufficient / CC preauth invalid, forced to cash**
```json
HTTP 200 OK
{
  "success": true,
  "data": {
    "order_id": "string",
    "is_sufficient": false,
    "force_to_cash": true
  }
}
```

**Case 4: Error**
```json
HTTP 500 Internal Server Error
{
  "error_code": "N2CS-50000",
  "error_message": "Internal server error"
}
```

### Response Logic

| `is_sufficient` | `force_to_cash` | Meaning |
|-----------------|-----------------|---------|
| `true` | `false` | OK, lanjut (balance cukup / preauth valid) |
| `false` | `false` | E-wallet: user notified top up, CC: preauth refreshed |
| `false` | `true` | Switch to CASH, notify driver "tagih tamu" |
```

---

## Contract: MRG → bb-trip (Payment Switch)

### Endpoint
```
POST /notify-payment-switch
Host: taxi-partner-gw
```

### Request
```json
{
  "order_id": "ORD-123456",
  "payment_method": "CASH",
  "reason": "CHARGE_FAILED | PREAUTH_FAILED",
  "timestamp": "2026-02-12T10:30:00Z"
}
```

---

## Notes

1. **Preauth tetap ada** selama trip sebagai penjagaan
2. **UPG Enhancement**: authorize vs capture selisih di-handle otomatis
3. **Outstanding**: UPG scheduler retry, user blocked sampai selesai
4. **Tips**: Excluded, akan di-handle di extended tipping flow terpisah
