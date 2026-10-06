# 📊 Deployment Overtime Summary Dashboard

**Last Updated:** 2026-07-09  
**Purpose:** Master dashboard for all deployment overtime tracking across teams

---

## 📈 Overall Statistics

| Metric | Value |
|--------|-------|
| **Total Deployments** | 15 |
| **Total Overtime Hours** | 38.0 hours + TBD |
| **Total Overtime Minutes** | 2280 minutes + TBD |
| **Average Hours per Deployment** | ~3.17 hours |
| **Deployments Participated** | 15 |
| **Success Rate** | TBD |
| **Incidents** | 2 (1x P2 Jan, 1x Rollback Mar) |
| **Rollbacks** | 2 (1x partial Feb, 1x full Mar) |

---

## 📋 By Team

### UPG (Universal Payment Gateway)
- **Total Deployments Participated:** 8
- **Total Hours:** 9.0 hours + TBD
- **Last Deployment:** 2026-07-08 (UPG Payment Release [2026.07.01] Pre-Auth by User ID Level) ✅ SUCCESS

### MRG (Meta Reservation Gateway)
- **Total Deployments Participated:** 5
- **Total Hours:** 16.0 hours
- **Last Deployment:** 2026-03-12 (FDS Feature) ⚠️ ROLLBACK

### Joint (MRG + UPG + BBD)
- **Total Deployments Participated:** 1
- **Total Hours:** 7.0 hours (simulation)
- **Last Deployment:** 2026-04-04 (FDS Pre-auth Simulation) ✅

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
| 2026-04-04 | MRG+BBD+UPG | FDS Pre-auth Simulation | 7.0 hours | ✅ SIMULATION | Step 6 & 8 skipped (IOT not working) |
| 2026-04-14 | UPG | UPG Payment Release | TBD | 🕐 SCHEDULED | Add initiated_by, remove FF, MPG1 VM→Kube |
| 2026-05-20 | UPG | [2026.05.03] Business Metric + GoPay/ShopeePay Fix | TBD | 🕐 SCHEDULED | 6 services: mpg-2, webhook-fwd, intermediate-ho, card-payment, routing-service, mrg-gateway |
| 2026-05-25 | UPG | [2026.05.04] GB Rent Revamp — Recharge extra/overtime | TBD | 🕐 SCHEDULED | mrg-gateway MR#1209, rollback tag v1.21.7 |
| 2026-07-01 | UPG | [2026.06.05] Nicepay inquiry-fail + timezone fix + tipping CC fallback | TBD | 🕐 SCHEDULED | mrg-gateway, nicepay-service, ecv-service, routing-service; rollback tags siap |
| 2026-07-08 | UPG | [2026.07.01] Pre-Auth by User ID Level (FDS) | 3.0 hours (22:00–01:00) | ✅ SUCCESS | routing-service, card-payment, mpg2, mrg-gateway; FDS pre-auth level BB ID + trx type gb_extra_overtime_daily |

---

```
Total Overtime Hours = Sum of all deployment hours outside 09:00-17:00
Date Range: 2026-01-27 - present
```

---

## 🔗 Related Files
- [[monthly-deployment-overtime-2026]] (Monthly Tracking 2026)
- [[2026-01-deployment-log]] (January 2026 Detail)
- [[2026-04-deployment-log]] (April 2026 Detail)
- [[2026-07-deployment-log]] (July 2026 Detail)
