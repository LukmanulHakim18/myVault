# 📊 Deployment Overtime Summary Dashboard

**Last Updated:** 2026-03-13  
**Purpose:** Master dashboard for all deployment overtime tracking across teams

---

## 📈 Overall Statistics

| Metric | Value |
|--------|-------|
| **Total Deployments** | 9 |
| **Total Overtime Hours** | 28.0 hours |
| **Total Overtime Minutes** | 1680 minutes |
| **Average Hours per Deployment** | 3.11 hours |
| **Deployments Participated** | 9 |
| **Success Rate** | 67% (Overall) |
| **Incidents** | 2 (1x P2 Jan, 1x Rollback Mar) |
| **Rollbacks** | 2 (1x partial Feb, 1x full Mar) |

---

## 📋 By Team

### UPG (Universal Payment Gateway)
- **Total Deployments Participated:** 3
- **Total Hours:** 6.0 hours
- **Last Deployment:** 2026-02-12 (Cross Payment Feature) ✅

### MRG (Meta Reservation Gateway)
- **Total Deployments Participated:** 5
- **Total Hours:** 16.0 hours
- **Last Deployment:** 2026-03-12 (FDS Feature) ⚠️ ROLLBACK

---

## 📅 Chronological List

| Date | Team | Feature | Duration | Outcome | Notes |
|------|------|---------|----------|---------|-------|
| 2026-01-27 | UPG | Cross Payment | 1.5 hours | ✅ SUCCESS | Smooth rollout |
| 2026-01-29 | MRG | MyBB 6.32 Support | 3.0 hours | ⚠️ INCIDENT (P2) | CC Pre-auth issue, 283 users impacted |
| 2026-01-29 | UPG | Cross Payment Enhancement | 4.0 hours | ✅ SUCCESS | MPG2, Card Payment, Mrg Gateway deployed |
| 2026-02-03 | MRG | MRG Services Update | 4.0 hours | ✅ SUCCESS | Api Gateway, Order/Payment/Notification deployed |
| 2026-02-12 | UPG | Cross Payment Feature | 0.5 hours | ✅ SUCCESS | Platform fee bugfix, pre-auth, cross payment fixes |
| 2026-02-12 | MRG | Bug Fixes & Tech Debt | 4.0 hours | ✅ SUCCESS | Multiple bug fixes and tech debt cleanup |
| 2026-02-19 | MRG | MyBB MYBB-5547 | 6.0 hours | ✅ SUCCESS | E-Wallet fixes, CC charging, Promo Coret rolled back |
| 2026-03-05 | MRG | MyBB Legacy (MRG 6.23.7) | ~6 hours | ⏳ PENDING | Enhancement release |
| 2026-03-12 | MRG+BBD+UPG | FDS Feature | 5.0 hours | ⚠️ ROLLBACK | GetWalletBalance error + CC→Cash issue on-trip |

---

```
Total Overtime Hours = Sum of all deployment hours outside 09:00-17:00
Date Range: 2026-01-27 - present
```

---

## 🔗 Related Files
- [[monthly-deployment-overtime-2026]] (Monthly Tracking 2026)
- [[2026-01-deployment-log]] (January 2026 Detail)
- [[2026-03-deployment-log]] (March 2026 Detail)
