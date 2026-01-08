# Legacy Migration & Technical Debt

## Overview
Notification Center service sedang dalam proses migrasi dan refactoring untuk menghilangkan legacy code dan technical debt. Dokumen ini merangkum status migrasi, rencana refactoring, dan technical debt yang perlu diselesaikan.

## Migration Status: 🟡 In Progress

### Completed Migrations ✅

#### 1. Rule Engine Factory Pattern (Nov 2025)
**Status**: ✅ **COMPLETED**

**Problem Solved**:
- Timing dependency issues (init before repo available)
- Mutable state in registry
- Unclear lifecycle management

**Implementation**:
- Migrated from instance-based injection to Factory Pattern
- Registry now stores factory functions, not instances
- Repository injected at creation time, not after registration

**Benefits**:
- ✅ Zero timing issues
- ✅ 100% thread-safe
- ✅ Type-safe via compiler
- ✅ 60% less boilerplate code

**Documentation**: [[03-rule-engine#Factory Pattern|Rule Engine Documentation]]

**Build Status**: ✅ All tests passing

---

### Active Migrations 🔄

#### 2. Legacy Notification Methods → Unified API
**Status**: 🔄 **IN PROGRESS** (60% complete)

**Target**: Q2 2025

**Legacy Methods Being Deprecated**:

| Legacy Method | Status | Replacement | Migration Path |
|---------------|--------|-------------|----------------|
| `OTPWALegacy` | 🟡 Active | `SendOpt` with WhatsApp type | Move clients to new API |
| `NotifyBookingResult` | 🟡 Active | Rule Engine events | Migrate to event-driven |
| `NotifyCarLocations` | 🟡 Active | Rule Engine events | Migrate to event-driven |
| `NotifyTripHistoryLegacy` | 🟡 Active | Email V2 Strategy | Use strategy pattern |
| `NotifyFareEstimationLegacy` | 🟡 Active | Email V2 Strategy | Use strategy pattern |
| `NotifyTripSharingLegacy` | 🟡 Active | Email V2 Strategy | Use strategy pattern |
| `NotifErrorLegacy` | 🟡 Active | Standardized error handling | Use new error format |
| `NotifSuccessLegacy` | 🟡 Active | Standardized success response | Use new response format |
| `ResetFirebaseLegacy` | 🟡 Active | `ResetPassword` + FCM | Split into separate concerns |

**Migration Strategy**:
1. **Phase 1 (Current)**: Both APIs running in parallel
2. **Phase 2 (Q1 2025)**: Deprecation warnings added
3. **Phase 3 (Q2 2025)**: Legacy APIs removed

**Client Migration Guide**:

```go
// ❌ OLD - Legacy OTP
req := &contract.OTPWALegacyRequest{
    PhoneNumber: "+628123456789",
    Otp:         "123456",
    Usecase:     "registration",
    Language:    "id",
}

// ✅ NEW - Unified OTP
req := &contract.SendOptRequest{
    InternalId:  "user_123",
    FullName:    "John Doe",
    Destination: "+628123456789",
    Otp:         "123456",
    Type:        contract.OtpType_WHATSAPP,
    Languange:   "id",
    Purpose:     contract.PurposeOtp_REGISTER,
}
```

**Breaking Changes**:
- Request/response format different
- Error codes standardized
- Rate limiting applied consistently

---

#### 3. Email Templates → Email Strategy Pattern
**Status**: 🔄 **IN PROGRESS** (75% complete)

**Target**: Q1 2025

**Current Coverage**:

| Channel | Service | Type | OrderCreated | PaymentReceived | DriverAssigned | EReceipt | Other |
|---------|---------|------|--------------|-----------------|----------------|----------|-------|
| WebReservation | RIDE | Airport | ✅ | ✅ | ✅ | ✅ | ⚠️ |
| WebReservation | RENT | BlueBird | ✅ | ✅ | ❌ | ❌ | ❌ |
| WebReservation | RENT | SilverBird | ✅ | ✅ | ❌ | ❌ | ❌ |
| WebReservation | RENT | GoldenBird | ✅ | ✅ | ❌ | ❌ | ❌ |
| MyBB | RIDE | Airport | ✅ | ✅ | ✅ | ⚠️ | ⚠️ |
| MyBB | RENT | BlueBird | ✅ | ✅ | ❌ | ❌ | ❌ |
| MyBB | RENT | SilverBird | ✅ | ✅ | ❌ | ❌ | ❌ |
| MyBB | RENT | GoldenBird | ✅ | ✅ | ❌ | ❌ | ❌ |

**Legend**:
- ✅ Implemented
- ⚠️ Partial
- ❌ Not yet implemented

**Missing Implementations**:
1. RENT vehicle types: DriverAssigned, EReceipt, Reschedule, Cancellation flows
2. RIDE: DriverAssignedChanged, Reschedule, CancelledBySystem flows
3. Corporate channel email strategies (planned for Q2)

**Migration Path**:
```go
// ❌ OLD - Direct template rendering
SendEmail(from, to, subject, html_content)

// ✅ NEW - Strategy-based routing
SendEmailV2(channel, service_category, email_type, specific_request)
```

---

#### 4. Unicush Push → Firebase Cloud Messaging
**Status**: 🔄 **IN PROGRESS** (40% complete)

**Target**: Q2 2025

**Current State**:
- FCM implemented and working
- Unicush still in use for legacy clients
- Dual-send mode active (both FCM + Unicush)

**Migration Phases**:
1. ✅ **Phase 1**: FCM infrastructure setup (DONE)
2. 🔄 **Phase 2**: Migrate Android app to FCM (IN PROGRESS)
3. ⏳ **Phase 3**: Migrate iOS app to FCM (Q1 2025)
4. ⏳ **Phase 4**: Remove Unicush code (Q2 2025)

**Client Impact**:
- Android: Update to FCM SDK
- iOS: Update to FCM iOS SDK (replaces APNs direct)
- Backend: No changes needed (service handles both)

**Performance Comparison**:

| Metric | Unicush | FCM | Improvement |
|--------|---------|-----|-------------|
| Latency | 500ms avg | 200ms avg | 60% faster |
| Success Rate | 92% | 98% | 6% better |
| Battery Impact | High | Low | Significant |
| Features | Basic | Rich | Topics, groups, data |

---

### Planned Migrations ⏳

#### 5. Redis Legacy Keys → Structured Keys
**Status**: ⏳ **PLANNED** (Q1 2025)

**Target**: Standardize all Redis key formats

**Current Issues**:
- Inconsistent key naming
- No expiration on some keys
- Manual JSON serialization

**New Standard**:
```
{service}:{entity}:{identifier}:{version}

Examples:
notificationcenter:lock:send_email:hash_abc123
notificationcenter:rate_limit:user_123:sms
notificationcenter:config:sms:v2
```

---

#### 6. Error Handling Standardization
**Status**: ⏳ **PLANNED** (Q1 2025)

**Current Issues**:
- Mix of error formats (legacy vs new)
- Inconsistent error codes
- Some errors not logged properly

**Target Format**:
```go
type NotificationError struct {
    Code        string    `json:"code"`
    Message     string    `json:"message"`
    Channel     string    `json:"channel"`
    Timestamp   time.Time `json:"timestamp"`
    Trace       string    `json:"trace,omitempty"`
}
```

**Error Code Standards**:
```
NOTIF_ERR_xxx - User errors (4xx)
NOTIF_SRV_xxx - Server errors (5xx)
NOTIF_EXT_xxx - External service errors
```

---

## Technical Debt

### High Priority 🔴

#### 1. Repository Interface Segregation
**Issue**: Repository interfaces terlalu gemuk (God Object pattern)

**Impact**:
- Hard to mock in tests
- Unclear dependencies
- Tight coupling

**Solution**:
```go
// ❌ Current - Fat interface
type Repository struct {
    Db            DatabaseRepo
    Redis         RedisRepo
    Bboneemail    EmailRepo
    Sms           SmsRepo
    Push          PushRepo
    Whatsapp      WhatsappRepo
    // ... 10+ more
}

// ✅ Target - Interface segregation
type EmailNotifier interface {
    SendEmail(ctx, req) error
    LogEmail(ctx, log) error
}

type SmsNotifier interface {
    SendSms(ctx, req) error
    LogSms(ctx, log) error
}

// Use only what you need
type OrderEmailStrategy struct {
    email EmailNotifier
    db    DatabaseRepo
}
```

**Effort**: 2 weeks
**Target**: Q1 2025

---

#### 2. Test Coverage Improvement
**Current Coverage**: ~65%
**Target Coverage**: 85%

**Areas Needing Tests**:
- Email strategy implementations (30% covered)
- Rule engine rules (50% covered)
- Error handling paths (40% covered)
- Integration tests (20% covered)

**Action Plan**:
1. Add unit tests for all email strategies
2. Add integration tests for rule engine
3. Add E2E tests for critical flows
4. Set up coverage gates in CI/CD

**Effort**: 3 weeks
**Target**: Q2 2025

---

#### 3. Configuration Management
**Issue**: Configuration scattered across multiple locations

**Current State**:
- Environment variables
- `.env` files
- Hardcoded constants
- Database config table

**Target**: Centralized config with validation

```go
type Config struct {
    Database  DatabaseConfig  `validate:"required"`
    Redis     RedisConfig     `validate:"required"`
    Firebase  FirebaseConfig  `validate:"required"`
    Email     EmailConfig     `validate:"required"`
    Sms       SmsConfig       `validate:"required"`
}

func LoadConfig() (*Config, error) {
    // Validate all at startup
    // Fail fast if invalid
}
```

**Effort**: 1 week
**Target**: Q1 2025

---

### Medium Priority 🟡

#### 4. Logging Standardization
**Issue**: Mix of structured and unstructured logs

**Action**:
- Standardize all logs to structured format
- Add request ID to all log lines
- Consistent log levels

**Effort**: 1 week
**Target**: Q2 2025

---

#### 5. Documentation Enhancement
**Issue**: Missing architecture diagrams and detailed docs

**Needed**:
- [ ] C4 Model diagrams (Context, Container, Component, Code)
- [ ] Sequence diagrams for critical flows
- [ ] Data flow diagrams
- [ ] Deployment architecture
- [ ] Runbook for on-call

**Effort**: 2 weeks
**Target**: Q2 2025

---

### Low Priority 🟢

#### 6. Dependency Updates
**Current**: Some dependencies are 1-2 versions behind

**Action**:
- Quarterly dependency review
- Security patch updates
- Major version upgrades (with testing)

**Effort**: Ongoing
**Schedule**: Quarterly

---

#### 7. Performance Optimization
**Current**: 95th percentile latency ~500ms

**Targets**:
- Email send: 200ms → 100ms
- SMS send: 150ms → 75ms
- Push send: 100ms → 50ms

**Optimizations**:
- Connection pooling tuning
- Parallel sends for multiple channels
- Redis pipeline usage
- Template pre-compilation

**Effort**: 2 weeks
**Target**: Q3 2025

---

## Migration Runbook

### Adding New Email Strategy

**Step-by-Step**:

1. **Define Strategy Struct** (`strategy_types.go`)
```go
type NewChannelServiceEmailStrategy struct {
    BaseEmailStrategy
    Repo *repository.Repository
}
```

2. **Implement Methods** (new file `implementations/new_strategy.go`)
```go
func (s *NewChannelServiceEmailStrategy) OrderCreated(...) {
    // Implementation
}
```

3. **Update Router** (`email_strategy.go`)
```go
case "NewChannel":
    return &NewChannelServiceEmailStrategy{
        Repo: esm.repo,
        BaseEmailStrategy: BaseEmailStrategy{Repo: esm.repo},
    }, nil
```

4. **Add Tests**
```go
func TestNewChannelServiceEmailStrategy_OrderCreated(t *testing.T) {
    // Test implementation
}
```

5. **Update Documentation**
- Add to coverage matrix
- Update routing table
- Add example requests

---

### Deprecating Legacy API

**Checklist**:

1. [ ] Identify all clients using legacy API
2. [ ] Create migration guide
3. [ ] Implement new API
4. [ ] Add parallel support (both APIs working)
5. [ ] Add deprecation warnings to legacy API
6. [ ] Notify all clients (email + Slack)
7. [ ] Monitor usage metrics (decrease over time)
8. [ ] Set sunset date (min 3 months notice)
9. [ ] Remove legacy code after sunset date
10. [ ] Clean up related code (tests, docs)

**Communication Template**:
```
Subject: [ACTION REQUIRED] API Deprecation Notice - {API_NAME}

Dear Team,

The {API_NAME} endpoint will be deprecated on {DATE}.

Reason: {REASON}
Replacement: {NEW_API}
Migration Guide: {LINK}
Support: {CONTACT}

Timeline:
- Now: Both APIs available
- {DATE-60d}: Deprecation warnings added
- {DATE-30d}: Final reminder
- {DATE}: Legacy API removed

Please migrate before {DATE} to avoid service disruption.
```

---

## Monitoring Migration Progress

### Metrics to Track

**API Usage**:
```promql
# Legacy API usage (should decrease)
sum(rate(notification_legacy_api_calls_total[5m])) by (endpoint)

# New API adoption (should increase)
sum(rate(notification_v2_api_calls_total[5m])) by (endpoint)
```

**Email Strategy Coverage**:
```promql
# Emails sent via strategy pattern
sum(rate(notification_email_v2_sent_total[5m])) by (channel, service, type)

# Emails sent via legacy method
sum(rate(notification_email_legacy_sent_total[5m]))
```

**Error Rates**:
```promql
# Should remain stable during migration
sum(rate(notification_errors_total[5m])) by (channel)
```

### Dashboard Links
- [Migration Progress](http://grafana/d/migration)
- [API Usage Comparison](http://grafana/d/api-usage)
- [Error Tracking](http://grafana/d/errors)

---

## Risk Assessment

### High Risk Items 🔴
1. **Removing legacy APIs too early** → Client breakage
2. **Rule engine factory bugs** → Notification failures
3. **Email strategy routing errors** → Wrong templates sent

**Mitigation**:
- Extended deprecation periods (3+ months)
- Comprehensive testing before deployment
- Canary deployments
- Feature flags for gradual rollout

### Medium Risk Items 🟡
1. **Performance regression during migration**
2. **Increased complexity during transition**
3. **Team bandwidth for parallel support**

**Mitigation**:
- Performance benchmarks before/after
- Documentation and knowledge sharing
- Dedicated migration support team

---

## Success Criteria

Migration considered successful when:

- [ ] All legacy APIs usage < 5% of total
- [ ] Email strategy coverage ≥ 95%
- [ ] Test coverage ≥ 85%
- [ ] Zero P0/P1 incidents related to migration
- [ ] Documentation complete
- [ ] Team onboarding updated
- [ ] Performance maintained or improved

---

## Related Documentation
- [[01-overview|Service Overview]]
- [[02-email-strategy|Email Strategy Pattern]]
- [[03-rule-engine|Rule Engine Architecture]]
- [[_docs/architecture_assessment.md|Architecture Assessment]]
- [[_docs/RULE_ENGINE_REFACTORING.md|Rule Engine Refactoring]]

---
**Last Updated**: 2025-01-07
**Migration Owner**: MRG Platform Team
**Review Cadence**: Bi-weekly