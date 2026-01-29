---
title: "Active Tasks Tracker - Lukmanul Hakim"
type: "task-tracking"
date: 2026-01-27
status: "active"
updated: 2026-01-27
---

# 📊 Task Tracker - Ongoing Work

**Last Updated**: 2026-01-27  
**Owner**: Lukmanul Hakim  

---

## 🔴 **HIGH PRIORITY - CRITICAL PATH**

### 1. **ECV for GB Rent Revamp - Push HO Architecture**
**Target Release**: v6.24 (W4 Feb - Regression Start)  
**Status**: 🔴 NOT STARTED  
**Owner**: Team Discussion (MRG + UPG + CP)

#### Background
- GB charging di depan (beda dari BB/SB yang belakang)
- Rent Revamp v6.24 hanya support ECV + cc_cp untuk hourly
- Perlu definisikan push HO flow untuk payment, refund, tipping

#### Key Questions to Resolve
1. **Flow Payment ECV**: Apakah mengikuti Cititrans pattern?
2. **Budget Calculation**: Validasi untuk cross-month orders (Jan order untuk Feb)
3. **Pelaporan HO**: MRG yang handle (decision sudah confirmed)

#### Decision Needed
- [ ] **Architecture Decision**: Single vs Multiple posting untuk HO
  - Option A: Single posting (simpler)
  - Option B: Multiple posting dengan journal routing (overtime, extra, tipping, cancel)
  
#### Subtasks
- [ ] Define journal routing rules (jika pilih Option B)
- [ ] Design topic schema untuk general rent topic
- [ ] Error handling & retry mechanism untuk push HO failure
- [ ] Push CP failure handling & recovery
- [ ] Testing scenarios:
  - [ ] Tipping scenarios
  - [ ] Refund/void scenarios
  - [ ] Cross-month order scenarios
  - [ ] Budget calculation validation
- [ ] Monitoring & alerting untuk multi-charging flow

#### Implementation Owner
- **MRG**: Handle tipping case + push HO dengan parameter `is_push_ho`
- **UPG**: Optional push HO (based on `is_push_ho = true`)
- **CP**: Direct payment push, refund/void handling, tipping update

#### Timeline
- **Week 1 (Jan 27-31)**: Architecture decision + journal routing design
- **Week 2-3 (Feb 3-14)**: CP internal assessment + MRG development
- **W4 Feb (Feb 17-21)**: Start regression & testing

#### Dependencies
```
Architecture Decision 
  ├── Journal Routing Design (if Option B)
  ├── Topic Schema Design
  ├── MRG Implementation (Tipping)
  ├── UPG Implementation (Optional)
  └── CP Assessment
  
All must complete before W4 Feb regression
```

---

### 2. **Traffic Spike Analysis - Order Gantung Incident**
**Incident Date**: 2025-01-12 (77k concurrent users)  
**Status**: 🟡 INVESTIGATING  
**Owner**: Lukmanul Hakim  
**Severity**: HIGH

#### Problem Summary
- Order stuck di ACTIVE status (tidak reach COMPLETE)
- Payment tidak tercatat di HO meski sudah charged
- Data inconsistency antara UPG dan HO

#### Investigation Phases

##### Phase 1: Data Collection (URGENT)
- [ ] Collect metrics dari Huawei Cloud (12 Jan full day)
- [ ] Export logs: MRG, UPG, Order Orchestrator
- [ ] Query database untuk list order gantung
- [ ] Collect payment transaction records
- [ ] Check HO records untuk missing payments
- [ ] Identify affected order count & financial impact

##### Phase 2: Analysis (HIGH)
- [ ] Analyze logs untuk error patterns
- [ ] Correlate order data across services
- [ ] Map timeline: spike → errors → degradation
- [ ] Identify root cause dari hypotheses:
  1. Database connection pool exhaustion
  2. Payment callback failure
  3. Circuit breaker activation
  4. Timeout configuration issue
  5. Race condition di payment flow

##### Phase 3: Root Cause Identification
- [ ] Validate hypothesis dengan data
- [ ] Document primary root cause
- [ ] Identify contributing factors

##### Phase 4: Remediation
- [ ] Short-term fix untuk recover order gantung
- [ ] Payment reconciliation process
- [ ] Customer communication plan

##### Phase 5: Prevention & Load Testing
- [ ] Load testing dengan 100k+ users
- [ ] Capacity planning update
- [ ] Auto-scaling configuration review
- [ ] Circuit breaker threshold tuning
- [ ] Database connection pool optimization
- [ ] HTTP timeout configuration review

#### Go Microservices Investigation Focus
```go
// Check metrics selama incident
- Goroutine count (leak detection)
- GC pause times
- Memory pressure
- HTTP client timeout configuration
```

#### Database Investigation Focus
```sql
-- Connection pool status
SELECT count(*) FROM pg_stat_activity;

-- Lock analysis
SELECT * FROM pg_locks WHERE NOT granted;

-- Slow query log
SELECT * FROM pg_stat_statements ORDER BY total_time DESC LIMIT 20;

-- Payment reconciliation
SELECT COUNT(*), payment_status 
FROM payments 
WHERE DATE(created_at) = '2025-01-12'
GROUP BY payment_status;
```

#### Success Criteria
- [x] Root cause identified
- [ ] All order gantung recovered
- [ ] Payment reconciliation completed
- [ ] Preventive measures implemented
- [ ] Load test successful (100k+ users)

---

### 3. **VP Initiative - Development Guidelines**
**Status**: 🟡 DRAFT  
**Priority**: MEDIUM-HIGH  
**Owner**: Lukmanul (with VP)  
**Type**: Organization-wide Guidelines

#### Scope
1. **Development Guideline untuk Developer**
   - Standard untuk tim development
   - Best practices & coding standards
   - Code review process
   - Testing requirements

2. **Database Best Practices**
   - Database access patterns
   - Query optimization
   - Slow query detection
   - Performance monitoring
   - Connection pooling
   - Transaction handling

#### Deliverables
- [ ] Development guideline document
- [ ] Database access best practices guide
- [ ] Slow query detection & debugging guide
- [ ] Code examples & templates
- [ ] Monitoring & alerting setup guide

#### Timeline
- Pending VP untuk confirm detail scope
- Estimate 2-3 weeks untuk completion

---

## 🟡 **MEDIUM PRIORITY - ONGOING**

### 4. **MCP (Model Context Protocol) Server Development**
**Status**: 🟡 IN PROGRESS  
**Owner**: Lukmanul  
**Goal**: Claude Desktop integration for development workflows

#### Current Work
- mcpmrg project generation dengan Enera Plus
- Repository management: `eneraplus add-repository --repository-name=RepoName`
- Proto update: `eneraplus update-project`

#### Next Steps
- [ ] Complete MCP server implementation
- [ ] Test Claude Desktop integration
- [ ] Document usage & setup guide

---

### 5. **Technical Documentation Updates**
**Status**: 🟡 ONGOING  
**Owner**: Lukmanul

#### Pending Documentation
- [ ] Two-Level ID Generation System - Complete documentation
- [ ] SPOF Analysis for MRG - Mitigation strategies detail
- [ ] GB Rent Revamp - Technical design docs
- [ ] ECV Payment Flow - Detailed diagrams & docs

---

## ✅ **COMPLETED RECENTLY**

### ✓ Reserved Budget ECV Mechanism
**Completed**: 2026-01-27  
**Approach**: Pre-order flow charge implementation

**Completed Tasks**:
- [x] Design reserved budget mechanism (moved to pre-order)
- [x] Implement early balance deduction (pre-order phase)
- [x] Prevent multiple advance orders (insufficient balance check)
- [x] Update validation logic (policy + balance + charge at pre-order)

**Impact**: Prevent order gantung, ensure balance consistency, reduce ID waste

### ✓ GCP to Huawei Cloud Migration
**Completed**: Aug 2025  
**Impact**: Migrated 17+ services, zero downtime

### ✓ Two-Level ID Generation System RFC
**Completed**: Available  
**Status**: Documented, ready for reference

### ✓ MRG SPOF Analysis
**Completed**: Available  
**Key Finding**: Order Orchestrator is SPOF, detailed mitigation provided

### ✓ Traffic Spike Analysis Initial Assessment
**Status**: Investigation checklist prepared  
**Next**: Execute investigation phases

---

## 📋 **SUMMARY BY DEADLINE**

### This Week (Jan 27 - Feb 2)
- [ ] ECV GB Rent Revamp architecture decision
- [ ] Start traffic spike investigation phase 1
- [ ] VP guidelines scope finalization

### Next 2 Weeks (Feb 3 - Feb 14)
- [ ] Complete traffic spike root cause analysis
- [ ] ECV MRG tipping implementation
- [ ] CP internal assessment (ECV)
- [ ] Database best practices documentation draft

### Critical Path (W4 Feb - Feb 17-21)
- [ ] ECV regression start (all implementation must be done)
- [ ] Load testing preparation (traffic spike prevention)

---

## 📊 **TASK METRICS**

| Category | Count | Status |
|----------|-------|--------|
| High Priority | 3 | 1 Investigating, 2 Not Started |
| Medium Priority | 2 | Both In Progress |
| Completed | 5 | All Done |
| **Total Tracking** | **10** | **Mixed** |

---

## 🔗 **Related Documents**

- [[2025-01-27 - ECV for GB Rent Revamp Discussion|ECV Meeting Notes]]
- [[2026-01-20 Meeting VP Architect - Development Guidelines|VP Meeting Notes]]
- [[2025-01-12-Traffic-Spike-Analysis|Traffic Spike Details]]
- [[00-Inbox/Reserved Budget eCV|Reserved Budget Status]]

---

**Last Review**: 2026-01-27  
**Next Review**: 2026-02-03  
**Owner**: Lukmanul Hakim