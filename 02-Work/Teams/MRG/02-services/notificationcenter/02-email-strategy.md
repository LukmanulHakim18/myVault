# Email Strategy Pattern

## Overview
Package `emailstrategy` adalah implementasi Strategy Pattern untuk mengelola berbagai strategi pengiriman email berdasarkan **channel**, **service category**, dan **vehicle type** dengan dependency injection repository.

## Package Location
```
pkg/emailstrategy/
├── email_strategy.go                    # Manager & routing logic
├── README.md                            # Comprehensive documentation
├── ARCHITECTURE.md                      # Architecture design
├── IMPLEMENTATION_GUIDE.md              # Step-by-step guide
├── QUICK_REFERENCE.md                   # Quick lookup
├── INDEX.md                             # Navigation index
└── implementations/
    ├── base_strategy.go                 # Foundation & helper methods
    ├── strategy_types.go                # All strategy struct definitions
    ├── webreservation_ride_airport_transfer.go
    ├── send_email_v2_webreservation_rent.go
    ├── send_email_v2_mybb_ride.go
    └── send_email_v2_mybb_rent.go
```

## Core Concepts

### 1. EmailV2Strategy Interface
Contract untuk semua email strategy implementations:

```go
type EmailV2Strategy interface {
    OrderCreated(ctx, in) (*contract.DefaultResponse, error)
    PaymentReceived(ctx, in) (*contract.DefaultResponse, error)
    DriverAssigned(ctx, in) (*contract.DefaultResponse, error)
    DriverAssignedChanged(ctx, in) (*contract.DefaultResponse, error)
    EReceipt(ctx, in) (*contract.DefaultResponse, error)
    Reschedule(ctx, in) (*contract.DefaultResponse, error)
    CancelledBySystem(ctx, in) (*contract.DefaultResponse, error)
    CancelledByUser(ctx, in) (*contract.DefaultResponse, error)
}
```

### 2. BaseEmailStrategy
Foundation strategy yang menyediakan:
- **Default implementations** untuk semua method (returns `501 Not Implemented`)
- **Helper method** `SendEmailBBOneWithLog` untuk email + logging
- **Repository access** via embedded `Repo` field

```go
type BaseEmailStrategy struct {
    Repo *repository.Repository
}

// Default behavior: returns ErrFlowNotCovered
func (b *BaseEmailStrategy) OrderCreated(...) (*contract.DefaultResponse, error) {
    return &contract.DefaultResponse{
        Code:    http.StatusNotImplemented,
        Message: "OrderCreated: this flow is not covered by the current strategy",
    }, ErrFlowNotCovered
}

// Helper untuk email + logging
func (b *BaseEmailStrategy) SendEmailBBOneWithLog(
    ctx context.Context,
    data model.SendEmailBBOneReq,
    log model.EmailLog,
) error {
    // Send email via BBOne service
    // Store log to database
    // Handle errors consistently
}
```

**Benefits**:
- ✅ Eliminasi ~60% boilerplate code
- ✅ Consistent error handling untuk unsupported flows
- ✅ Centralized email + logging logic
- ✅ Direct repository access

### 3. EmailStrategyManager
Manager untuk routing strategy berdasarkan request:

```go
type EmailStrategyManager struct {
    repo *repository.Repository
}

func NewEmailStrategyManager(repo *repository.Repository) *EmailStrategyManager {
    return &EmailStrategyManager{repo: repo}
}

func (esm *EmailStrategyManager) GetEmailV2Strategy(
    channel string,
    request *contract.SendEmailV2Request,
) (EmailV2Strategy, error) {
    // Route berdasarkan channel, service, type
}
```

## Strategy Routing

### Routing Logic Flow
```
Request
  │
  ├─ Channel (WebReservation | MyBB)
  │   │
  │   ├─ Service Category (RIDE | RENT)
  │   │   │
  │   │   └─ Vehicle Type
  │   │       ├─ AIRPORT_TRANSFER (RIDE only)
  │   │       ├─ BLUEBIRD (RENT only)
  │   │       ├─ SILVERBIRD (RENT only)
  │   │       └─ GOLDENBIRD (RENT only)
  │   │
  │   └─ Strategy Instance
  │       └─ Execute Method
```

### Routing Table

| Channel | Service | Type | Strategy Class |
|---------|---------|------|----------------|
| WebReservation | RIDE | AIRPORT_TRANSFER | `WebReservationRideAirportTransferEmailStrategy` |
| WebReservation | RENT | BLUEBIRD | `WebReservationRentBluebirdEmailV2Strategy` |
| WebReservation | RENT | SILVERBIRD | `WebReservationRentSilverbirdEmailV2Strategy` |
| WebReservation | RENT | GOLDENBIRD | `WebReservationRentGoldenbirdEmailV2Strategy` |
| MyBB | RIDE | AIRPORT_TRANSFER | `MyBBRideAirportTransferEmailV2Strategy` |
| MyBB | RENT | BLUEBIRD | `MyBBRentBluebirdEmailV2Strategy` |
| MyBB | RENT | SILVERBIRD | `MyBBRentSilverbirdEmailV2Strategy` |
| MyBB | RENT | GOLDENBIRD | `MyBBRentGoldenbirdEmailV2Strategy` |

## Implementation Pattern

### Creating a New Strategy

#### Step 1: Define Struct (strategy_types.go)
```go
type NewChannelServiceTypeEmailStrategy struct {
    BaseEmailStrategy  // Always embed
    Repo *repository.Repository
}
```

#### Step 2: Implement Methods (new_strategy.go)
```go
package emailstrategyimplementation

// NewChannelServiceTypeEmailStrategy handles emails for [Channel] [Service]
//
// Supported flows:
// - OrderCreated ✅
// - PaymentReceived ✅
//
// Not supported (via BaseEmailStrategy):
// - DriverAssigned ❌
// - EReceipt ❌
// - Reschedule ❌
// - CancelledBySystem ❌
// - CancelledByUser ❌

func (s *NewChannelServiceTypeEmailStrategy) OrderCreated(
    ctx context.Context,
    in *contract.EmailV2OrderCreatedRequest,
) (*contract.DefaultResponse, error) {
    // 1. Prepare email data
    data := model.PayloadEmailOrderCreated{
        OrderID:      in.GetOrderId(),
        CustomerName: in.GetCustomerName(),
        // ... other fields
    }
    
    // 2. Parse template
    emailBuffer, err := util.ParseTemplateBBOne(template, data)
    if err != nil {
        return nil, err
    }
    
    // 3. Prepare email log
    emailLog := model.EmailLog{
        SentAt:       time.Now(),
        EmailAddress: in.GetEmail(),
        EmailSubject: "Order Confirmation",
        EmailContent: emailBuffer.String(),
    }
    
    // 4. Send email with logging using helper
    emailData := model.SendEmailBBOneReq{
        To:          []model.EmailRecipient{{Email: in.GetEmail()}},
        From:        model.EmailSender{Email: constants.NoReplyEmail},
        Subject:     "Order Confirmation",
        HtmlContent: emailBuffer.String(),
    }
    
    err = s.SendEmailBBOneWithLog(ctx, emailData, emailLog)
    if err != nil {
        return &contract.DefaultResponse{
            Code:    http.StatusInternalServerError,
            Message: err.Error(),
        }, err
    }
    
    return &contract.DefaultResponse{
        Code:    http.StatusOK,
        Message: constants.Success,
    }, nil
}

// Other methods NOT implemented - handled by BaseEmailStrategy automatically
```

#### Step 3: Update Router (email_strategy.go)
```go
func (esm *EmailStrategyManager) getNewChannelStrategy(
    request *contract.SendEmailV2Request,
) (EmailV2Strategy, error) {
    return &impl.NewChannelServiceTypeEmailStrategy{
        Repo: esm.repo,
        BaseEmailStrategy: impl.BaseEmailStrategy{Repo: esm.repo},
    }, nil
}
```

## Usage in Usecase

```go
func (u *UseCase) SendEmailV2(
    ctx context.Context,
    request *contract.SendEmailV2Request,
) (*contract.DefaultResponse, error) {
    // 1. Create strategy manager with repo injection
    manager := emailstrategy.NewEmailStrategyManager(u.Repo)
    
    // 2. Get strategy based on channel and request
    strategy, err := manager.GetEmailV2Strategy(
        request.GetChannel(),
        request,
    )
    if err != nil {
        return nil, err
    }
    
    // 3. Execute method based on email type
    switch request.GetEmailType() {
    case contract.EmailV2Type_ORDER_CREATED:
        if req, ok := request.Request.(*contract.SendEmailV2Request_OrderCreated); ok {
            return strategy.OrderCreated(ctx, req.OrderCreated)
        }
    case contract.EmailV2Type_PAYMENT_RECEIVED:
        if req, ok := request.Request.(*contract.SendEmailV2Request_PaymentReceived); ok {
            return strategy.PaymentReceived(ctx, req.PaymentReceived)
        }
    // ... other cases
    }
    
    return &contract.DefaultResponse{
        Code:    400,
        Message: "Invalid email type",
    }, nil
}
```

## Error Handling

### Handling Unsupported Flows
```go
response, err := strategy.OrderCreated(ctx, request)

// Check if flow not covered
if errors.Is(err, emailstrategyimplementation.ErrFlowNotCovered) {
    // This is expected - flow not supported by strategy
    log.Warn("Email flow not covered - graceful degradation",
        "channel", channel,
        "service", service,
        "type", emailType,
    )
    // Don't crash - return gracefully
    return response, nil
}

// Handle real errors
if err != nil {
    log.Error("Failed to send email", "error", err)
    return nil, err
}

return response, nil
```

## Available Strategies

### WebReservation Channel

| Service | Type | Methods Implemented | Status |
|---------|------|---------------------|--------|
| RIDE | AIRPORT_TRANSFER | OrderCreated, PaymentReceived, DriverAssigned, EReceipt | ✅ Partial |
| RENT | BLUEBIRD | OrderCreated, PaymentReceived | ✅ Partial |
| RENT | SILVERBIRD | OrderCreated, PaymentReceived | ✅ Partial |
| RENT | GOLDENBIRD | OrderCreated, PaymentReceived | ✅ Partial |

### MyBB Channel

| Service | Type | Methods Implemented | Status |
|---------|------|---------------------|--------|
| RIDE | AIRPORT_TRANSFER | OrderCreated, PaymentReceived, DriverAssigned | ✅ Partial |
| RENT | BLUEBIRD | OrderCreated, PaymentReceived | ✅ Partial |
| RENT | SILVERBIRD | OrderCreated, PaymentReceived | ✅ Partial |
| RENT | GOLDENBIRD | OrderCreated, PaymentReceived | ✅ Partial |

**Legend**:
- ✅ Partial: Implements core methods, others via BaseEmailStrategy

## Best Practices

### 1. Always Embed BaseEmailStrategy
```go
// ✅ CORRECT
type MyStrategy struct {
    BaseEmailStrategy  // Always embed for default behavior
    Repo *repository.Repository
}

// ❌ WRONG - Missing base strategy
type MyStrategy struct {
    Repo *repository.Repository
}
```

### 2. Document Supported Flows
```go
// MyStrategy handles emails for MyBB RIDE orders
//
// Supported flows:
// - OrderCreated ✅
// - PaymentReceived ✅
// - DriverAssigned ✅
//
// Not supported (via BaseEmailStrategy):
// - DriverAssignedChanged ❌
// - EReceipt ❌
// - Reschedule ❌
// - CancelledBySystem ❌
// - CancelledByUser ❌
type MyStrategy struct {
    BaseEmailStrategy
    Repo *repository.Repository
}
```

### 3. Use Helper Method for Email + Logging
```go
// ✅ GOOD - Use centralized helper
err := s.SendEmailBBOneWithLog(ctx, emailData, emailLog)

// ❌ BAD - Manual implementation
s.Repo.Bboneemail.SendEmail(ctx, emailData)
s.Repo.Db.CreateEmailLogs(ctx, logs)
// Prone to inconsistency and bugs
```

### 4. Override Only What You Need
```go
// ✅ GOOD - Only implement supported methods
func (s *MyStrategy) OrderCreated(...) { /* implementation */ }
func (s *MyStrategy) PaymentReceived(...) { /* implementation */ }
// Other methods automatically handled by BaseEmailStrategy

// ❌ BAD - Don't implement unsupported flows just to return errors
func (s *MyStrategy) EReceipt(...) {
    return nil, errors.New("not supported")
}
// Let BaseEmailStrategy handle this!
```

### 5. Consistent Error Handling Pattern
```go
// In usecase layer
response, err := strategy.SomeMethod(ctx, request)

// Always check for ErrFlowNotCovered first
if errors.Is(err, emailstrategyimplementation.ErrFlowNotCovered) {
    log.Warn("Flow not covered - expected behavior")
    return response, nil  // Graceful degradation
}

// Then handle real errors
if err != nil {
    log.Error("Email sending failed", "error", err)
    return nil, err
}

return response, nil
```

## Benefits

### Code Reduction
- **Before**: ~150 lines boilerplate per strategy
- **After**: ~60 lines actual business logic
- **Savings**: 60% reduction in repetitive code

### Maintainability
- Clear separation: One file per strategy implementation
- Easy to add new strategies: Follow simple 3-step pattern
- No duplicate error handling code

### Type Safety
- Compiler-enforced interface implementation
- Go's embedding ensures all methods available
- No runtime surprises

### Consistency
- All unsupported flows return same error format
- Centralized email + logging logic
- Predictable behavior across strategies

### Testability
- Easy to mock repository
- Clear dependencies via injection
- Unit testable in isolation

## Testing Example

```go
func TestMyStrategy_OrderCreated(t *testing.T) {
    // Setup
    mockRepo := &repository.Repository{
        Bboneemail: mockBBOneEmail,
        Db:         mockDatabase,
    }
    
    strategy := &MyStrategy{
        Repo:              mockRepo,
        BaseEmailStrategy: BaseEmailStrategy{Repo: mockRepo},
    }
    
    ctx := context.Background()
    
    // Test implemented method
    t.Run("sends email successfully", func(t *testing.T) {
        request := &contract.EmailV2OrderCreatedRequest{
            Email:   "test@example.com",
            OrderId: "ORDER123",
        }
        
        response, err := strategy.OrderCreated(ctx, request)
        
        assert.NoError(t, err)
        assert.Equal(t, http.StatusOK, response.Code)
    })
    
    // Test NOT implemented method
    t.Run("unsupported flow returns ErrFlowNotCovered", func(t *testing.T) {
        request := &contract.EmailV2EReceiptRequest{}
        
        response, err := strategy.EReceipt(ctx, request)
        
        assert.Error(t, err)
        assert.True(t, errors.Is(err, ErrFlowNotCovered))
        assert.Equal(t, http.StatusNotImplemented, response.Code)
        assert.Contains(t, response.Message, "not covered")
    })
}
```

## Production Status
- ✅ **Production-ready and stable**
- ✅ Used in all notification emails
- ✅ Comprehensive test coverage
- ✅ Battle-tested in production

## Future Enhancements
- [ ] Add more vehicle types (future products)
- [ ] Implement remaining email flows per strategy
- [ ] Add template versioning system
- [ ] Performance metrics per strategy

## Related Documentation
- [[01-overview|Service Overview]]
- [[04-api-reference|API Reference]]
- [[pkg/emailstrategy/README.md|Full Package Documentation]]

---
**Last Updated**: 2025-01-07
**Pattern Status**: Production-Ready
**Coverage**: 8 strategies, 8 email types