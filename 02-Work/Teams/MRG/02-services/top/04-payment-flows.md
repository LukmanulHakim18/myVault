# Payment Flows - Taxi Order Processor

## Overview
TOP service mengelola complete payment lifecycle untuk orders, dari pre-authorization hingga final charging. Service terintegrasi dengan **UPG (Universal Payment Gateway)** untuk payment processing.

## Payment Methods Supported

| Payment Method | Code | Pre-Auth | Auto-Charge | Notes |
|----------------|------|----------|-------------|-------|
| **Cash** | `cash` | ❌ | ❌ | No payment processing |
| **Credit Card** | `credit_card` | ✅ | ✅ | Requires pre-auth |
| **GoPay** | `gopay` | ❌ | ✅ | E-wallet |
| **DANA** | `dana` | ❌ | ✅ | E-wallet |
| **OVO** | `ovo` | ❌ | ✅ | E-wallet |
| **ShopeePay** | `shopeepay` | ❌ | ✅ | E-wallet |
| **TCash** | `tcash` | ❌ | ✅ | E-wallet |
| **ECV** | `ecv_*` | ❌ | ✅ | Corporate voucher |

---

## Payment Lifecycle

### Complete Flow Diagram

```
┌─────────────────────────────────────────────────────────────┐
│                    ORDER CREATION                            │
└───────────────────────┬─────────────────────────────────────┘
                        │
                        ▼
              ┌─────────────────┐
              │ Payment Method? │
              └────────┬────────┘
                       │
        ┌──────────────┼──────────────┬──────────────┐
        │              │              │              │
        ▼              ▼              ▼              ▼
   ┌────────┐   ┌──────────┐  ┌──────────┐  ┌──────────┐
   │  CASH  │   │CREDIT CARD│  │ E-WALLET │  │   ECV    │
   └───┬────┘   └─────┬────┘  └────┬─────┘  └────┬─────┘
       │              │             │             │
       │              ▼             │             │
       │        ┌──────────┐        │             │
       │        │ PRE-AUTH │        │             │
       │        └─────┬────┘        │             │
       │              │             │             │
       │         ┌────┴────┐        │             │
       │         │         │        │             │
       │    ┌────▼───┐ ┌───▼────┐   │             │
       │    │APPROVED│ │DECLINED│   │             │
       │    └────┬───┘ └───┬────┘   │             │
       │         │         │        │             │
       │         │      ┌──▼────────▼─────────────▼───┐
       │         │      │   ORDER CANCELLED/FAILED    │
       │         │      └─────────────────────────────┘
       │         │
       └─────────┴───────────┬─────────────────────────┘
                             │
                             ▼
                    ┌────────────────┐
                    │ TRIP IN PROGRESS│
                    └────────┬────────┘
                             │
                             ▼
                    ┌────────────────┐
                    │  TRIP COMPLETED │
                    │  (DROP_OFF)     │
                    └────────┬────────┘
                             │
        ┌────────────────────┼────────────────────┐
        │                    │                    │
        ▼                    ▼                    ▼
   ┌────────┐         ┌──────────┐        ┌──────────┐
   │  CASH  │         │CREDIT CARD│        │ E-WALLET │
   │ (PAID) │         │ CAPTURE  │        │  CHARGE  │
   └────────┘         └─────┬────┘        └────┬─────┘
                            │                   │
                       ┌────┴────┐         ┌────┴────┐
                       │         │         │         │
                  ┌────▼───┐ ┌───▼────┐ ┌──▼────┐ ┌──▼────┐
                  │SUCCESS │ │ FAILED │ │SUCCESS│ │FAILED │
                  └────────┘ └────────┘ └───────┘ └───────┘
```

---

## Payment Method Details

### 1. Cash Payment

**Flow**:
```
1. User selects cash payment
2. Order created (no payment processing)
3. Trip completed
4. Driver collects cash
5. Order marked as "paid"
```

**Characteristics**:
- ✅ No pre-auth required
- ✅ No charging process
- ✅ Simple flow
- ❌ No payment tracking
- ❌ Manual cash handling

**Final State**:
```json
{
  "payment_method": "cash",
  "payment_status": "paid",
  "charge_status": "success",
  "outstanding": 0,
  "paid_amount": 55000,
  "final_fare": 55000
}
```

---

### 2. Credit Card Payment

**Complete Flow**:

#### Step 1: Pre-Authorization
**When**: Before order dispatch (during CreateOrder)

**Process**:
```go
1. Calculate estimated fare:
   estimatedFare = fareEstimate.Max * 1.2  // 20% buffer

2. Call UPG PreAuth:
   req := &ppContract.PreAuthRequest{
       UserId:            user.InternalID,
       PaymentIdentifier: creditCard.Token,
       Amount:            estimatedFare,
       OrderId:           order.ID,
   }
   
3. Wait for callback:
   - Approved → Continue order creation
   - Declined → Cancel order, return error
```

**Pre-Auth Callback**:
```go
// Callback dari UPG
PreAuthOrderCallback(request) {
    if request.PreAuthStatus == "approved" {
        // Continue order dispatch
        dispatchToiTOP(order)
    } else {
        // Cancel order
        cancelOrder(order, "Payment declined")
    }
}
```

**Database State**:
```sql
INSERT INTO orders_pre_auth (
    order_id,
    payment_identifier,
    pre_auth_amount,
    pre_auth_status,
    created_at
) VALUES (
    123456,
    'card_token_abc',
    66000,  -- 55000 * 1.2
    'approved',
    NOW()
);
```

#### Step 2: Trip Completion
**When**: Order reaches DROP_OFF state

**Process**:
```go
1. Calculate final fare:
   finalFare = tripFare + extraFees - promoDiscount + tips

2. Compare with pre-auth:
   if finalFare <= preAuthAmount {
       // Capture pre-auth amount
       captureAmount = finalFare
   } else {
       // Need additional charge
       captureAmount = preAuthAmount
       additionalCharge = finalFare - preAuthAmount
   }

3. Call UPG Capture:
   req := &ppContract.CaptureRequest{
       UserId:       user.InternalID,
       OrderId:      order.ID,
       Amount:       captureAmount,
       PreAuthId:    preAuth.ID,
   }
```

#### Step 3: Final Charging
**When**: Immediately after trip completion

**Scenarios**:

**A. Final Fare = Pre-Auth Amount**
```go
Capture: 66000
Additional Charge: 0
Status: SUCCESS
```

**B. Final Fare < Pre-Auth Amount**
```go
Capture: 55000
Refund: 11000 (66000 - 55000)
Status: SUCCESS
```

**C. Final Fare > Pre-Auth Amount**
```go
Capture: 66000
Additional Charge: 5000 (70000 - 66000)
Total Charged: 70000
Status: SUCCESS
```

**Database Updates**:
```sql
UPDATE taxi_orders SET
    charge_status = 'success',
    payment_status = 'paid',
    final_fare = 55000,
    paid_amount = 55000,
    outstanding = 0
WHERE id = 123456;
```

---

### 3. E-Wallet Payment (GoPay, DANA, OVO, ShopeePay, TCash)

**Flow**:

#### Step 1: Order Creation
**When**: CreateOrder API called

**Validations**:
```go
1. Check wallet balance:
   balance := PaymentProcessor.GetEwalletBalance(user, provider)
   
2. Validate sufficient balance:
   if balance < estimatedFare {
       return error("Insufficient balance")
   }

3. Check for existing active orders:
   activeOrders := GetActiveOrdersWithEwallet(user, provider)
   if len(activeOrders) > 0 {
       return error("Ongoing e-wallet order exists")
   }
```

**Error Codes**:
```go
TOP-4093  // Fare exceeds balance
TOP-4032  // Ongoing e-wallet order exists
```

#### Step 2: Trip Completion
**When**: Order reaches DROP_OFF state

**Charging Process**:
```go
1. Calculate final fare:
   finalFare = tripFare + extraFees - promoDiscount + tips

2. Lock order (prevent concurrent charging):
   redis.SetNX("charge:lock:#{orderId}", 1, 30*time.Second)

3. Update charge status to "charging":
   updateChargeStatus(order, "charging")

4. Call UPG Charge:
   req := &ppContract.ChargeRequest{
       UserId:       user.InternalID,
       OrderId:      order.ID,
       Provider:     paymentMethod,  // gopay, dana, etc
       Amount:       finalFare,
   }
   
5. Handle response:
   if success {
       updateChargeStatus(order, "success")
       outstanding = 0
   } else {
       updateChargeStatus(order, "failed", failureReason)
       outstanding = finalFare
   }

6. Release lock:
   redis.Del("charge:lock:#{orderId}")
```

#### Step 3: Handling Charge Failures

**Failure Scenarios**:

**A. Insufficient Balance**
```json
{
  "error_code": "INSUFFICIENT_BALANCE",
  "message": "E-wallet balance insufficient",
  "current_balance": 45000,
  "required_amount": 55000,
  "shortfall": 10000
}
```

**Actions**:
1. Mark order as outstanding
2. Send notification to user
3. Update charge_status = "failed"
4. Set failure_reason = "Insufficient balance"

**B. Provider Timeout**
```json
{
  "error_code": "PROVIDER_TIMEOUT",
  "message": "E-wallet provider not responding"
}
```

**Actions**:
1. Retry charging (max 3 attempts)
2. If still fails, mark as outstanding
3. Queue for manual review

**C. Account Restricted**
```json
{
  "error_code": "ACCOUNT_RESTRICTED",
  "message": "E-wallet account restricted"
}
```

**Actions**:
1. Mark as outstanding
2. Contact user via email/push
3. Provide alternative payment methods

#### Step 4: Retry Mechanism

**Auto Retry**:
```go
func RetryCharging(order *model.Order) {
    maxRetries := 3
    retryDelay := time.Minute * 5
    
    for i := 0; i < maxRetries; i++ {
        time.Sleep(retryDelay)
        
        err := ChargeEwallet(order)
        if err == nil {
            return // Success
        }
        
        log.Warn("Retry charging failed", 
            "order_id", order.ID,
            "attempt", i+1,
            "error", err)
    }
    
    // All retries failed
    NotifyOpsTeam("Manual intervention required", order.ID)
}
```

---

### 4. ECV (Electronic Corporate Voucher)

**Flow**:

#### Step 1: Voucher Validation
**When**: CreateOrder API called

**Process**:
```go
1. Validate ECV voucher:
   req := &ppContract.ValidateECVRequest{
       VoucherId:    voucherId,
       VoucherType:  voucherType,
       ServiceType:  serviceType,
       CarType:      carType,
       ValidateTime: pickupTime,
   }
   
   result := PaymentProcessor.ValidateECV(req)

2. Check voucher applicability:
   - Valid date range?
   - Valid service type?
   - Valid car type?
   - Usage limit not exceeded?

3. Lock voucher for this order:
   redis.SetNX("ecv:lock:#{voucherId}", orderId, 30*time.Minute)
```

**Validation Errors**:
```go
TOP-40011  // ECV not applicable for date
TOP-40012  // ECV not applicable for car type
TOP-40014  // Trip voucher validation failed
```

#### Step 2: Trip Completion
**When**: Order reaches ECV_DROP_OFF state (State 7)

**Charging Process**:
```go
1. Calculate final fare:
   finalFare = tripFare + extraFees

2. Charge to corporate account:
   req := &ppContract.ChargeECVRequest{
       VoucherId:     voucherId,
       OrderId:       order.ID,
       Amount:        finalFare,
       CompanyCode:   corporateAccount,
   }

3. Generate corporate invoice:
   invoice := GenerateInvoice(order, corporateAccount)

4. Update voucher usage:
   UpdateVoucherUsage(voucherId, order.ID)

5. Release voucher lock:
   redis.Del("ecv:lock:#{voucherId}")
```

**Special Handling**:
```
- No user payment required
- Billed to corporate account
- Different receipt format
- Corporate reporting required
```

---

## Charge Status States

### State Machine for Charging

```
NOT_CHARGED
    │
    ▼
CHARGING ────┐
    │        │
    │        │ (timeout/error)
    │        │
    ▼        ▼
SUCCESS   FAILED
```

### Status Definitions

| Status | Description | Final | Retryable |
|--------|-------------|-------|-----------|
| `NOT_CHARGED` | No charging attempted yet | ❌ | N/A |
| `CHARGING` | Charging in progress | ❌ | ❌ |
| `SUCCESS` | Successfully charged | ✅ | ❌ |
| `FAILED` | Charging failed | ✅ | ✅ |

### Status Updates

**Update Charge Status Flow**:
```go
func UpdateChargeStatus(
    orderId int64,
    status string,
    failureReason string,
) error {
    // Validate transition
    currentStatus := GetCurrentChargeStatus(orderId)
    if !isValidTransition(currentStatus, status) {
        return errors.New("invalid status transition")
    }
    
    // Update database
    tx := BeginTransaction()
    defer tx.Rollback()
    
    err := tx.Orders.UpdateChargeStatus(orderId, status, failureReason)
    if err != nil {
        return err
    }
    
    // Update payment_status based on charge_status
    if status == "success" {
        tx.Orders.UpdatePaymentStatus(orderId, "paid")
    } else if status == "failed" {
        tx.Orders.UpdatePaymentStatus(orderId, "pending")
    }
    
    // Commit
    return tx.Commit()
}
```

---

## Outstanding Payments

### Tracking Outstanding

**Database Schema**:
```sql
SELECT 
    order_id,
    payment_method,
    charge_status,
    final_fare,
    paid_amount,
    (final_fare - paid_amount) as outstanding
FROM taxi_orders
WHERE charge_status = 'failed'
  OR payment_status = 'pending';
```

### User Experience

**Has Outstanding**:
```json
{
  "has_outstanding": true,
  "outstanding_orders": [
    {
      "order_id": 123456,
      "outstanding_amount": 55000,
      "payment_method": "gopay",
      "failure_reason": "Insufficient balance",
      "can_retry": true
    }
  ],
  "total_outstanding": 55000
}
```

**Blocking Rules**:
```go
// Block new order creation if outstanding exists
func CanCreateOrder(userId string) error {
    outstanding := GetOutstandingPayments(userId)
    if outstanding.Total > 0 {
        return errors.New("Please settle outstanding payments first")
    }
    return nil
}
```

---

## Payment Callbacks

### AdjustChargeStatus Callback
**Called By**: UPG (Universal Payment Gateway)

**Purpose**: Update charge status setelah payment processing

**Request**:
```json
{
  "user_id": "user_123",
  "payment_method": "gopay",
  "order_id": 123456,
  "charge_status": "success",
  "failure_reason": "",
  "outstanding": 0,
  "trip_fare": 50000,
  "final_fare": 55000
}
```

**Process**:
```go
1. Validate request signature (security)
2. Get order from database
3. Update charge_status
4. Update outstanding amount
5. Send notification to user
6. Update Firebase
7. Sync order summary
```

---

### PreAuthOrderCallback
**Called By**: UPG (for credit card pre-auth)

**Purpose**: Update pre-auth status

**Request**:
```json
{
  "order_id": 123456,
  "pre_auth_status": "approved",
  "pre_auth_amount": 66000,
  "payment_identifier": "card_token_abc",
  "failure_reason": ""
}
```

**Process**:
```go
1. Get order from database
2. Update pre-auth status in orders_pre_auth table
3. If approved:
   - Continue order dispatch to iTOP
4. If declined:
   - Cancel order
   - Send notification
   - Refund any fees charged
```

---

## Fare Calculation

### Components

```go
type FareComponents struct {
    TripFare           float64  // Base fare from meter/fixed
    AdminFee           float64  // Service fee
    HandlingFee        float64  // Handling fee (if applicable)
    AirportFee         float64  // Airport surcharge
    ExtraTimeFee       float64  // Extra time charges
    TollFee            float64  // Toll road charges
    PromoDiscount      float64  // Promo discount
    Tips               float64  // Driver tips (optional)
    CancellationFee    float64  // If cancelled
}

func CalculateFinalFare(components FareComponents) float64 {
    total := components.TripFare
    total += components.AdminFee
    total += components.HandlingFee
    total += components.AirportFee
    total += components.ExtraTimeFee
    total += components.TollFee
    total -= components.PromoDiscount
    total += components.Tips
    total += components.CancellationFee
    
    return total
}
```

### Fare Storage

**order_fees table**:
```sql
CREATE TABLE order_fees (
    id SERIAL PRIMARY KEY,
    order_id BIGINT NOT NULL,
    fee_type VARCHAR(50) NOT NULL,  -- 'admin', 'handling', 'airport', etc
    fee_amount DECIMAL(10,2) NOT NULL,
    created_at TIMESTAMP DEFAULT NOW()
);

INSERT INTO order_fees VALUES
    (1, 123456, 'admin_fee', 5000),
    (2, 123456, 'handling_fee', 2000),
    (3, 123456, 'tips', 10000);
```

---

## Error Handling

### Common Payment Errors

#### 1. Insufficient Balance (E-wallet)
```json
{
  "code": "TOP-4093",
  "message": "E-wallet balance insufficient",
  "details": {
    "current_balance": 45000,
    "required_amount": 55000,
    "shortfall": 10000,
    "payment_method": "gopay"
  }
}
```

**Resolution**:
1. Notify user to top-up
2. Mark order as outstanding
3. Allow retry after top-up

---

#### 2. Credit Card Declined
```json
{
  "code": "TOP-4008",
  "message": "Credit card payment declined",
  "details": {
    "decline_reason": "Insufficient funds",
    "card_last_4": "1234"
  }
}
```

**Resolution**:
1. Cancel order
2. Suggest alternative payment method
3. No soft ban (payment issue, not user fault)

---

#### 3. Payment Gateway Timeout
```json
{
  "code": "TOP-5001",
  "message": "Payment gateway timeout",
  "details": {
    "provider": "gopay",
    "timeout_seconds": 30
  }
}
```

**Resolution**:
1. Retry charging (auto retry 3x)
2. If still fails, mark as outstanding
3. Alert ops team

---

#### 4. Concurrent Payment Limit
```json
{
  "code": "TOP-4031",
  "message": "Credit card concurrent limit reached",
  "details": {
    "active_orders": 3,
    "max_allowed": 3
  }
}
```

**Resolution**:
1. Block new order creation
2. Wait for existing orders to complete
3. Retry after completion

---

## Monitoring & Alerts

### Key Metrics

```prometheus
# Payment method distribution
top_payment_method_total{method="cash"} 3500
top_payment_method_total{method="gopay"} 2800
top_payment_method_total{method="credit_card"} 1200

# Charge success rate
top_charge_success_rate{method="gopay"} 0.98
top_charge_success_rate{method="credit_card"} 0.99

# Outstanding payments
top_outstanding_payments_total 45
top_outstanding_amount_total 2475000
```

### Alerts

```yaml
# High charge failure rate
- alert: HighChargeFailureRate
  expr: |
    rate(top_charge_failed_total[5m]) /
    rate(top_charge_attempts_total[5m]) > 0.05
  severity: warning

# Outstanding accumulation
- alert: HighOutstandingPayments
  expr: top_outstanding_amount_total > 5000000
  severity: critical
```

---

## Best Practices

### 1. Always Lock During Charging
```go
// Prevent concurrent charging
lockKey := fmt.Sprintf("charge:lock:%d", orderId)
acquired := redis.SetNX(lockKey, 1, 30*time.Second)
if !acquired {
    return errors.New("charging already in progress")
}
defer redis.Del(lockKey)
```

### 2. Idempotent Charging
```go
// Check if already charged
if order.ChargeStatus == "success" {
    return errors.New("order already charged successfully")
}
```

### 3. Comprehensive Logging
```go
log.Info("charging e-wallet",
    "order_id", orderId,
    "user_id", userId,
    "provider", paymentMethod,
    "amount", finalFare,
    "attempt", attemptNumber)
```

### 4. User-Friendly Error Messages
```go
// Localized error messages
if err == InsufficientBalance {
    return cLang.CustomTextByLanguage(ctx,
        "Insufficient e-wallet balance. Please top-up.",
        "Saldo e-wallet tidak cukup. Silakan top-up.")
}
```

---

## Related Documentation
- [[01-overview|Service Overview]]
- [[02-api-reference|API Reference]]
- [[03-state-machine|State Machine]]
- [[05-dependencies|Dependencies]]

---
**Last Updated**: 2025-01-07
**Payment Integration**: UPG v2
**Supported Payment Methods**: 8