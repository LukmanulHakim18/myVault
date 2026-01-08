# Incident: EzPay Race Condition dengan IOT Event

**Incident ID**: BB01267851  
**Date**: 4 Agustus 2025, 03:44-04:36 WIB  
**Severity**: MEDIUM  
**Status**: ✅ Resolved (Escalated to BBD)  
**Related Migration**: GCP to Huawei Cloud Migration

---

## 📋 Summary

User mengalami multiple failed EzPay attempts karena race condition antara EzPay request dari customer dan IOT event "start argo" dari driver. Customer melakukan request EzPay sebelum IOT event diterima oleh sistem, menyebabkan validation failure dengan pesan "Please make sure your driver is already login and argometer has been activated".

---

## 👤 User Information

```
Name   : Abrar Dia
BB ID  : BB01267851
Email  : abrar_dia@yahoo.co.id
Phone  : +6287877382431
```

---

## 🚗 Vehicle Information

```
Car Number : PD2879
Service    : EzPay
State      : Street Hailing (saat incident)
```

---

## 🔍 Incident Details

### Order Timeline

| Order ID | State | Time (WIB) | Status | Note |
|----------|-------|------------|--------|------|
| 109552891 | -2 | 04:14:50 | Failed | 6th attempt |
| 109552240 | -2 | 04:13:14 | Failed | 5th attempt |
| 109552173 | 2 | 03:53:57 | **Success** | 4th attempt ✅ |
| 109551979 | -2 | 03:47:41 | Failed | 3rd attempt |
| 109551935 | -2 | 03:46:13 | Failed | 2nd attempt |
| 109551885 | 2 | 03:45:57 | **Success** | 1st attempt ✅ |

**Pattern**: 6 attempts dalam 30 menit, dengan 2 success dan 4 failures (alternating pattern).

### Error Log
```json
{
  "log": "[Error] 2025/08/04 03:44:54.453824356 order_controller.go:1735 [event=retry_easy_ride_booking_error] [message=Please make sure your driver is already login and argometer has been activated] [topic=retry_easy_ride_booking] [user_id=BB01267851] [device_id=5a78f48118ad0bc3] [order_id=109551885] [payload=...]",
  "stream": "stdout",
  "time": "2025-08-03T20:44:54.453877982Z"
}
```

---

## 🔬 Root Cause Analysis

### Primary Cause: Race Condition

**From BBD Team Investigation**:

1. **Vehicle State**: Mobil dalam keadaan **street hailing** (tidak dalam order)
2. **Customer Action**: EzPay request diterima jam **03:44 WIB**
3. **IOT Event Delay**: Event "start argo" tercatat di BBD jam **04:36 WIB**
4. **Time Gap**: **52 menit** antara request dan IOT event

### Race Condition Scenario

```
Timeline:
03:44 - Customer scan QR code dan request EzPay
03:44 - System cek: apakah argo sudah start? ❌ NO
03:44 - Validation FAILED: "Please make sure driver is login and argo activated"
03:45 - Customer retry #2 → ✅ SUCCESS (driver mungkin manual start argo)
03:46 - Customer retry #3 → ❌ FAILED
03:47 - Customer retry #4 → ❌ FAILED
03:53 - Customer retry #5 → ✅ SUCCESS
04:13 - Customer retry #6 → ❌ FAILED
04:14 - Customer retry #7 → ❌ FAILED
04:36 - IOT event "start argo" finally received by BBD system
```

### Contributing Factors

1. **IOT Event Reliability**
   - IOT tidak reliable mengirim event real-time
   - Network delay atau device issue
   - 52 menit delay adalah unacceptable

2. **Validation Logic**
   - System terlalu strict require IOT event BEFORE EzPay
   - No fallback mechanism
   - No grace period untuk IOT event arrival

3. **Driver Behavior**
   - Driver mungkin lupa start argo secara manual
   - Driver dalam street hailing mode tanpa proper argo activation

4. **User Experience**
   - Error message tidak jelas untuk user
   - No guidance untuk retry atau contact support
   - Multiple retries causing frustration

---

## 💡 Impact Assessment

### Customer Impact
- **Severity**: MEDIUM
- **Users Affected**: 1 confirmed (potentially more with street hailing)
- **Service**: EzPay (street hailing)
- **Duration**: ~30 menit (multiple failed attempts)
- **Financial**: Possible lost ride if customer gives up

### Driver Impact
- Multiple order creation attempts dapat confusing driver
- State changes dapat cause UI glitches di driver app

### Business Impact
- Poor EzPay user experience
- Trust issue dengan EzPay reliability
- Possible customer churn

### Technical Impact
- IOT event timing reliability concern
- Race condition dalam order creation flow
- Validation logic terlalu strict

---

## 🛠️ Resolution

### Immediate Actions (by BBD Team)
1. ✅ Investigate IOT device for PD2879
2. ✅ Verify IOT event transmission reliability
3. ✅ Confirm vehicle state and argo status
4. ✅ Root cause: IOT event delay 52 menit

### Recommended Technical Solutions

#### Solution 1: Add Grace Period (Short-term)
```go
// order_validation.go

const IOT_GRACE_PERIOD = 2 * time.Minute

func (v *Validator) ValidateArgoStatus(ctx context.Context, carNumber string) error {
    // Check IOT event
    iotEvent, err := v.iotClient.GetLatestArgoEvent(ctx, carNumber)
    if err == nil && iotEvent.IsArgoStarted {
        return nil // IOT confirms argo is started
    }
    
    // Fallback: Check last known state with grace period
    lastOrder, err := v.orderRepo.GetLastSuccessfulOrder(ctx, carNumber)
    if err == nil && time.Since(lastOrder.CreatedAt) < IOT_GRACE_PERIOD {
        // Recent successful order, likely argo is still active
        log.Warn("Using grace period for argo validation", 
            "car_number", carNumber,
            "last_order_time", lastOrder.CreatedAt)
        return nil
    }
    
    return errors.New("Please make sure your driver's argometer is activated")
}
```

#### Solution 2: Async IOT Validation (Long-term)
```go
// Decouple IOT validation dari order creation critical path

func (uc *UseCase) CreateEzPayOrder(ctx context.Context, req *dto.EzPayRequest) (*dto.Order, error) {
    // Create order first (optimistic)
    order, err := uc.createOrder(ctx, req)
    if err != nil {
        return nil, err
    }
    
    // Validate IOT async
    go func() {
        if err := uc.validateIOTAsync(order.ID, req.CarNumber); err != nil {
            // Cancel order if IOT validation fails
            uc.cancelOrder(order.ID, "IOT validation failed")
            // Notify customer
            uc.notifyCustomer(order.CustomerID, "ezpay_cancelled", err)
        }
    }()
    
    return order, nil
}
```

#### Solution 3: Improve Error Messaging
```go
// Better user-facing error messages

type ValidationError struct {
    Code    string
    Message string
    Actions []string // Suggested actions
}

func NewArgoNotStartedError() *ValidationError {
    return &ValidationError{
        Code: "ARGO_NOT_STARTED",
        Message: "Driver's meter is not active yet.",
        Actions: []string{
            "Please ask the driver to start the meter",
            "Wait a moment and try again",
            "Contact customer support if issue persists",
        },
    }
}
```

---

## 📈 Monitoring & Alerts

### New Metrics to Track
1. **IOT Event Lag**
   - Time between order creation and IOT event arrival
   - Alert if lag > 5 minutes

2. **EzPay Failure Rate**
   - Track failures by error type
   - Alert if "argo not started" > 5% of EzPay orders

3. **Retry Pattern**
   - Track users with multiple failed attempts
   - Identify problematic vehicles/drivers

### Dashboard Widgets
```
- EzPay Success Rate (target: >95%)
- IOT Event Lag P50/P95/P99
- Failed Orders by Reason
- Vehicles with High Failure Rate
```

---

## 🔄 Prevention Measures

### Short-term (1-2 weeks)
1. ✅ Escalate to BBD for IOT reliability improvement
2. [ ] Add grace period logic untuk argo validation
3. [ ] Improve error messaging untuk customers
4. [ ] Create runbook untuk EzPay IOT issues
5. [ ] Monitor IOT event lag closely

### Medium-term (1-2 months)
1. [ ] Implement async IOT validation pattern
2. [ ] Add fallback mechanism (manual verification by driver)
3. [ ] Create automated tests untuk race condition scenarios
4. [ ] Improve driver app to show real-time argo status

### Long-term (3-6 months)
1. [ ] IOT infrastructure upgrade (with BBD)
2. [ ] Move to event-driven architecture untuk IOT events
3. [ ] Implement predictive validation (ML-based argo status prediction)
4. [ ] Consider alternative argo detection methods (camera, GPS, etc.)

---

## 🤝 Stakeholder Communication

### Customer Support
- Update FAQ dengan EzPay argo issue
- Train CS team untuk handle similar complaints
- Provide script untuk customer guidance

### Driver Operations
- Communicate importance of starting argo properly
- Add reminder in driver app
- Track drivers with frequent "argo not started" issues

### BBD Team
- Share incident report
- Request IOT reliability improvement
- Schedule quarterly review meeting for IOT performance

---

## 📚 Related Documents

- `02-Work/Architecture-Office/initiatives/gcp-to-hwc-migration.md` - Main migration doc
- `02-Work/Teams/MRG/02-services/booking-service/` - Booking service docs
- `02-Work/Teams/MRG/05-runbooks/ezpay-troubleshooting.md` - EzPay runbook (TODO)
- External: BBD IOT System Documentation

---

## 👥 People Involved

**Reported by**: Customer (Abrar Dia)  
**Investigated by**: Platform Team (Lukmanul Hakim)  
**Root Cause Analysis**: BBD Team  
**Stakeholders**: Product Team, Driver Operations, Customer Support

---

## 📋 Action Items

- [ ] **BBD Team**: Improve IOT event reliability (52 min delay unacceptable)
- [ ] **Platform Team**: Implement grace period logic untuk argo validation
- [ ] **Mobile Team**: Improve error messaging UI
- [ ] **Driver Ops**: Communication campaign untuk drivers about argo activation
- [ ] **Customer Support**: Update knowledge base dengan troubleshooting guide

---

**Document Created**: 4 Agustus 2025  
**Last Updated**: 4 Agustus 2025  
**Status**: Under Monitoring (Escalated to BBD)

**Tags**: #incident #ezpay #race-condition #iot #street-hailing #bbd #driver-app #mrg
