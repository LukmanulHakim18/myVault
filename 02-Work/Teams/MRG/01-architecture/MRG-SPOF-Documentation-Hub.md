---
tags:
  - mrg
  - architecture
  - spof
  - index
  - documentation-hub
type: index
created: '2026-01-08'
updated: '2026-01-08'
---
# MRG SPOF Documentation Hub

**Purpose**: Central index untuk semua dokumentasi terkait Single Point of Failure assessment dan mitigation  
**Last Updated**: 2026-01-08

---

## 🎯 Quick Navigation

### Core Assessment
| Document | Description | Status |
|----------|-------------|--------|
| [[MRG SPOF Assessment & Mitigation Strategy]] | Main SPOF assessment document (v3.0) | ✅ Active |

### Implementation Guides
| Document | Description | Priority |
|----------|-------------|----------|
| [[Circuit-Breaker-Implementation-Guide]] | Circuit breaker pattern untuk Go services | 🔴 P0 - Sprint 1 |
| [[Patroni-HA-Setup-Guide]] | PostgreSQL HA dengan Patroni | 🔴 P0 - Sprint 1-2 |

### Runbooks (Incident Response)
| Document | Severity | Blast Radius |
|----------|----------|--------------|
| [[RUNBOOK-Database-Failover]] | P0 | 100% |
| [[RUNBOOK-Order-Orchestrator-Down]] | P0 | 90% |
| [[RUNBOOK-TPG-BBD-Down]] | P0 | 60% |

---

## 📊 SPOF Summary Dashboard

### Current Risk Status

```
┌────────────────────────────────────────────────────────────────┐
│                    SPOF RISK OVERVIEW                          │
├────────────────────────────────────────────────────────────────┤
│                                                                 │
│   INFRASTRUCTURE                                                │
│   ══════════════                                                │
│   PostgreSQL HA     [██████████] 100% risk  ⚠️ Needs Patroni   │
│   Redis Cluster     [███░░░░░░░]  30% risk  ✅ Sentinel OK     │
│   RabbitMQ HA       [████░░░░░░]  40% risk  ⚠️ Needs Cluster   │
│                                                                 │
│   SERVICES                                                      │
│   ════════                                                      │
│   Order Orchestrator [██████░░░░]  60% risk  ⚠️ Needs CB       │
│   TPG / BBD          [████████░░]  80% risk  🔴 HIGH RISK      │
│   Payment Processor  [████░░░░░░]  40% risk  ⚠️ Partial CB     │
│   User Service       [██░░░░░░░░]  20% risk  ✅ Has fallback   │
│                                                                 │
│   EXTERNAL DEPS                                                 │
│   ═════════════                                                 │
│   BBD System         [████████░░]  80% risk  🔴 Single dep     │
│   Payment Gateways   [██░░░░░░░░]  20% risk  ✅ Multi-gateway  │
│   Map Providers      [███░░░░░░░]  30% risk  ✅ Has fallback   │
│                                                                 │
└────────────────────────────────────────────────────────────────┘
```

### Key Metrics Targets

| Metric | Current | Target | Timeline |
|--------|---------|--------|----------|
| Database RTO | ~15-30 min | < 5 min | Q1 2026 |
| Database RPO | < 1 min | 0 | Q1 2026 |
| Circuit Breaker Coverage | 0% | 100% critical | Q1 2026 |
| SPOF Services with Fallback | 40% | 80% | Q2 2026 |

---

## 📅 Implementation Roadmap

### Sprint 1 (Current)
- [ ] 🔴 TPG: Implement circuit breaker for BBD
- [ ] 🔴 TPG: Add manual dispatch queue
- [ ] 🟠 Order Orchestrator: Add CB for dependencies
- [ ] 🟠 Setup Patroni POC environment

### Sprint 2
- [ ] 🔴 Patroni: Production deployment
- [ ] 🟠 Order Detail: Active/History split
- [ ] 🟠 Payment Processor: Complete CB implementation

### Sprint 3-4
- [ ] 🟠 Session Manager: Pricing fallback
- [ ] 🟡 Archive strategy for old orders
- [ ] 🟡 Mobile app polling optimization

### Q2 2026
- [ ] Service mesh evaluation (Istio/Linkerd)
- [ ] Chaos engineering framework
- [ ] Multi-region preparation

---

## 📁 Document Structure

```
/02-Work/Teams/MRG/
├── 01-architecture/
│   ├── MRG SPOF Assessment & Mitigation Strategy.md  ← Main doc
│   ├── design-docs/
│   │   ├── Circuit-Breaker-Implementation-Guide.md
│   │   └── Patroni-HA-Setup-Guide.md
│   └── MRG-SPOF-Documentation-Hub.md  ← You are here
│
└── 05-runbooks/
    └── incident-response/
        ├── RUNBOOK-Database-Failover.md
        ├── RUNBOOK-Order-Orchestrator-Down.md
        └── RUNBOOK-TPG-BBD-Down.md
```

---

## 🔗 Related Team Documents

### Service Documentation
- [[02-Work/Teams/MRG/02-services/orderorchestrator/README|Order Orchestrator]]
- [[02-Work/Teams/MRG/02-services/taxipartnergateway/README|Taxi Partner Gateway]]
- [[02-Work/Teams/MRG/02-services/userservice/README|User Service]]

### Architecture
- [[02-Work/Teams/MRG/01-architecture/dependency-graph|Service Dependency Graph]]
- [[02-Work/Teams/MRG/01-architecture/diagrams/system-architecture|System Architecture]]

---

## 📞 Contacts

| Role | Name | Slack |
|------|------|-------|
| Platform Engineering Lead | Lukmanul Hakim | @lukmanulhakim |
| DBA Team | - | #dba-team |
| On-Call | - | #mrg-oncall |

---

## 📝 Changelog

| Date | Version | Changes |
|------|---------|---------|
| 2026-01-08 | 3.0 | Added: Blast Radius, RTO/RPO, Code Audit, External Deps, DB HA |
| 2026-01-08 | 3.0 | Created: Runbooks, Circuit Breaker Guide, Patroni Guide |
| 2026-01-08 | 2.0 | Added: Flow/Logic SPOF, Health Check Reference |
| 2025-12-xx | 1.0 | Initial SPOF assessment |

---

**Owner**: MRG Platform Engineering Team  
**Review Cycle**: Monthly
