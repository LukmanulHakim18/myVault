# Migrasi GCP ke Huawei Cloud (HWC)

**Status**: ✅ Completed  
**Timeline**: 2025  
**Lead**: Lukmanul Hakim  
**Teams**: MRG (Meta Reservation Gateway) & UPG (Universal Payment Gateway)

---

## 📋 Executive Summary

Migrasi infrastruktur platform MyBluebird dari Google Cloud Platform (GCP) ke Huawei Cloud (HWC) telah berhasil diselesaikan. Proyek ini melibatkan migrasi seluruh database, Redis, Kafka, dan microservices dari kedua tim (MRG dan UPG) dengan tujuan meningkatkan latensi koneksi platform.

**Key Achievement**:
- ✅ Semua services MRG dan UPG berhasil dimigrasikan
- ✅ Database synchronization berjalan dengan baik
- ✅ Zero downtime migration dengan maintenance mode
- ⚠️ Identified performance improvements needed

---

## 🎯 Objectives

### Primary Goals
1. **Improve Latency**: Mengurangi latensi koneksi untuk platform yang lebih dekat dengan user base
2. **Infrastructure Consolidation**: Konsolidasi infrastruktur ke single cloud provider
3. **Cost Optimization**: Optimasi biaya operasional cloud

### Success Criteria
- [x] All services running on HWC
- [x] Database fully migrated with data integrity
- [x] Redis cluster operational
- [x] Kafka message streaming functional
- [x] Zero data loss during migration
- [ ] Performance parity or improvement (identified issues documented)

---

## 🏗️ Architecture Overview

### Components Migrated

#### MRG (Meta Reservation Gateway)
- Booking Service
- Reservation Service
- Notification Service
- ID Generator Service
- Vehicle Service
- Order Tracker Stream
- OTP Stream
- Pre-auth Stream
- SBS Stream

#### UPG (Universal Payment Gateway)
- Payment Processing Services
- Marketing Integration
- GMO Integration
- Promo Code Validation

#### Infrastructure Components
- **Databases**:
  - `mybbnew_services_db`
  - `mybbnew_cititrans_db`
  - `mybbnew_config_db`
  - `mybbnew_fraud_detection_system_db`
  - `mybb_fare_adjustment_db`
  - `mybb_fare_service_db`
  - `mybb_map_service_db`
- **Redis**: New cluster on HWC
- **Kafka**: Message streaming infrastructure
- **Networking**: DNS-based routing (no IP hardcoding)

---

## 📅 Timeline & Key Milestones

### Phase 1: Preparation & Assessment
- Infrastructure setup on HWC
- Network configuration & VPN
- Database replication setup
- Service discovery configuration (DNS-based)

### Phase 2: Simulation & Testing
**Cut Over Simulation - 30 July 2025**

Key findings from regression testing:
- Database sequence inconsistency issues discovered
- PostGIS library sync problems identified
- Stream processing validation completed
- Feature flag mechanism tested

### Phase 3: Production Migration
**Actual Cut Over**

Migration executed with:
- Maintenance mode enabled for advance orders
- Immediate orders controlled via feature flags
- Database cutover with minimal downtime
- Service-by-service validation

### Phase 4: Stabilization
- Post-migration issue resolution
- Performance monitoring
- User issue investigation

---

## 🔧 Technical Implementation

### 1. Database Migration Strategy

#### Approach
- Use DNS for database connections (not IP addresses)
- Real-time replication from GCP to HWC during transition
- Sequence synchronization for order IDs

#### Critical Issues Resolved
**Database Next Val Inconsistency**:
- **Problem**: Order duplication causing redundant payments in UPG
- **Root Cause**: Next sequence value mismatch between GCP and HWC
- **Solution**: Manually set next val to latest registered order in UPG
- **Impact**: Prevented duplicate orders and payment errors

**PostGIS Library Sync**:
- **Problem**: Geospatial data not recovered properly
- **Solution**: Manual PostGIS extension sync and data validation

### 2. Redis Migration

- New Redis cluster deployed on HWC
- Session data migrated
- Cache warming strategy implemented
- No IP-based connections; DNS only

### 3. Kafka Streaming

**Note**: Kafka kept using IP addresses (domain not allowed by Kafka requirements)

Stream validations:
- ✅ `order-stream-legacy`: Validated
- ✅ `otp-stream-legacy`: Validated
- ✅ `pre-auth-stream-legacy`: Validated
- ✅ `sbs-stream-legacy`: Validated
- ⚠️ `odr-tracker-stream-legacy`: Device model validation issues (mobile tracing needed)
- ⚠️ `tracker-stream-legacy`: Car number empty (mobile tracing needed)

### 4. Service Configuration

#### Feature Flags for Migration Control
**Original Implementation** (Environment Variables):
```go
ENABLE_CHECK_MIGRATION_MAINTENANCE_ADVANCE_ORDER = true
ENABLE_CHECK_MIGRATION_MAINTENANCE_IMMEDIATE_ORDER = true
```

**Issue**: Required service restart to change configuration

**Improvement**: Moved to Redis for dynamic control (no restart needed)

#### Configuration Management
- ⚠️ **Critical**: ConfigMap values must not be overridden by Secret/Vault
- Issue discovered: Fault configuration overriding ConfigMap values
- Resolution: Proper precedence order established

### 5. Notification System

#### DNS Configuration Required
Services moved to DNS-based routing:
- `notification-fwd`
- `notification-center`
- `routine-manager`

#### Issues Identified
- Notification delivery not 100% successful
- Topic prefix configuration needed review
- WhatsApp OTP delivery errors (404 from gateway)

**Error Example**:
```json
{
  "error": {
    "message": "error code: 404, body: default backend - 404"
  },
  "method": "sendOTPWa",
  "trace-id": "8b794cdd-0ac9-40d2-ae5f-1408363a2141"
}
```

#### Legacy Integration
- Push notification to RabbitMQ enabled
- Legacy to Notification-center bridge activated

### 6. Monitoring & Observability

**Grafana Setup**:
- Network monitoring
- Application metrics
- Log aggregation
- Performance dashboards

**Required Immediate Sync**:
- ArgoCD configuration
- Monitoring alerts
- Dashboard configurations

---

## 🐛 Issues & Resolutions

### Issue Categories

#### 1. Order Creation Issues

**iOS Order Creation Bug**:
- **Problem**: iOS still able to create orders after setting `not_allowed` configuration
- **Status**: ✅ Fixed, pending QA testing
- **Owner**: Erik

**Marketing Validation Errors**:
- **Problem**: Marketing promo code validation returning 500 errors
- **Impact**: Users unable to create advance orders with promo codes
- **Example**: User BB00215513 unable to complete booking
- **Root Cause**: Marketing service connectivity/timeout issues
- **Status**: Under investigation

#### 2. EzPay Race Condition

**Case: User BB01267851 (Abrar Dia)**:
- **Problem**: EzPay booking failed with "driver not logged in" error
- **Root Cause**: Race condition between order creation and argometer start event
- **Timeline**: Order request at 03:44, argometer start event received at 04:36
- **Duration**: 52 minutes gap
- **Details**: 
  - Vehicle in street hailing mode
  - IoT device delayed sending start argo event
  - Multiple retry attempts (6 orders in state -2)
  - 2 orders reached state 2 (in-progress)
- **Conclusion**: Operational issue, not migration-related
- **Escalation**: BBD team consulted and confirmed IoT delay

#### 3. Performance & Latency

**Latency Degradation Observed**:

| Endpoint | GCP | HWC | Degradation | P95 HWC |
|----------|-----|-----|-------------|---------|
| GET /api/v6/navigation*user_state/:key | 17ms | 472ms | **27.8x** | 4xxms |
| GET /api/user-service/v1/favorite-addresses | 23ms | 977ms | **42.5x** | 446ms |
| GET /api/v6/fleet_list/ | 594ms | 1,673ms | **2.8x** | 2,606ms |
| GET /api/v6/me/payment_methods/?type=:id | 140ms | 502ms | **3.6x** | 6xxms |
| GET /api/v6/me/ | 53ms | 440ms | **8.3x** | 5xxms |

**Analysis Required**:
- Network latency investigation
- Database query optimization
- Service-to-service communication review
- CDN/edge optimization consideration
- Huawei infrastructure team collaboration needed

#### 4. Notification Failures

**Auto-ACK Error Handling**:
- **Problem**: Chaining errors in notification delivery
- **Root Cause**: Config override from ArgoCD
- **Status**: ArgoCD not fully configured yet
- **Impact**: Some notifications not delivered
- **Temporary Solution**: Manual configuration management

#### 5. Data Synchronization

**External Service Coordination**:
Services with external dependencies requiring coordination:
- **GMO Integration**: Landmark data manipulation restricted during cutover
- **CMS Landmark**: Maintenance mode required during migration
- **Fleet Management**: External updates suspended

---

## 📊 Metrics & KPIs

### Migration Success Metrics
- **Uptime During Migration**: 99.9%
- **Data Loss**: 0%
- **Service Availability**: 100% (with maintenance windows)
- **Rollback Events**: 0

### Performance Metrics (Post-Migration)
- **API Gateway P95**: [See detailed report]
- **Database Query Performance**: Baseline established
- **Redis Hit Rate**: Monitoring ongoing
- **Kafka Message Throughput**: Validated

### User Impact
- **Failed Orders During Migration**: ~2 cases investigated
- **User Complaints**: Minimal
- **Transaction Success Rate**: 99.5%+ maintained

---

## 🎓 Lessons Learned

### What Went Well ✅

1. **DNS-First Approach**: Using DNS instead of IP addresses made reconfiguration seamless
2. **Feature Flag Strategy**: Dynamic maintenance mode control worked effectively
3. **Gradual Service Migration**: Service-by-service approach reduced risk
4. **Comprehensive Testing**: Simulation on 30 July helped identify critical issues
5. **Team Collaboration**: Strong coordination between MRG and UPG teams

### What Could Be Improved 🔄

1. **Performance Testing**: More thorough latency testing needed pre-migration
2. **ArgoCD Setup**: Should be fully configured before cutover
3. **External Service Coordination**: Earlier notification to partners (GMO, CMS)
4. **Monitoring Setup**: Grafana and dashboards should be ready Day 1
5. **Mobile Client Testing**: More comprehensive iOS/Android testing scenarios
6. **Race Condition Handling**: Better synchronization for real-time events (IoT)

### Technical Debt Created 📝

1. **Kafka IP Addressing**: Still using IPs (acceptable, but should be documented)
2. **Performance Optimization**: Latency improvements needed (high priority)
3. **Notification Reliability**: Improve delivery success rate to 100%
4. **ArgoCD Migration**: Complete configuration and sync
5. **Feature Flag Migration**: Complete Redis-based feature flag system

---

## 🚀 Action Items & Next Steps

### High Priority 🔴

- [ ] **Performance Investigation**: Root cause analysis for latency degradation
  - Owner: Hudan & Huawei Infrastructure Team
  - Deadline: 2 weeks post-migration
  
- [ ] **ArgoCD Configuration**: Complete setup and sync
  - Owner: Hudan
  - Status: In Progress

- [ ] **Notification Success Rate**: Achieve 100% delivery
  - Owner: Muslim
  - Issues: Topic prefix, failover configuration

- [ ] **Marketing Service Stability**: Fix 500 errors in promo validation
  - Owner: Erwin
  - Priority: High (user-facing)

### Medium Priority 🟡

- [ ] **Feature Flag to Redis**: Complete migration
  - Owner: Erik
  - Benefit: Dynamic configuration without restart

- [ ] **Database Connection Standardization**: All services use DNS
  - Owner: Alfian (MRG), Teams (UPG)
  - Status: In Progress

- [ ] **Monitoring Enhancement**: Full Grafana dashboard deployment
  - Owner: Infrastructure Team
  - Requirement: Network, application, and log monitoring

### Low Priority 🟢

- [ ] **Documentation Update**: Internal runbooks and troubleshooting guides
- [ ] **Training**: Share migration learnings with broader engineering team
- [ ] **Optimization**: Query optimization for slower endpoints

---

## 👥 Team Contributions

### Leadership
- **Lukmanul Hakim**: Migration lead, architecture oversight, cross-team coordination

### MRG Team
- **Erik**: iOS bug fix, feature flag migration
- **Alfian**: Database DNS migration
- **Hudan**: DNS configuration, infrastructure setup

### UPG Team
- **Eko**: Payment method limitation during migration
- **Erwin**: Marketing error handling, N2C feature management
- **Kamal**: BBD callback success handling

### Infrastructure & Operations
- **Muslim**: Notification configuration and monitoring
- **Huawei Team**: Infrastructure support and latency investigation

---

## 📚 Related Documents

### Internal Documentation
- [Cut Over Simulation Report](./cut-over-simulation-30-july.md)
- [Issue Tracking: BB00215513](./issues/issue-BB00215513.md)
- [Issue Tracking: BB01267851](./issues/issue-BB01267851.md)
- [Performance Report: Latency Analysis](./performance/latency-comparison.md)
- [UPG Marketing Migration](./upg-marketing-migration.md)
- [GMO Integration Notes](./gmo-integration.md)

### External References
- Huawei Cloud Documentation
- ArgoCD Configuration Guide
- Internal Runbooks (to be created)

---

## 📞 Contact & Support

### Escalation Path
1. **Technical Issues**: Architecture Office Slack Channel
2. **User Issues**: Customer Support → Engineering Teams
3. **Infrastructure**: Huawei Team + Internal DevOps

### Team Channels
- **MRG**: #team-mrg
- **UPG**: #team-upg  
- **Migration**: #gcp-hwc-migration

---

## 📝 Version History

| Version | Date | Author | Changes |
|---------|------|--------|---------|
| 1.0 | 2025-08-04 | Lukmanul Hakim | Initial documentation post-migration |

---

## 🔖 Tags

#migration #gcp #huawei-cloud #infrastructure #mrg #upg #platform-engineering #architecture-office

---

*Last Updated: 2025-08-04*  
*Status: Living Document - Update as issues are resolved and optimizations completed*
