# GCP to Huawei Cloud Migration - Incidents

**Migration Period**: Juli - Agustus 2025  
**Total Incidents**: 2 (Post-Migration)

---

## 📊 Incident Summary

| ID | Date | Severity | Service | Status | Description |
|----|------|----------|---------|--------|-------------|
| [BB00215513](incident-BB00215513-marketing-error.md) | 2025-08-03 | HIGH | Marketing | ✅ Resolved | Promo code validation 500 error |
| [BB01267851](incident-BB01267851-ezpay-race-condition.md) | 2025-08-04 | MEDIUM | EzPay/IOT | ✅ Escalated | Race condition IOT event delay |

---

## 🎯 Incident Categories

### Service Errors
- **Marketing Service 500 Error** (BB00215513)
  - Impact: Promo code validation failures
  - Root Cause: Server error post-migration
  - Solution: Error handling improvement + wrapper

### Integration Issues
- **EzPay IOT Race Condition** (BB01267851)
  - Impact: Failed EzPay orders
  - Root Cause: IOT event delay (52 minutes)
  - Solution: Grace period + BBD escalation

---

## 📈 Incident Metrics

### By Severity
- HIGH: 1 incident (50%)
- MEDIUM: 1 incident (50%)

### By Team
- UPG: 1 incident (Marketing Service)
- MRG: 1 incident (EzPay/Booking Service)
- External (BBD): 1 incident contributing factor

### Resolution Time
- Average: ~4 hours
- Fastest: 2 hours (BB00215513)
- Slowest: Escalated to external team (BB01267851)

---

## 🔍 Common Patterns

### Post-Migration Issues
1. **Service Integration Problems**
   - External service timeouts
   - Network latency impact
   - Configuration mismatches

2. **Race Conditions**
   - IOT event timing issues
   - Async process synchronization
   - External dependency delays

3. **Error Handling Gaps**
   - Missing error wrappers
   - Poor user-facing messages
   - No graceful degradation

---

## ✅ Lessons Learned

### What We Improved
1. ✅ Better error handling patterns
2. ✅ Improved monitoring and alerting
3. ✅ Faster incident response process
4. ✅ Better documentation practices

### Areas for Further Improvement
1. [ ] Pre-migration integration testing
2. [ ] More comprehensive load testing
3. [ ] Better external dependency management
4. [ ] Automated incident detection

---

## 📚 Related Documents

- [Main Migration Document](../../initiatives/gcp-to-hwc-migration.md)
- [Post-Migration Review](../postmortems/) (TODO)
- MRG Services: `02-Work/Teams/MRG/02-services/`
- UPG Services: `02-Work/Teams/UPG/02-services/`

---

**Last Updated**: 4 Agustus 2025  
**Maintained by**: Platform Engineering Team

**Tags**: #incidents #migration #gcp #huawei #post-mortem
