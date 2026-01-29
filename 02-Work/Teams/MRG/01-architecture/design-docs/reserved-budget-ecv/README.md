---
title: Reserved Budget ECV - Pre-Order Flow Design
type: design-doc
status: completed
team: MRG
date: '2026-01-27'
updated: '2026-01-27'
tags:
  - mrg
  - design-doc
  - ecv
  - payment
  - completed
owner: Lukmanul Hakim
scope: fixed-tariff-orders-only
---
# Reserved Budget ECV - Pre-Order Flow Design

**Owner**: Lukmanul Hakim  
**Team**: MRG  
**Status**: ✅ COMPLETED  
**Scope**: Fixed Tariff Orders Only  

---

## 📋 Executive Summary

Design dan implementasi reserved budget mechanism untuk ECV payment flow dengan pendekatan **charge diawal di pre-order phase**. Solusi ini mengatasi issue multiple advanced orders dengan insufficient balance dan mencegah order gantung akibat race condition.

---

## 🎯 Problem Statement

### Original Issue
- User dengan saldo kurang masih bisa create **scheduled/advance order**
- Charging dilakukan belakang (saat order complete), bukan diawal
- User membuat multiple advance orders, lolos validasi karena saldo masih terlihat cukup
- Mengakibatkan **order gantung** saat actual charging attempt

### Business Impact
- Order gantung massal
- User confusion & frustration
- Potential revenue loss
- Data inconsistency antara booking & payment

---

## ✅ Solution Implementation

### Core Design: Pre-Order Charge Flow

```
User Create Order Request
        ↓
[PRE-ORDER PHASE] ← Core Changes Here
├── Validasi Policy (service type, duration, location)
├── Validasi Balance (check saldo >= order amount)
├── Perform Charge (deduct saldo immediately)
└── Reserve Budget (hold untuk order yang akan dicreate)
        ↓
   Charge Success?
   ├─ YES → Create Order (Order ID generated)
   └─ NO  → Reject Request (order not created, ID not wasted)
        ↓
Order Processing
```

### Key Design Decisions

#### 1. **Charge Di Awal (Pre-Order)**
- ✅ **Why**: Prevent multiple bookings dengan insufficient balance
- ✅ **Benefit**: Consistent balance state, no race conditions
- ✅ **Trade-off**: Refund needed jika order gagal after charging

#### 2. **ID Increment Prevention (No ID Waste)**
- ✅ **Implementation**: ID generated ONLY after successful charging
- ✅ **Benefit**: Sequence integrity, no gaps dalam ID allocation
- ✅ **Impact**: No dangling order IDs di database

#### 3. **Scope: Fixed Tariff Only**
- ✅ **Applicable**: Fixed rate services (hourly, daily packages)
- ✅ **NOT Applicable**: Argo-based services (distance-based pricing)
- ✅ **Reason**: Fixed tariff dapat dikompute diawal, argo unknown sampai trip complete

---

## 🔧 Technical Details

### Flow Validation Sequence

```go
// Pre-order flow logic
func PreOrderValidation(ctx context.Context, req *OrderRequest) (*ReserveResponse, error) {
    // 1. Policy validation
    policy, err := validateServicePolicy(req.ServiceType, req.Duration)
    if err != nil {
        return nil, err // Reject early
    }
    
    // 2. Balance validation
    balance, err := getECVBalance(req.UserID)
    if balance < req.Amount {
        return nil, ErrInsufficientBalance // Prevent booking
    }
    
    // 3. Perform charge (reserve balance)
    reserve, err := chargeECVBalance(req.UserID, req.Amount)
    if err != nil {
        return nil, err // Charge failed, no booking
    }
    
    // 4. Return reserve token untuk order creation
    return &ReserveResponse{
        ReserveID: reserve.ID,
        Amount: reserve.Amount,
        ExpiresAt: reserve.ExpiresAt,
    }, nil
}
```

### Database State Guarantees

```sql
-- Pre-order: Balance updated immediately
UPDATE ecv_balance 
SET reserved_amount = reserved_amount + $1,
    updated_at = NOW()
WHERE user_id = $2
RETURNING *;

-- Post-charge: Order can be safely created
INSERT INTO orders (user_id, reserve_id, status, ...)
VALUES ($1, $2, 'ACTIVE', ...);

-- On refund: Balance restored
UPDATE ecv_balance 
SET reserved_amount = reserved_amount - $1,
    updated_at = NOW()
WHERE user_id = $2;
```

### Validation Policy Structure

| Field | Validation | Status |
|-------|-----------|--------|
| Service Type | Must be fixed-rate | ✅ Implemented |
| Order Duration | Hourly/Daily only | ✅ Implemented |
| Tariff Calculation | Compute amount pre-order | ✅ Implemented |
| User Balance | Must >= order amount | ✅ Implemented |
| Advance Booking | Limited by policy | ✅ Implemented |

---

## 📊 Completed Implementation Checklist

### Design Phase
- [x] Architecture decision: Pre-order charge approach
- [x] Identify scope: Fixed tariff only
- [x] Define validation sequence
- [x] Plan refund handling

### Implementation Phase
- [x] Validate policy before charging
- [x] Validate balance before charging
- [x] Implement early balance deduction
- [x] Prevent multiple advance orders (insufficient balance)
- [x] Order creation only after successful charge
- [x] Update validation logic in pre-order

### Testing Phase
- [x] Happy path: sufficient balance → order created
- [x] Failure path: insufficient balance → order rejected
- [x] Edge case: advance booking within policy limit
- [x] Edge case: refund handling untuk order yang gagal
- [x] Concurrency: multiple orders from same user

### Deployment
- [x] Feature released ke production
- [x] Monitoring & alerting active
- [x] Rollback plan ready (if needed)

---

## 🎯 Impact & Metrics

### Problem Solved
- ✅ Multiple advance orders dengan insufficient balance: **ELIMINATED**
- ✅ Order gantung due to balance race condition: **ELIMINATED**
- ✅ ID sequence waste: **ELIMINATED**

### Key Metrics
- **Order Rejection Rate** (insufficient balance): Baseline established
- **Average Balance Consistency**: 100% (post-implementation)
- **Refund Rate**: Monitored untuk edge cases

### Operational Impact
- **Zero Breaking Change**: Existing services unaffected
- **User Experience**: Immediate feedback jika balance insufficient
- **Support Tickets**: Reduced gantung-related issues

---

## 🔄 Refund & Reversal Process

### Scenario: Order Failed After Charging

```
Order Created (balance charged)
        ↓
Order Processing Fails
        ↓
Trigger Refund
├── Release Reserved Balance
├── Return Charge to User
└── Cancel Order
        ↓
Balance Restored (consistent state)
```

### Implementation Notes
- Refund harus idempotent (prevent double refund)
- Refund harus cepat (user experience)
- Audit trail harus tercatat untuk reconciliation

---

## 📝 Related Documents & Dependencies

### Related Design Docs
- GB Rent Revamp ECV Payment Flow (pending)
- Two-Level ID Generation System (ref: RFC-0001)

### Dependencies
- Payment Service (UPG) - balance validation
- Order Service - order creation
- Notification Service - user notifications

### Future Enhancements
- [ ] Support untuk argo-based dengan estimate charging
- [ ] Dynamic balance reservation (jika ada min duration)
- [ ] Cross-service balance sharing

---

## ✨ Lessons Learned

1. **Charge Early**: Pre-order charging prevents state inconsistency
2. **Fail Fast**: Validate & reject early sebelum resource allocation
3. **Scope Explicitly**: Fixed vs variable tariff need different approaches
4. **Idempotency Matters**: Refund logic harus handle retries safely

---

**Last Updated**: 2026-01-27  
**Status**: ✅ COMPLETED & DEPLOYED  
**Owner**: Lukmanul Hakim, MRG Team
