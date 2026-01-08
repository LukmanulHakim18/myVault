# UPG Marketing Migration Tasks

**Team**: UPG (Universal Payment Gateway) - Marketing Integration  
**Status**: ✅ Completed  
**Migration Date**: 2025  
**Lead**: Erwin

---

## 📋 Overview

Marketing service migration tasks untuk memastikan promotional features dan user balance features tetap berjalan dengan baik setelah migrasi ke Huawei Cloud.

---

## 🎯 Migration Objectives

1. **Maintain Marketing Features**: Promo codes, campaigns, referrals
2. **User Balance Management**: Need-to-Cash (N2C) feature handling
3. **Payment Method Restrictions**: Temporary cash-only during migration
4. **Error Handling**: Wrap marketing errors with MyBB formatter
5. **Whitelisting**: Support untuk gradual rollout via headers

---

## ✅ Completed Tasks

### 1. Routing Header Enhancement
**Status**: ✅ Done (Dev)  
**Owner**: Erik  
**Description**: Add routing header untuk user whitelisting

#### Implementation
```go
// Middleware to check user whitelist for HWC routing
func HuaweiRoutingMiddleware(next http.Handler) http.Handler {
    return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
        userID := getUserIDFromContext(r.Context())
        
        // Check if user is whitelisted for Huawei Cloud
        if isWhitelisted(userID) {
            // Add header to route to HWC
            r.Header.Set("X-Route-Target", "huawei")
            r.Header.Set("huawei", "true")
        } else {
            // Route to GCP (during gradual migration)
            r.Header.Set("X-Route-Target", "gcp")
        }
        
        next.ServeHTTP(w, r)
    })
}
```

#### Usage
```http
GET /api/v6/payment/methods HTTP/1.1
Host: api.mybluebird.com
Authorization: Bearer <token>
X-Route-Target: huawei
huawei: true
```

**Benefits**:
- Gradual rollout capability
- A/B testing support
- Easy rollback per user
- Controlled migration

---

### 2. Payment Method Restriction
**Status**: ✅ Done (Dev)  
**Owner**: Eko  
**Description**: Update payment list to cash-only during migration window

#### Implementation
```go
// Payment method filter during migration
func (s *PaymentService) GetAvailablePaymentMethods(ctx context.Context, userID string) ([]PaymentMethod, error) {
    // Check if migration maintenance mode is active
    if s.featureFlags.IsEnabled("payment.migration_maintenance") {
        // Return only cash payment
        return []PaymentMethod{
            {
                ID:          "cash",
                Type:        "cash",
                Name:        "Cash",
                Description: "Pay with cash to driver",
                Icon:        "cash_icon.png",
                IsEnabled:   true,
                Priority:    1,
            },
        }, nil
    }
    
    // Normal flow - return all available methods
    return s.getAllPaymentMethods(ctx, userID)
}
```

#### API Response During Migration
```json
{
  "payment_methods": [
    {
      "id": "cash",
      "type": "cash",
      "name": "Cash",
      "description": "Pay with cash to driver",
      "icon": "cash_icon.png",
      "is_enabled": true,
      "priority": 1
    }
  ],
  "message": "During system maintenance, only cash payment is available. Other payment methods will be restored shortly."
}
```

**Benefits**:
- Reduced complexity during migration
- No payment gateway issues
- Simplified transaction flow
- Easy to revert

---

### 3. N2C Feature Toggle
**Status**: ✅ Done (Dev)  
**Owner**: Erwin  
**Description**: Disable Need-to-Cash feature during migration by adding markup to user's balance

#### Background: N2C Feature
Need-to-Cash (N2C) allows users to mark their balance for cash withdrawal:
- User requests "convert balance to cash"
- Next ride: driver pays user the balance in cash
- Balance deducted from user account

#### Migration Issue
- Balance data syncing between GCP and HWC
- Risk of double withdrawal
- Transaction consistency concerns

#### Solution: Temporary Markup
```go
// N2C service during migration
func (s *N2CService) ProcessN2CRequest(ctx context.Context, userID string, amount decimal.Decimal) error {
    // Check migration maintenance mode
    if s.featureFlags.IsEnabled("n2c.migration_maintenance") {
        // Add markup to user balance instead of processing withdrawal
        return s.addBalanceMarkup(ctx, userID, amount, "migration_hold")
    }
    
    // Normal N2C processing
    return s.processWithdrawal(ctx, userID, amount)
}

// Add markup to hold balance without actual withdrawal
func (s *N2CService) addBalanceMarkup(ctx context.Context, userID string, amount decimal.Decimal, reason string) error {
    markup := BalanceMarkup{
        UserID:    userID,
        Amount:    amount,
        Reason:    reason,
        Status:    "pending",
        CreatedAt: time.Now(),
    }
    
    // Store markup for post-migration processing
    return s.markupRepo.Create(ctx, markup)
}
```

#### Post-Migration Processing
```go
// After migration, process pending markups
func (s *N2CService) ProcessPendingMarkups(ctx context.Context) error {
    markups, err := s.markupRepo.GetPending(ctx)
    if err != nil {
        return err
    }
    
    for _, markup := range markups {
        // Process each pending N2C request
        if err := s.processWithdrawal(ctx, markup.UserID, markup.Amount); err != nil {
            log.Error().Err(err).Str("user_id", markup.UserID).Msg("Failed to process pending N2C")
            continue
        }
        
        // Mark as processed
        markup.Status = "completed"
        s.markupRepo.Update(ctx, markup)
    }
    
    return nil
}
```

**Benefits**:
- No lost N2C requests
- Data consistency maintained
- User balance protected
- Clear audit trail

---

### 4. Marketing Error Handling
**Status**: 🔄 In Progress  
**Owner**: Erwin  
**Description**: Wrap marketing service errors with MyBB error formatter

#### Problem
Marketing service errors tidak konsisten dengan MyBB error format:
```json
// Marketing service error (inconsistent)
{
  "error": "promo code validation failed",
  "code": 500
}
```

#### Solution: Error Wrapper
```go
// Error wrapper for marketing service
type MarketingError struct {
    OriginalError error
    Code          string
    Message       string
    Details       map[string]interface{}
}

// Convert marketing errors to MyBB format
func (e *MarketingError) ToMyBBError() *MyBBError {
    return &MyBBError{
        Code:    e.Code,
        Message: e.Message,
        Type:    "marketing_error",
        Details: e.Details,
        Timestamp: time.Now(),
    }
}

// Example usage in handler
func (h *PromoHandler) ValidatePromoCode(w http.ResponseWriter, r *http.Request) {
    promoCode := r.URL.Query().Get("code")
    
    // Call marketing service
    valid, err := h.marketingClient.ValidatePromoCode(promoCode)
    if err != nil {
        // Wrap error
        marketingErr := &MarketingError{
            OriginalError: err,
            Code:          "PROMO_VALIDATION_FAILED",
            Message:       "Unable to validate promotional code",
            Details: map[string]interface{}{
                "promo_code": promoCode,
                "reason":     err.Error(),
            },
        }
        
        // Return in MyBB format
        w.Header().Set("Content-Type", "application/json")
        w.WriteHeader(http.StatusBadRequest)
        json.NewEncoder(w).Encode(marketingErr.ToMyBBError())
        return
    }
    
    // Success response
    // ...
}
```

#### MyBB Error Format
```json
{
  "error": {
    "code": "PROMO_VALIDATION_FAILED",
    "message": "Unable to validate promotional code",
    "type": "marketing_error",
    "details": {
      "promo_code": "BBMERDEKA",
      "reason": "service unavailable"
    },
    "timestamp": "2025-08-03T06:55:32Z"
  }
}
```

**Benefits**:
- Consistent error format
- Better error tracking
- Improved user experience
- Easier debugging

---

## 🔍 Assessment: State Change Handling

### Backend State Assessment Tasks

Erwin melakukan assessment untuk berbagai state changes di marketing features:

#### State -3: Promo Code Expired
**Assessment**: ✅ Completed  
**Behavior**: Handle expired promo codes gracefully
```go
func (s *PromoService) ValidatePromoCode(code string) (*PromoCode, error) {
    promo, err := s.repo.GetByCode(code)
    if err != nil {
        return nil, err
    }
    
    // State -3: Expired
    if promo.ExpiryDate.Before(time.Now()) {
        return nil, &PromoError{
            State:   -3,
            Message: "Promotional code has expired",
            Code:    code,
        }
    }
    
    return promo, nil
}
```

#### State -2: Promo Code Invalid/Inactive
**Assessment**: ✅ Completed  
**Behavior**: Reject invalid or deactivated promo codes
```go
// State -2: Invalid or inactive
if !promo.IsActive {
    return nil, &PromoError{
        State:   -2,
        Message: "Promotional code is not active",
        Code:    code,
    }
}
```

#### State 0: Promo Code Pending Validation
**Assessment**: ✅ Completed  
**Behavior**: Hold order while validating promo
```go
// State 0: Pending validation
if promo.RequiresManualApproval {
    return &PromoValidation{
        State:   0,
        Status:  "pending",
        Message: "Promotional code requires approval",
    }, nil
}
```

#### State 1: Promo Code Validated
**Assessment**: ✅ Completed  
**Behavior**: Apply discount to order
```go
// State 1: Validated and ready
return &PromoValidation{
    State:      1,
    Status:     "validated",
    Discount:   promo.DiscountAmount,
    PromoType:  promo.Type,
    ValidUntil: promo.ExpiryDate,
}, nil
```

#### State 4: Promo Code Applied to Order
**Assessment**: ✅ Completed  
**Behavior**: Confirm promo application
```go
// State 4: Applied to order
order.PromoCodeID = promo.ID
order.DiscountAmount = promo.DiscountAmount
order.State = 4  // Promo applied
return s.orderRepo.Update(order)
```

#### State 6: Promo Code Fully Redeemed
**Assessment**: ✅ Completed  
**Behavior**: Complete promo lifecycle
```go
// State 6: Redeemed/completed
promo.UsageCount++
promo.LastUsedAt = time.Now()

if promo.UsageCount >= promo.MaxUsage {
    promo.State = 6  // Fully redeemed
    promo.IsActive = false
}

return s.promoRepo.Update(promo)
```

---

## 📊 Testing & Validation

### Test Scenarios

#### 1. Whitelisted User Journey
```bash
# Test with whitelisted user
curl -X POST "https://api.mybluebird.com/api/v6/orders" \
  -H "Authorization: Bearer $WHITELISTED_USER_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
    "service_type": "blue_bird_taxi",
    "pickup": {...},
    "destination": {...}
  }'

# Expected: Request routed to HWC with header "huawei: true"
```

#### 2. Cash-Only Payment
```bash
# Test payment methods during migration
curl -X GET "https://api.mybluebird.com/api/v6/me/payment_methods" \
  -H "Authorization: Bearer $TOKEN"

# Expected response:
{
  "payment_methods": [
    {
      "id": "cash",
      "type": "cash",
      "name": "Cash"
    }
  ],
  "message": "During system maintenance, only cash payment is available."
}
```

#### 3. N2C During Migration
```bash
# Test N2C request during migration
curl -X POST "https://api.mybluebird.com/api/v6/balance/need-to-cash" \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
    "amount": 50000
  }'

# Expected: Balance marked up, not immediately processed
{
  "status": "pending",
  "message": "Your request is being processed and will be completed after system maintenance",
  "reference_id": "markup_12345"
}
```

#### 4. Marketing Error Format
```bash
# Test promo code with invalid code
curl -X POST "https://api.mybluebird.com/api/v6/promo/validate" \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
    "promo_code": "INVALID"
  }'

# Expected: MyBB formatted error
{
  "error": {
    "code": "PROMO_NOT_FOUND",
    "message": "Promotional code not found",
    "type": "marketing_error",
    "details": {
      "promo_code": "INVALID"
    }
  }
}
```

---

## 🐛 Known Issues & Workarounds

### Issue: Marketing Service 500 Error
**Reported In**: [Issue BB00215513](./issues/issue-BB00215513.md)  
**Status**: Under Investigation  
**Workaround**: 
- Retry logic implemented
- Fallback to no-promo order creation
- User notification of promo unavailability

---

## 📋 Post-Migration Cleanup

### Tasks After Migration Complete

1. **Remove Feature Flags**:
   ```bash
   # Disable migration-specific flags
   - payment.migration_maintenance
   - n2c.migration_maintenance
   - marketing.validation_bypass
   ```

2. **Process Pending N2C Markups**:
   ```go
   // Run batch job to process pending markups
   go run scripts/process_pending_n2c.go
   ```

3. **Restore Full Payment Methods**:
   ```go
   // Re-enable all payment methods
   s.featureFlags.Disable("payment.migration_maintenance")
   ```

4. **Remove Routing Headers**:
   ```go
   // Remove HWC routing logic after full migration
   // Keep as permanent feature for A/B testing (optional)
   ```

5. **Validate Marketing Integration**:
   ```bash
   # Test all promo codes
   # Verify campaign tracking
   # Check analytics data
   ```

---

## 📞 Contacts

### Team Assignments
- **Erik**: Routing header implementation
- **Eko**: Payment method restrictions
- **Erwin**: N2C handling, marketing error wrapper, state assessments

### Escalation
- **Technical Lead**: Lukmanul Hakim
- **Product Owner**: [Product Manager]
- **Support**: UPG Team Channel

---

## 📚 Related Documentation

- [Main Migration README](./README.md)
- [Issue BB00215513 - Marketing Error](./issues/issue-BB00215513.md)
- [UPG Architecture](../UPG/01-architecture/)
- [Marketing API Integration](../UPG/03-apis/contracts/marketing-api.md)

---

## 🏷️ Tags

#upg #marketing #payment #n2c #promo-codes #migration #feature-flags

---

*Last Updated: 2025-08-04*  
*Status: Completed*  
*Owner: UPG Team (Erik, Eko, Erwin)*
