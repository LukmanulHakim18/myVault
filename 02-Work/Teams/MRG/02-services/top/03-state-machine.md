# State Machine - Order Lifecycle

## Overview
TOP service menggunakan **State Pattern** untuk mengelola order lifecycle. Setiap state memiliki behavior dan transition rules yang berbeda, ensuring valid state progression dan preventing invalid transitions.

## State Machine Architecture

### Design Pattern
```go
type OrderState interface {
    // State transition methods
    UpdateIncomingStateLookingForDriver(ctx, req) (*model.Order, error)
    UpdateIncomingStateEnRoute(ctx, req) (*model.Order, error)
    UpdateIncomingStateCanceled(ctx, req) (*model.Order, error)
    UpdateIncomingStateOnTrip(ctx, req) (*model.Order, error)
    UpdateIncomingStateTracking(ctx, req) (*model.Order, error)
    UpdateIncomingStateDropOff(ctx, req) (*model.Order, error)
    UpdateIncomingStateDropOffEcv(ctx, req) (*model.Order, error)
    UpdateIncomingStateTaxiNoShow(ctx, req) (*model.Order, error)
    UpdateIncomingStateNoTaxi(ctx, req) (*model.Order, error)
    UpdateIncomingStateChatRoomCreated(ctx, req) (*model.Order, error)
    UpdateIncomingStateOrderFailed(ctx, req) (*model.Order, error)
    UpdateIncomingStateOrderTimeout(ctx, req) (*model.Order, error)
    
    // Order operations
    CancelOrder(ctx, req) error
    RateOrder(ctx, req) error
    RetryOrder(ctx, req) (*model.Order, error)
    Reschedule(ctx, req) (*model.Order, error)
    HideOrder(ctx, req) error
}
```

### State Processor
```go
type OrderStateProcess struct {
    // All state implementations
    orderTimeOut     OrderState  // State -3
    orderInitial     OrderState  // State -2
    orderFailed      OrderState  // State -1
    lookingForDriver OrderState  // State 0
    enRoute          OrderState  // State 1
    canceled         OrderState  // State 2
    noTaxi           OrderState  // State 3
    onTrip           OrderState  // State 4
    customerNoShow   OrderState  // State 5
    tracking         OrderState  // State 6
    ecvDropOff       OrderState  // State 7
    dropOff          OrderState  // State 8
    offlineDispatch  OrderState  // State 9
    chatRoomCreated  OrderState  // State 10
    
    currentState     OrderState  // Current active state
    Order            *model.Order
    Repo             *repository.Repository
}
```

---

## Complete State Diagram

```
                    ┌─────────────────────────────────┐
                    │    Order Creation               │
                    │    (CreateOrder API)            │
                    └──────────────┬──────────────────┘
                                   │
                                   ▼
                        ┌──────────────────┐
                        │  ORDER_INITIAL   │
                        │    (State -2)    │
                        └─────────┬────────┘
                                  │
                     ┌────────────┼────────────┐
                     │            │            │
                     ▼            ▼            ▼
              ┌──────────┐ ┌────────────┐ ┌──────────┐
              │ORDER     │ │LOOKING FOR │ │ORDER     │
              │FAILED    │ │DRIVER      │ │TIMEOUT   │
              │(State -1)│ │(State 0)   │ │(State -3)│
              └──────────┘ └─────┬──────┘ └──────────┘
                [TERMINAL]       │         [TERMINAL]
                                 │
                    ┌────────────┼────────────┬────────────┐
                    │            │            │            │
                    ▼            ▼            ▼            ▼
              ┌─────────┐  ┌─────────┐ ┌─────────┐ ┌──────────┐
              │EN ROUTE │  │NO TAXI  │ │CANCELED │ │TAXI NO   │
              │(State 1)│  │(State 3)│ │(State 2)│ │SHOW (5)  │
              └────┬────┘  └─────────┘ └─────────┘ └──────────┘
                   │        [TERMINAL]  [TERMINAL]  [TERMINAL]
                   │
                   ├────────────┬────────────┐
                   │            │            │
                   ▼            ▼            ▼
              ┌─────────┐ ┌─────────┐ ┌──────────┐
              │ON TRIP  │ │CANCELED │ │TAXI NO   │
              │(State 4)│ │(State 2)│ │SHOW (5)  │
              └────┬────┘ └─────────┘ └──────────┘
                   │       [TERMINAL]  [TERMINAL]
                   │
                   ▼
              ┌─────────┐
              │TRACKING │
              │(State 6)│
              └────┬────┘
                   │
                   ├────────────┐
                   │            │
                   ▼            ▼
              ┌─────────┐ ┌──────────┐
              │ECV DROP │ │DROP OFF  │
              │OFF (7)  │ │(State 8) │
              └─────────┘ └──────────┘
              [TERMINAL]  [TERMINAL]

Additional Special States:
┌──────────────┐  ┌──────────────┐
│OFFLINE       │  │CHAT ROOM     │
│DISPATCH (9)  │  │CREATED (10)  │
└──────────────┘  └──────────────┘
[TERMINAL]        [SPECIAL]
```

---

## State Details

### Initial States

#### State -3: ORDER_TIMEOUT
**Description**: Order creation timeout (no response dari iTOP dalam waktu tertentu).

**Entry Conditions**:
- Timeout dari ORDER_INITIAL state
- Tidak ada response dari iTOP dalam timeout window

**Allowed Operations**:
- ❌ Cannot cancel
- ❌ Cannot rate
- ❌ Cannot reschedule
- ✅ Can retry (RetryOrder)

**Terminal State**: Yes

**Example Scenario**:
```
User creates order → Submit to iTOP → No response after 30s → ORDER_TIMEOUT
```

---

#### State -2: ORDER_INITIAL
**Description**: Initial state setelah order creation, menunggu dispatch ke iTOP.

**Entry Conditions**:
- Order just created via CreateOrder API
- Validation passed
- Payment pre-auth approved (if applicable)

**Allowed Transitions**:
- → LOOKING_FOR_DRIVER (0) - Successfully submitted to iTOP
- → ORDER_FAILED (-1) - Submission failed
- → ORDER_TIMEOUT (-3) - Timeout

**Allowed Operations**:
- ❌ Cannot cancel (too early)
- ❌ Cannot rate
- ❌ Cannot reschedule

**Duration**: Usually < 5 seconds

---

#### State -1: ORDER_FAILED
**Description**: Order creation failed (rejected by iTOP atau internal error).

**Entry Conditions**:
- iTOP rejection
- Internal validation failure
- Payment pre-auth declined

**Allowed Operations**:
- ❌ Cannot cancel (already failed)
- ❌ Cannot rate
- ❌ Cannot reschedule
- ✅ Can retry (RetryOrder)

**Terminal State**: Yes

**Failure Reasons**:
- Area not covered
- Invalid service type
- Payment pre-auth declined
- iTOP system error

---

### Active States

#### State 0: LOOKING_FOR_DRIVER
**Description**: Mencari driver available untuk order.

**Entry Conditions**:
- Successfully submitted to iTOP
- Payment pre-auth approved (if CC)

**Allowed Transitions**:
- → EN_ROUTE (1) - Driver assigned dan on the way
- → NO_TAXI (3) - No driver available
- → CANCELED (2) - User cancelled atau system timeout
- → TAXI_NO_SHOW (5) - Driver assigned tapi no-show
- → CHAT_ROOM_CREATED (10) - Chat room established

**Allowed Operations**:
- ✅ Can cancel (CancelOrder)
- ❌ Cannot rate (not completed)
- ✅ Can reschedule (Reschedule) - for future bookings only
- ❌ Cannot hide

**Duration**: Usually 1-5 minutes

**What Happens**:
1. Order submitted ke iTOP dispatch system
2. System mencari driver terdekat yang available
3. Driver receives order notification
4. Driver accepts → transition ke EN_ROUTE
5. No driver accepts dalam timeout → transition ke NO_TAXI

**User Experience**:
```
Mobile App shows:
- "Looking for driver..."
- Searching animation
- Estimated waiting time
- Cancel button enabled
```

---

#### State 1: EN_ROUTE
**Description**: Driver assigned dan on the way ke pickup location.

**Entry Conditions**:
- Driver accepted order
- Driver info available (name, phone, vehicle)

**Allowed Transitions**:
- → ON_TRIP (4) - Driver arrived, passenger picked up
- → CANCELED (2) - User cancelled
- → TAXI_NO_SHOW (5) - Driver no-show

**Allowed Operations**:
- ✅ Can cancel (CancelOrder) - with possible cancellation fee
- ❌ Cannot rate (not completed)
- ❌ Cannot reschedule (driver already assigned)
- ❌ Cannot hide

**Driver Info Available**:
```go
{
  "driver_id": "DRV001",
  "driver_name": "Pak Driver",
  "driver_phone": "+628123456789",
  "driver_photo": "https://cdn.bluebird.id/drivers/001.jpg",
  "driver_rating": 4.8,
  "vehicle_number": "B 1234 ABC",
  "vehicle_model": "Toyota Camry"
}
```

**Real-time Updates**:
- Driver location (GPS tracking)
- ETA to pickup location
- Driver can call passenger
- Passenger can call driver

**User Experience**:
```
Mobile App shows:
- Driver info card
- Live driver location on map
- ETA countdown
- Call driver button
- Cancel button (may incur fee)
```

**Duration**: Varies based on distance, typically 5-30 minutes

---

#### State 4: ON_TRIP
**Description**: Trip in progress, passenger picked up.

**Entry Conditions**:
- Driver arrived at pickup location
- Passcode verified (optional)
- Trip started

**Allowed Transitions**:
- → TRACKING (6) - Normal trip progression
- → CANCELED (2) - Trip cancelled mid-trip (rare)

**Allowed Operations**:
- ❌ Cannot cancel (trip started)
- ❌ Cannot rate (not completed)
- ❌ Cannot reschedule
- ❌ Cannot hide

**What Happens**:
- Trip fare calculation starts (meter or fixed fare)
- GPS tracking active
- Route recording
- Driver navigates to dropoff location

**User Experience**:
```
Mobile App shows:
- Live trip tracking
- Current route on map
- Estimated arrival time
- Running fare (if metered)
- Driver info
```

**Duration**: Varies based on distance and traffic

---

#### State 6: TRACKING
**Description**: Trip nearing completion, preparing for dropoff.

**Entry Conditions**:
- Trip approaching destination
- Driver navigating to dropoff point

**Allowed Transitions**:
- → DROP_OFF (8) - Normal trip completion
- → ECV_DROP_OFF (7) - ECV trip completion (special voucher handling)

**Allowed Operations**:
- ❌ Cannot cancel
- ❌ Cannot rate (not completed)
- ❌ Cannot reschedule
- ❌ Cannot hide

**What Happens**:
- Final fare calculation
- Payment charging initiated
- Driver prepares to end trip

**Duration**: Usually 1-5 minutes

---

### Terminal States

#### State 2: CANCELED
**Description**: Order cancelled (by user, driver, or system).

**Entry Conditions**:
- User cancelled via CancelOrder API
- Driver cancelled
- System cancelled (timeout, no response)

**Cancelled By**:
```go
const (
    CancelByCustomer = 1
    CancelBySystem   = 2
    CancelByDriver   = 3
)
```

**Cancellation Fees**:
```
LOOKING_FOR_DRIVER: No fee
EN_ROUTE (< 5 min): No fee
EN_ROUTE (> 5 min): Rp 10.000
ON_TRIP: Based on distance travelled
```

**Soft Ban Tracking**:
```
IF user cancels ≥ 5 orders within 2 hours
THEN soft ban for 24 hours
STORED IN: soft_ban_users table
```

**Allowed Operations**:
- ❌ Cannot cancel (already cancelled)
- ❌ Cannot rate
- ❌ Cannot reschedule
- ✅ Can hide (HideTrip)

**Terminal State**: Yes

---

#### State 3: NO_TAXI
**Description**: No driver available untuk order.

**Entry Conditions**:
- Timeout di LOOKING_FOR_DRIVER state
- All nearby drivers busy atau offline
- No driver accepts dalam timeout window

**Allowed Operations**:
- ❌ Cannot cancel (already failed)
- ❌ Cannot rate
- ❌ Cannot reschedule
- ✅ Can retry (RetryOrder)
- ✅ Can hide (HideTrip)

**Terminal State**: Yes

**User Experience**:
```
Mobile App shows:
- "No driver available"
- Suggestion to retry
- Alternative service types
- Retry button
```

---

#### State 5: TAXI_NO_SHOW (Customer No Show)
**Description**: Driver arrived tapi passenger tidak muncul.

**Entry Conditions**:
- Driver marked as "arrived" at pickup location
- Passenger tidak muncul dalam waiting time (usually 5-10 minutes)
- Driver reports no-show

**Allowed Operations**:
- ❌ Cannot cancel
- ❌ Cannot rate
- ❌ Cannot reschedule
- ✅ Can hide (HideTrip)

**Terminal State**: Yes

**Cancellation Fee**: Usually applied (Rp 15.000 - Rp 25.000)

---

#### State 7: ECV_DROP_OFF
**Description**: Trip completed dengan ECV (Electronic Corporate Voucher).

**Entry Conditions**:
- Trip completed
- Payment method = ECV
- Special voucher handling required

**What's Different from Regular DROP_OFF**:
- Voucher validation at completion
- Special billing to corporate account
- Different payment flow

**Allowed Operations**:
- ❌ Cannot cancel (already completed)
- ✅ Can rate (RateOrder)
- ❌ Cannot reschedule
- ✅ Can hide (HideTrip)

**Terminal State**: Yes

**Payment Flow**:
```
1. Trip completed
2. Validate ECV voucher
3. Charge to corporate account
4. Generate invoice
5. Send receipt
```

---

#### State 8: DROP_OFF
**Description**: Trip completed successfully (normal completion).

**Entry Conditions**:
- Driver arrived at dropoff location
- Trip ended by driver
- Fare calculated

**Allowed Operations**:
- ❌ Cannot cancel (already completed)
- ✅ Can rate (RateOrder) - with optional tips
- ❌ Cannot reschedule
- ✅ Can hide (HideTrip)

**Terminal State**: Yes

**Payment Processing**:
```
Cash:
  - No charging needed
  - Mark as paid

E-wallet (GoPay, DANA, OVO, ShopeePay, TCash):
  1. Charge e-wallet account
  2. Handle insufficient balance
  3. Update charge_status
  4. Send receipt

Credit Card:
  1. Capture pre-auth amount
  2. Adjust if fare different
  3. Update charge_status
  4. Send receipt
```

**Final Fare Components**:
```go
FinalFare = TripFare 
          + ExtraFees (handling_fee, airport_fee, etc)
          - PromoDiscount
          + Tips
```

**User Experience**:
```
Mobile App shows:
- Trip completed
- Fare breakdown
- Rate driver prompt
- Add tips option
- Receipt available
```

---

#### State 9: OFFLINE_DISPATCH
**Description**: Order dispatched via offline system (special case).

**Entry Conditions**:
- Manual dispatch by operator
- iTOP offline
- Emergency fallback

**Allowed Operations**:
- Limited operations
- Manual intervention required

**Terminal State**: Yes

**Rare State**: Only used in special circumstances

---

### Special States

#### State 10: CHAT_ROOM_CREATED
**Description**: Chat room established between passenger and driver.

**Entry Conditions**:
- Driver assigned
- Chat service initialized
- Room ID received from chat provider (Qiscus)

**Not a State Transition**:
- This is more of a "flag" state
- Can occur during LOOKING_FOR_DRIVER atau EN_ROUTE
- Doesn't replace current state

**What Happens**:
```
1. Driver assigned
2. Create chat room in chat service
3. Get room ID
4. Store in chat_rooms table
5. Enable chat in mobile app
```

**Allowed Operations**:
- All operations from parent state
- Additional: Chat messaging enabled

---

## State Transition Rules

### Transition Matrix

| From State | To State | Trigger | Conditions |
|------------|----------|---------|------------|
| ORDER_INITIAL (-2) | LOOKING_FOR_DRIVER (0) | iTOP accepts | Submit success |
| ORDER_INITIAL (-2) | ORDER_FAILED (-1) | iTOP rejects | Validation failed |
| ORDER_INITIAL (-2) | ORDER_TIMEOUT (-3) | Timeout | No response |
| LOOKING_FOR_DRIVER (0) | EN_ROUTE (1) | Driver accepts | Driver assigned |
| LOOKING_FOR_DRIVER (0) | NO_TAXI (3) | Timeout | No driver available |
| LOOKING_FOR_DRIVER (0) | CANCELED (2) | User/system cancels | Cancel request |
| LOOKING_FOR_DRIVER (0) | TAXI_NO_SHOW (5) | Driver no-show | Driver no-show |
| EN_ROUTE (1) | ON_TRIP (4) | Driver starts trip | Pickup confirmed |
| EN_ROUTE (1) | CANCELED (2) | User cancels | Cancel request |
| EN_ROUTE (1) | TAXI_NO_SHOW (5) | Driver no-show | Driver no-show |
| ON_TRIP (4) | TRACKING (6) | Nearing destination | Trip progressing |
| ON_TRIP (4) | CANCELED (2) | Emergency cancel | Rare case |
| TRACKING (6) | DROP_OFF (8) | Driver ends trip | Normal completion |
| TRACKING (6) | ECV_DROP_OFF (7) | Driver ends trip | ECV payment |

### Forbidden Transitions

These transitions are **explicitly forbidden** and will return error:

```
❌ DROP_OFF → any state (terminal)
❌ CANCELED → any state (terminal)
❌ NO_TAXI → any state except RETRY (terminal)
❌ ON_TRIP → LOOKING_FOR_DRIVER (cannot go backwards)
❌ DROP_OFF → EN_ROUTE (cannot go backwards)
❌ Any terminal → non-terminal state (except via RetryOrder)
```

**Error Response**:
```json
{
  "code": "TOP-4005",
  "message": "Forbidden state transition",
  "current_state": 8,
  "requested_state": 1
}
```

---

## Operation Availability Matrix

| Operation | States Where Allowed | Notes |
|-----------|---------------------|-------|
| **CancelOrder** | 0, 1 | Fee may apply if state=1 |
| **RateOrder** | 7, 8 | Only terminal completed states |
| **HideTrip** | 2, 3, 5, 7, 8 | Only terminal states |
| **RetryOrder** | -3, -1, 3 | Only failed states |
| **Reschedule** | 0 | Only future bookings in state 0 |

### Cancel Order Rules

```go
func (state) CanCancel() bool {
    allowedStates := []int{
        STATE_LOOKING_FOR_DRIVER, // 0
        STATE_EN_ROUTE,            // 1
    }
    return contains(allowedStates, currentState)
}

func (state) GetCancellationFee() float64 {
    switch currentState {
    case STATE_LOOKING_FOR_DRIVER:
        return 0  // No fee
    case STATE_EN_ROUTE:
        if timeInState < 5*time.Minute {
            return 0  // No fee if < 5 minutes
        }
        return 10000  // Rp 10.000
    default:
        return 0
    }
}
```

### Rate Order Rules

```go
func (state) CanRate() bool {
    allowedStates := []int{
        STATE_DROP_OFF,     // 8
        STATE_ECV_DROP_OFF, // 7
    }
    return contains(allowedStates, currentState)
}

func (state) ValidateRating(rating int32, tips float64) error {
    if rating < 1 || rating > 5 {
        return errors.New("rating must be 1-5")
    }
    if tips > 0 && tips < 1000 {
        return errors.New("tips minimum Rp 1.000")
    }
    return nil
}
```

---

## Implementation Details

### State Files

Each state has dedicated implementation file:

```
pkg/state/
├── base_state.go                    # Base state struct
├── must_implemented.go              # Default implementations (returns forbidden)
├── order_state.go                   # Interface definition
├── state_-3_order_time_out.go      # State -3
├── state_-2_order_initial.go       # State -2
├── state_-1_order_failed.go        # State -1
├── state_0_looking_for_driver.go   # State 0 (most complex)
├── state_1_en_route.go             # State 1
├── state_2_order_canceled.go       # State 2
├── state_3_no_taxi.go              # State 3
├── state_4_on_trip.go              # State 4
├── state_5_customer_no_show.go     # State 5
├── state_6_tracking.go             # State 6
├── state_7_ecv_drop_off.go         # State 7
├── state_8_drop_off.go             # State 8
├── state_9_dispatcher_offline.go   # State 9
├── state_10_chat_room_created.go   # State 10
└── unknown.go                       # Unknown state handler
```

### State Initialization

```go
func NewOrderStateProcess(order *model.Order, repo *repository.Repository) *OrderStateProcess {
    process := &OrderStateProcess{
        Order: order,
        Repo:  repo,
    }
    
    // Initialize all states
    process.orderInitial = &OrderInitial{OrderStateProcess: process}
    process.lookingForDriver = &LookingForDriver{OrderStateProcess: process}
    process.enRoute = &EnRoute{OrderStateProcess: process}
    process.onTrip = &OnTrip{OrderStateProcess: process}
    process.tracking = &Tracking{OrderStateProcess: process}
    process.dropOff = &DropOff{OrderStateProcess: process}
    // ... initialize other states
    
    // Set current state based on order.StateEnum
    process.setCurrentState(order.StateEnum)
    
    return process
}
```

### State Transition Example

```go
// Example: Transition from LOOKING_FOR_DRIVER to EN_ROUTE
func (s *LookingForDriver) UpdateIncomingStateEnRoute(
    ctx context.Context, 
    req *contract.NotifyPartnerEvent,
) (*model.Order, error) {
    // 1. Update order with driver info
    s.OrderStateProcess.Order.DriverID = req.Driver.Nip
    s.OrderStateProcess.Order.DriverName = req.Driver.Name
    s.OrderStateProcess.Order.VehicleNumber = req.Vehicle.VehicleNo
    s.OrderStateProcess.Order.StateEnum = STATE_EN_ROUTE
    
    // 2. Save to database
    updatedOrder, err := s.OrderStateProcess.Repo.DB.Order.UpdateOrder(
        ctx, 
        *s.OrderStateProcess.Order,
    )
    if err != nil {
        return nil, err
    }
    
    // 3. Update state machine
    s.OrderStateProcess.setState(s.OrderStateProcess.enRoute)
    
    // 4. Send notifications
    s.OrderStateProcess.sendDriverAssignedNotification(ctx)
    
    // 5. Update Firebase for real-time
    s.OrderStateProcess.updateFirebase(ctx)
    
    // 6. Sync order summary
    s.OrderStateProcess.pushSyncOrderSummary(ctx, updatedOrder.ID)
    
    return updatedOrder, nil
}
```

---

## Best Practices

### 1. Always Validate State Before Operations

```go
func (uc *UseCase) CancelOrder(ctx context.Context, req *contract.CancelBookingRequest) error {
    // Get order
    order, err := uc.Repo.DB.Order.GetByID(ctx, req.OrderId)
    if err != nil {
        return err
    }
    
    // Initialize state machine
    stateProcess := NewOrderStateProcess(order, uc.Repo)
    
    // Delegate to current state
    return stateProcess.CancelOrder(ctx, req)
}
```

### 2. Use MustImplemented for Default Behavior

```go
type MustImplemented struct{}

// Default implementation returns Forbidden error
func (*MustImplemented) RateOrder(ctx context.Context, req *contract.RateOrderRequest) error {
    return errors.New("operation not allowed in current state")
}

// States override only allowed operations
type DropOff struct {
    *MustImplemented  // Inherit defaults
}

func (d *DropOff) RateOrder(ctx context.Context, req *contract.RateOrderRequest) error {
    // Custom implementation for DROP_OFF state
    return d.rateOrderImpl(ctx, req)
}
```

### 3. State Transitions Should Be Atomic

```go
func (s *State) transitionTo(ctx context.Context, newState OrderState) error {
    tx, err := s.Repo.DB.BeginTx(ctx)
    if err != nil {
        return err
    }
    defer tx.Rollback()
    
    // 1. Update order
    err = tx.Order.UpdateState(ctx, s.Order.ID, newState)
    if err != nil {
        return err
    }
    
    // 2. Log state change
    err = tx.OrderStateLog.Create(ctx, log)
    if err != nil {
        return err
    }
    
    // 3. Commit transaction
    return tx.Commit()
}
```

### 4. Always Send Notifications After State Change

```go
func (s *State) afterTransition(ctx context.Context) {
    // Send push notification
    s.sendPushNotification(ctx)
    
    // Update Firebase
    s.updateFirebase(ctx)
    
    // Trigger webhook (if configured)
    s.triggerWebhook(ctx)
    
    // Sync order summary
    s.syncOrderSummary(ctx)
}
```

---

## Troubleshooting

### Common Issues

#### State Stuck in LOOKING_FOR_DRIVER
**Symptoms**: Order tidak berubah state setelah lama
**Causes**:
- iTOP not responding
- Network issues
- Driver pool empty

**Solutions**:
```go
// Check iTOP connection
iTOP.HealthCheck()

// Retry dispatch
RetryOrder(orderId)

// Manual transition (admin only)
ForceTransitionState(orderId, STATE_NO_TAXI)
```

---

#### Payment Charging Failed in DROP_OFF
**Symptoms**: Order completed tapi charge_status = "failed"
**Causes**:
- E-wallet insufficient balance
- Credit card declined
- Payment gateway timeout

**Solutions**:
```go
// Retry charging
AdjustChargeStatus(orderId, "charging")

// Check payment status
PaymentProcessor.GetPaymentStatus(paymentId)

// Manual adjustment (if needed)
UpdateChargeStatus(orderId, "success", finalFare)
```

---

#### Cannot Cancel Order
**Symptoms**: Cancel request returns forbidden error
**Cause**: Order in state yang tidak allow cancel

**Check**:
```bash
# Get current state
curl -X POST /v1/orders/get-order \
  -d '{"order_id": 123456}'

# Verify cancellation rules
if state == ON_TRIP || state == TRACKING || state == DROP_OFF:
    return "Cannot cancel - trip in progress or completed"
```

---

## Monitoring & Metrics

### Key Metrics

```prometheus
# State distribution
top_order_state_current{state="0"} 150    # LOOKING_FOR_DRIVER
top_order_state_current{state="1"} 80     # EN_ROUTE
top_order_state_current{state="4"} 120    # ON_TRIP

# State transition duration
top_state_transition_duration_seconds{from="0",to="1"} 180.5
top_state_transition_duration_seconds{from="1",to="4"} 420.3

# Terminal state distribution
top_orders_terminal_state_total{state="8"} 5420   # DROP_OFF (success)
top_orders_terminal_state_total{state="2"} 320    # CANCELED
top_orders_terminal_state_total{state="3"} 180    # NO_TAXI
```

### Alerts

```yaml
# Long time in LOOKING_FOR_DRIVER
- alert: OrderStuckLookingForDriver
  expr: top_state_duration_seconds{state="0"} > 600
  severity: warning
  
# High cancellation rate
- alert: HighCancellationRate
  expr: rate(top_orders_terminal_state_total{state="2"}[5m]) > 0.2
  severity: critical
```

---

## Related Documentation
- [[01-overview|Service Overview]]
- [[02-api-reference|API Reference]]
- [[04-payment-flows|Payment Flows]]
- [[05-dependencies|Dependencies]]

---
**Last Updated**: 2025-01-07
**State Machine Version**: v2.0
**Total States**: 13