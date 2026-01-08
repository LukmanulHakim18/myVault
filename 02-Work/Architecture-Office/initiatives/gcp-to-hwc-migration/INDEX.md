# 🚀 GCP to HWC Migration - Quick Navigation

**Migration Lead**: Lukmanul Hakim  
**Status**: ✅ Completed (with ongoing optimizations)  
**Date**: 2025

---

## 📑 Quick Links

### Core Documentation
- 📖 [**Main README**](./README.md) - Complete migration overview, timeline, and lessons learned
- 🧪 [**Cut Over Simulation**](./cut-over-simulation-30-july.md) - Regression testing results and findings
- 📊 [**Performance Analysis**](./performance/latency-comparison.md) - Latency comparison and optimization plans

### Issues & Resolutions
- 🐛 [Issue BB00215513](./issues/issue-BB00215513.md) - Marketing promo validation error
- 🐛 [Issue BB01267851](./issues/issue-BB01267851.md) - EzPay race condition (resolved)

### Integration Documentation
- 🗺️ [GMO Integration](./gmo-integration.md) - Landmark management coordination
- 💳 [UPG Marketing Migration](./upg-marketing-migration.md) - Payment and promo features

### Related Projects
- ✅ [[02-Work/Teams/MRG/01-architecture/design-docs/optimalisasi-placement/README|Optimalisasi Placement MRG]] - Drop-off & Pick-up Point Accuracy Improvement (Completed)

---

## 🎯 Quick Start Guide

### For New Team Members
1. Read [Main README](./README.md) first for complete context
2. Review [Cut Over Simulation](./cut-over-simulation-30-july.md) to understand what was tested
3. Check [Performance Analysis](./performance/latency-comparison.md) for current state
4. Review relevant issues in [Issues folder](./issues/)

### For Incident Response
1. Check [Issues folder](./issues/) for similar cases
2. Review [Main README - Issues Section](./README.md#-issues--resolutions)
3. Follow escalation path in documentation
4. Use Slack channels: #team-mrg, #team-upg, #gcp-hwc-migration

### For Performance Optimization
1. Start with [Latency Comparison](./performance/latency-comparison.md)
2. Review [Action Items in Main README](./README.md#-action-items--next-steps)
3. Check current monitoring dashboards
4. Coordinate with Hudan & Huawei Infrastructure Team

---

## 📊 Migration Stats

### Migration Timeline
- **Preparation**: Multiple weeks
- **Simulation**: 30 July 2025
- **Production Cutover**: [Actual Date]
- **Stabilization**: Ongoing

### Key Metrics
- **Uptime**: 99.9%
- **Data Loss**: 0%
- **Services Migrated**: All MRG + UPG services
- **Issues Found**: 5 (2 critical, resolved)
- **Performance**: Optimization in progress

---

## 🔍 Document Structure

```
gcp-to-hwc-migration/
├── README.md                          # Main documentation
├── INDEX.md                           # This file
├── cut-over-simulation-30-july.md     # Simulation results
├── gmo-integration.md                 # GMO coordination
├── upg-marketing-migration.md         # UPG marketing tasks
├── issues/
│   ├── issue-BB00215513.md           # Marketing error
│   └── issue-BB01267851.md           # EzPay race condition
└── performance/
    └── latency-comparison.md          # Performance analysis
```

---

## 🎓 Key Learnings

### What Worked Well ✅
1. DNS-first approach (no IP hardcoding)
2. Feature flags for maintenance mode
3. Comprehensive simulation before cutover
4. Strong cross-team collaboration

### What Needs Improvement 🔄
1. Performance testing should be more thorough
2. Earlier external partner coordination
3. ArgoCD should be ready before cutover
4. More comprehensive mobile client testing

### Technical Debt 📝
1. Performance optimization (highest priority)
2. ArgoCD configuration completion
3. Notification delivery improvement
4. Feature flags migration to Redis

---

## 🚨 Critical Information

### Ongoing Critical Issues
- **Performance**: Significant latency degradation vs GCP
  - See: [Performance Analysis](./performance/latency-comparison.md)
  - Priority: 🔴 Critical
  - Owner: Hudan + Huawei Team

### Monitoring & Alerts
- **Grafana**: [Dashboard Link]
- **Alerts**: Configure alerts for P95 > 1 second
- **On-call**: Use PagerDuty rotation

---

## 👥 Team & Contacts

### Leadership
- **Migration Lead**: Lukmanul Hakim

### MRG Team
- Erik, Alfian, Hudan, [others]

### UPG Team
- Eko, Erwin, Kamal, [others]

### External Partners
- **GMO**: Aldian Aby (internal liaison)
- **BBD**: IoT device management
- **Huawei**: Infrastructure support

---

## 📞 Support Channels

### Slack
- `#team-mrg` - MRG team discussions
- `#team-upg` - UPG team discussions
- `#gcp-hwc-migration` - Migration-specific channel
- `#architecture-office` - Cross-team architecture

### Escalation
1. **Technical**: Post in team channel
2. **Urgent**: Mention @lukmanulhakim
3. **Critical**: Use PagerDuty + phone escalation

---

## 📅 Review Cycle

This documentation is reviewed and updated:
- **Daily**: During active migration/optimization
- **Weekly**: During stabilization phase
- **Monthly**: After stabilization complete

---

## 🏷️ Tags

#migration #gcp #huawei-cloud #infrastructure #mrg #upg #platform-engineering

---

*Last Updated: 2025-08-04*  
*Next Review: Weekly until performance optimization complete*
