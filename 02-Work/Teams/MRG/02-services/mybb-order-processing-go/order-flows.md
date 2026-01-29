---
tags:
  - mrg
  - service
  - flows
  - order-processing
parent: '[[README]]'
created: '2026-01-29'
updated: '2026-01-29'
---
# Order Processing - Order Flows

**Parent**: [[README|Order Processing Service]]

---

## 🔄 Request Booking Flow

### Sequence Diagram

```mermaid
sequenceDiagram
    participant AG as API Gateway
    participant OP as Order Processing
    participant LR as Location Registry
    participant PP as Payment Processor
    participant MPG as Meta Payment Gateway
    participant ITOP as ITOP/BBD
    participant DB as PostgreSQL
    participant MG as MongoDB

    AG->>OP: TOPIC_REQUEST_BOOKING
    
    Note over OP: Parse & Validate Message
    
    OP->>OP: Check Redis Lock (prevent duplicate)
    
    OP->>OP: Check User Banned Status
    alt User Hard Banned
        OP-->>AG: Error: User Banned
    end
    
    OP->>LR: GetAreaByLocation(lat, lng)
    LR-->>OP: Area + Service Types
    
    OP->>OP: Validate Service Type
    
    alt Payment = Credit Card + PreAuth Enabled
        OP->>PP: RequestPreAuth(order, amount)
        PP-->>OP: PreAuth Response
        alt PreAuth Required
            OP->>DB: Create Order (state: INITIAL)
            OP-->>AG: NOTIFY_BOOKING_RESULT (PreAuth Pending)
            Note over OP: Wait for POST_REQUEST_BOOKING callback
        end
    end
    
    alt Payment = Wallet
        OP->>MPG: GetWalletBalance()
        MPG-->>OP: Balance
        alt Insufficient Balance
            OP-->>AG: Error: Insufficient Balance
        end
        OP->>PP: ReserveWallet(amount)
    end
    
    OP->>DB: Create Order
    DB-->>OP: Order Created
    
    OP->>MG: Create Order Info
    
    alt Service = Golden Bird
        OP->>OP: Handle GB Booking
        alt GB Rent Multidays
            OP-->>AG: NOTIFY_BOOKING_RESULT
            Note over OP: End Flow (No BBD)
        end
    end
    
    OP->>ITOP: CreateNewJob
    ITOP-->>OP: Job Created (PassCode, ExternalOrderID)
    
    OP->>DB: Update Order (PassCode, ExternalOrderID)
    
    OP-->>AG: NOTIFY_BOOKING_RESULT
```

### Key Components

1. **Redis Lock** - Prevent duplicate order creation
2. **Soft Ban Check** - User dengan banyak cancel
3. **Hard Ban Check** - User blacklist
4. **Area Validation** - Validasi lokasi pickup
5. **PreAuth Flow** - Credit card pre-authorization
6. **Wallet Reserve** - E-wallet balance reservation
7. **BBD Integration** - ITOP dispatch

---

## 🔄 Post Request Booking Flow (PreAuth Callback)

```mermaid
sequenceDiagram
    participant PP as Payment Processor
    participant OP as Order Processing
    participant ITOP as ITOP/BBD
    participant DB as PostgreSQL
    participant AG as API Gateway

    PP->>OP: TOPIC_POST_REQUEST_BOOKING
    
    OP->>OP: Get Order from Redis Session
    
    alt PreAuth Success
        OP->>ITOP: CreateNewJob
        ITOP-->>OP: Job Created
        OP->>DB: Update Order (PassCode, ExternalOrderID)
        OP-->>AG: NOTIFY_BOOKING_RESULT (Success)
    else PreAuth Failed
        OP->>DB: Update Order (state: NONE)
        OP-->>AG: NOTIFY_BOOKING_RESULT (PreAuth Failed)
    end
```

---

## 🔄 Update Current Booking Flow

```mermaid
sequenceDiagram
    participant BBD as BBD Dispatch
    participant OP as Order Processing
    participant DB as PostgreSQL
    participant AG as API Gateway

    BBD->>OP: TOPIC_UPDATE_CURRENT_BOOKING
    
    Note over OP: Event Consumer (no group)
    
    OP->>DB: Get Order by ExternalOrderID
    
    alt Order Found
        OP->>OP: Update Order State
        OP->>OP: Update Driver Info
        OP->>OP: Update Vehicle Info
        OP->>DB: Save Order
        OP-->>AG: Notify Order Changes
    end
```

### State Transitions
| From State | To State | Trigger |
|------------|----------|---------|
| INITIAL | LOOKING_FOR_DRIVER | BBD accepted job |
| LOOKING_FOR_DRIVER | DRIVER_FOUND | Driver assigned |
| DRIVER_FOUND | ON_TRIP | Trip started |
| ON_TRIP | TRIP_ENDED | Driver end trip |

---

## 🔄 End Trip Flow

```mermaid
sequenceDiagram
    participant BBD as BBD Dispatch
    participant OP as Order Processing
    participant PP as Payment Processor
    participant DB as PostgreSQL
    participant GCS as Google Cloud Storage
    participant Email as Email Service
    participant AG as API Gateway

    BBD->>OP: TOPIC_END_TRIP
    
    OP->>OP: Acquire Redis Lock
    
    OP->>DB: Get Order by ExternalOrderID + ItopID
    
    alt Order Not Found or Already Ended
        OP-->>OP: Skip Processing
    end
    
    OP->>OP: Calculate Final Fare
    
    alt Payment = Wallet
        OP->>PP: ChargeWallet(finalFare)
        PP-->>OP: Charge Result
        alt Charge Failed
            OP->>DB: Update Order (charge_status: failed)
        end
    end
    
    alt Payment = Credit Card
        OP->>PP: CapturePreAuth(finalFare)
        PP-->>OP: Capture Result
    end
    
    OP->>DB: Update Order (state: TRIP_ENDED)
    
    par Generate Trip Image
        OP->>GCS: Upload Trip Map Image
    and Send Receipt
        OP->>Email: Send Trip Receipt
    end
    
    OP-->>AG: Notify Order Changes
    OP-->>AG: TOPIC_NOTIFY_END_TRIP
```

### Fare Calculation
```go
// Fare calculation components
FinalFare = TripFare + ServicesFee + Tips - PromoDiscount

// Where:
// TripFare = Metered fare atau Fixed fare
// ServicesFee = Handling fee, platform fee, etc.
// Tips = Optional driver tips
// PromoDiscount = Promo/voucher discount
```

---

## 🔄 Cancel Booking Flow

```mermaid
sequenceDiagram
    participant AG as API Gateway
    participant OP as Order Processing
    participant PP as Payment Processor
    participant ITOP as ITOP/BBD
    participant DB as PostgreSQL

    AG->>OP: TOPIC_CANCEL_BOOKING
    
    OP->>OP: Acquire Redis Lock
    
    OP->>DB: Get Order
    
    alt Order Cannot Be Cancelled
        OP-->>AG: Error: Invalid State
    end
    
    OP->>OP: Calculate Cancellation Fee
    
    alt Has Cancellation Fee
        alt Payment = Wallet
            OP->>PP: ChargeWallet(cancellationFee)
        else Payment = Credit Card
            OP->>PP: ChargeCancellationFee(fee)
        end
    end
    
    alt PreAuth Exists
        OP->>PP: ReleasePreAuth()
    end
    
    OP->>ITOP: CancelBooking(externalOrderID)
    
    OP->>DB: Update Order (state: CANCELLED)
    
    OP->>OP: Update Cancel Counter (for soft ban)
    
    OP-->>AG: Notify Order Changes
```

### Cancellation Rules
| State | Can Cancel | Fee |
|-------|------------|-----|
| INITIAL | ✅ Yes | No |
| LOOKING_FOR_DRIVER | ✅ Yes | No |
| DRIVER_FOUND | ✅ Yes | Maybe |
| ON_TRIP | ❌ No | - |
| TRIP_ENDED | ❌ No | - |

---

## 🔄 Cancel Booking by System Flow

```mermaid
sequenceDiagram
    participant SYS as System/BBD
    participant OP as Order Processing
    participant PP as Payment Processor
    participant DB as PostgreSQL
    participant AG as API Gateway

    SYS->>OP: TOPIC_CANCEL_BOOKING_BY_SYSTEM
    
    Note over OP: Event Consumer (no group)
    
    OP->>DB: Get Order
    
    alt Wallet Payment with Reserved Balance
        OP->>PP: RefundWallet(reservedAmount)
    end
    
    alt PreAuth CC
        OP->>PP: ReleasePreAuth()
    end
    
    OP->>DB: Update Order (state: CANCELLED, reason: SYSTEM)
    
    OP-->>AG: Notify Order Changes
```

---

## 🔄 Payment Callback Flows

### Update Charge Status

```mermaid
sequenceDiagram
    participant PP as Payment Processor
    participant OP as Order Processing
    participant DB as PostgreSQL
    participant AG as API Gateway

    PP->>OP: TOPIC_UPDATE_CHARGE_STATUS
    
    Note over OP: Event Consumer (no group)
    
    OP->>DB: Get Order by ID
    
    OP->>DB: Update charge_status
    
    alt Charge Success
        OP->>OP: Calculate Points
        OP->>DB: Update Points
    end
    
    OP-->>AG: Notify Order Changes
```

### Update Refund Status

```mermaid
sequenceDiagram
    participant PP as Payment Processor
    participant OP as Order Processing
    participant DB as PostgreSQL

    PP->>OP: TOPIC_UPDATE_REFUND_STATUS
    
    OP->>DB: Get Order by ID
    
    OP->>DB: Update refund_status
    
    alt Refund Failed + Can Retry
        OP->>OP: Schedule Retry
    end
```

---

## 🔄 Rating Flow

```mermaid
sequenceDiagram
    participant AG as API Gateway
    participant OP as Order Processing
    participant DB as PostgreSQL

    AG->>OP: TOPIC_RATE_DRIVER
    
    OP->>DB: Get Order
    
    alt Order State != TRIP_ENDED
        OP-->>AG: Error: Invalid State
    end
    
    OP->>DB: Update Rating (stars, feedback)
    
    alt Rating < 4 Stars
        OP->>OP: Request Detailed Feedback
    end
    
    OP->>DB: Update Order (state: COMPLETED)
    
    OP-->>AG: Rating Saved
```

---

## 🔄 Trip History Flow

```mermaid
sequenceDiagram
    participant AG as API Gateway
    participant OP as Order Processing
    participant DB as PostgreSQL

    AG->>OP: TOPIC_GET_TRIP_HISTORY_V5
    
    OP->>DB: Query Orders (with pagination)
    Note over DB: Filter by user_id, service_type<br/>Order by created_at DESC
    
    DB-->>OP: Orders List
    
    OP->>OP: Transform to Response
    
    OP-->>AG: Trip History Response
```

---

## 🔗 Related Documentation

- [[README|Order Processing Service]]
- [[dependencies|Dependencies]]
- [[kafka-topics|Kafka Topics Reference]]
- [[api-reference|API Reference]]

---

#mrg #service #flows #order-processing

---

*Last Updated*: 2026-01-29
