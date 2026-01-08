# Cut Over Simulation - 30 Juli 2025

**Type**: Regression Testing  
**Date**: 30 Juli 2025  
**Purpose**: Validate migration readiness before production cutover  
**Status**: ✅ Completed with findings

---

## 🎯 Simulation Objectives

1. Validate database migration and synchronization
2. Test feature flag mechanisms
3. Verify stream processing integrity
4. Identify potential issues before production cutover
5. Validate rollback procedures

---

## 🔍 Key Findings

### 1. Database Issues

#### Database Sequence Inconsistency ⚠️
**Problem**: 
- Next value (next val) inconsistent between GCP and HWC databases
- Causing order ID duplication

**Impact**:
- Duplicate orders created
- Redundant payment attempts in UPG
- Transaction errors and confusion

**Root Cause**:
- Database sequence not properly synchronized during replication
- Manual intervention required to align sequences

**Solution Implemented**:
```sql
-- Set next val to latest order in UPG
SELECT setval('order_id_seq', (SELECT MAX(id) FROM orders) + 1);
```

**Validation**:
- ✅ No more duplicate orders after fix
- ✅ Payment processing normalized
- ✅ Order ID generation consistent

#### PostGIS Library Sync Issue ⚠️
**Problem**:
- PostGIS geospatial extension not properly synchronized
- Geolocation data not recovered correctly

**Impact**:
- Map features potentially affected
- Location-based queries unreliable

**Solution**:
- Manual PostGIS extension installation and configuration
- Data validation for geospatial columns
- Re-import spatial reference systems

**Validation**:
- ✅ PostGIS functions working
- ✅ Spatial queries returning correct results
- ✅ Map integration validated

---

## 🔄 Stream Processing Validation

### Successfully Validated Streams ✅

#### 1. Order Stream Legacy
**Status**: ✅ DONE  
**Validation**:
- Message format correct
- Throughput maintained
- Consumer processing confirmed
- No data loss detected

#### 2. OTP Stream Legacy  
**Status**: ✅ DONE  
**Validation**:
- OTP generation working
- Delivery tracking functional
- Retry mechanism validated
- No message drops

#### 3. Pre-Auth Stream Legacy
**Status**: ✅ DONE  
**Validation**:
- Authorization flow complete
- Token generation working
- Payment pre-authorization confirmed
- No security issues

#### 4. SBS (Silver Bird Service) Stream Legacy
**Status**: ✅ DONE  
**Validation**:
- Premium service booking flow
- Event processing correct
- No message corruption

---

### Streams Requiring Further Investigation ⚠️

#### 1. Order Tracker Stream Legacy
**Status**: ⚠️ Needs Mobile Tracing  
**Issue**: `device-model` field not valid

**Details**:
```json
{
  "order_id": "...",
  "device_model": null,  // ❌ Missing or invalid
  "tracking_data": {...}
}
```

**Required Actions**:
- Trace mobile client implementation
- Validate device model detection logic
- Verify Android and iOS payload formats
- Test with different device types

**Priority**: Medium  
**Owner**: Mobile Team + MRG

#### 2. Tracker Stream Legacy
**Status**: ⚠️ Needs Mobile Tracing  
**Issue**: `car-number` field empty

**Details**:
```json
{
  "driver_id": "...",
  "car_number": "",  // ❌ Empty string
  "location": {...}
}
```

**Potential Causes**:
- Mobile app not sending car number
- Field mapping issue in API
- Legacy data migration incomplete
- Driver assignment timing issue

**Required Actions**:
- Review mobile app tracking logic
- Validate driver-vehicle assignment flow
- Check database migration for car_number field
- Test with active drivers

**Priority**: Medium  
**Owner**: Mobile Team + Fleet Management

---

## 🚩 Feature Flag Testing

### Maintenance Mode Flags

#### Configuration Tested
```bash
ENABLE_CHECK_MIGRATION_MAINTENANCE_ADVANCE_ORDER=true
ENABLE_CHECK_MIGRATION_MAINTENANCE_IMMEDIATE_ORDER=true
```

### Test Scenarios

#### Scenario 1: Advance Orders
**Config**: `ENABLE_CHECK_MIGRATION_MAINTENANCE_ADVANCE_ORDER=true`  
**Expected**: Block new advance bookings  
**Result**: ✅ Working as intended  
**Note**: Users shown maintenance message

#### Scenario 2: Immediate Orders
**Config**: `ENABLE_CHECK_MIGRATION_MAINTENANCE_IMMEDIATE_ORDER=true`  
**Expected**: Block instant ride requests  
**Result**: ✅ Working as intended  
**Note**: Alternative transport options suggested to users

---

### Issues Discovered 🐛

#### Service Restart Requirement
**Problem**:
- Feature flag changes require full service restart
- No hot-reload capability
- Potential downtime during flag changes

**Impact**:
- Cannot dynamically enable/disable maintenance mode
- Risk during emergency flag updates
- Operational complexity

**Proposed Solution**:
- Move feature flags to Redis
- Implement hot-reload mechanism
- No restart needed for flag changes

**Status**: Improvement backlog (see main README action items)

---

## ⏱️ Performance Observations

### Response Time Baseline
- Established baseline metrics for comparison
- Some endpoints showing increased latency
- Full performance analysis documented separately

### Resource Utilization
- CPU usage within normal range
- Memory consumption acceptable
- Database connection pool healthy
- Redis cache hit rate good

---

## 🔄 Rollback Testing

### Rollback Scenarios Tested

#### Scenario 1: Database Rollback
**Trigger**: Critical data integrity issue  
**Procedure**: 
1. Switch DNS back to GCP
2. Verify data consistency
3. Validate service connectivity

**Result**: ✅ Rollback successful in < 5 minutes

#### Scenario 2: Service Rollback
**Trigger**: Service instability on HWC  
**Procedure**:
1. Update service discovery
2. Redirect traffic to GCP
3. Monitor health checks

**Result**: ✅ Clean rollback, no data loss

#### Scenario 3: Partial Rollback
**Trigger**: Specific service issue  
**Procedure**:
1. Roll back single service
2. Keep others on HWC
3. Maintain data consistency

**Result**: ✅ Selective rollback working

---

## 📋 Action Items from Simulation

### Critical (Pre-Cutover) 🔴
- [x] Fix database sequence inconsistency
- [x] Resolve PostGIS sync issues
- [ ] Investigate device-model validation (defer to post-migration)
- [ ] Investigate car-number empty issue (defer to post-migration)

### Important (Pre-Cutover) 🟡
- [x] Document rollback procedures
- [x] Prepare monitoring dashboards
- [x] Set up alerting rules
- [ ] Plan Redis-based feature flags (post-migration improvement)

### Nice to Have 🟢
- [ ] Automate rollback scripts
- [ ] Improve simulation environment
- [ ] Create detailed runbooks

---

## 📊 Test Coverage Summary

| Component | Test Status | Issues Found | Issues Resolved |
|-----------|-------------|--------------|-----------------|
| Database Migration | ✅ Tested | 2 | 2 |
| Redis | ✅ Tested | 0 | 0 |
| Kafka Streams (Core) | ✅ Tested | 0 | 0 |
| Kafka Streams (Tracking) | ⚠️ Partial | 2 | 0 (deferred) |
| Feature Flags | ✅ Tested | 1 | 0 (improvement) |
| Rollback | ✅ Tested | 0 | 0 |
| Monitoring | ✅ Tested | 0 | 0 |

---

## 🎓 Lessons from Simulation

### What Worked Well ✅
1. **Database sync detection**: Caught critical sequence issue before production
2. **Stream validation**: Systematic approach found edge cases
3. **Rollback testing**: Confidence in emergency procedures
4. **Team coordination**: Clear communication during simulation

### Areas for Improvement 🔄
1. **Earlier testing**: Simulation should be done 2 weeks before cutover
2. **Mobile integration**: Need better mobile team involvement in testing
3. **Automated checks**: More automation for database validation
4. **Load testing**: Add load testing to simulation

---

## ✅ Go/No-Go Decision

### Pre-Cutover Checklist
- [x] All critical issues resolved
- [x] Rollback procedures documented and tested
- [x] Monitoring ready
- [x] Team trained on procedures
- [x] Communication plan ready
- [ ] All stream issues resolved (deferred, acceptable)

### Decision: ✅ GO for Production Cutover

**Rationale**:
- Critical database issues resolved
- Core functionality validated
- Rollback procedures proven
- Minor issues (device-model, car-number) acceptable for post-migration resolution
- Risk level: Low to Medium

**Approved By**: Lukmanul Hakim (Architecture Lead)  
**Date**: 2025-07-30

---

## 📝 Notes & Observations

### Question A1 (From Original Document)
**Question**: 
- Berapa lama eksekusi satu messagenya?
- Apakah tidak apa-apa dan membuat bingung user terkait notifikasi?

**Analysis**:
- Message execution time: < 100ms average
- User confusion risk: Low (maintenance messages clear)
- Notification timing: Acceptable for migration window

**Decision**: Proceed with current notification strategy

---

## 📎 Related Documents
- [Main Migration README](./README.md)
- [Performance Analysis](./performance/latency-comparison.md)
- [Issue Tracking](./issues/)

---

*Document Status: Final*  
*Last Updated: 2025-07-30*  
*Next Review: Post-production cutover*
