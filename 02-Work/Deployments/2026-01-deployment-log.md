# 📅 January 2026 - Deployment Log

**Month:** January 2026  
**Total Hours:** 8.5 hours  
**Total Deployments:** 3  

---

## Summary by Team

| Team | Deployments | Hours | Success | Incident |
|------|------------|-------|---------|----------|
| UPG | 2 | 5.5 | 2 | 0 |
| MRG | 1 | 3.0 | 0 | 1 |
| **TOTAL** | **3** | **8.5 hours** | **2** | **1** |

---

## Deployment Details

### 2026-01-27: Cross Payment Deployment
- **Team:** UPG
- **Feature:** Cross Payment (GoPay, ShopeePay, Transaction, bbecv-adapter)
- **Scope:** Full rollout
- **Participating:** ✅ Yes
- **Start Time:** 21:30
- **End Time:** 23:00
- **Duration:** 1 hour 30 minutes (1.5 hours)
- **Outcome:** ✅ SUCCESS
- **Notes:** Smooth rollout, all services online, no incidents reported

---

### 2026-01-29: MyBB 6.32 Support ⚠️ INCIDENT
- **Team:** MRG
- **Feature:** MyBB version 6.32 support
- **Scope:** 
  - Corporate Card (whitelisted users)
  - Recharge with other payment
  - Landmark/Auto snap
  - Distance limitation
  - Profile/Default payment
  - Memory leak fix - Order stream legacy
- **Participating:** ✅ Yes
- **Start Time:** 22:00
- **End Time:** 01:00 (+1 day)
- **Duration:** 3 hours
- **PIC Developers:**
  - Corporate Card: @Kamal Firdaus
  - Recharge Payment: @Erik Rio Setiawan  
  - Landmark/Auto Snap: @Naufal
  - Distance Limitation: @Erik Rio Setiawan
  - Profile/Default Payment: @Isfan
  - Memory Leak Fix: Team
- **Outcome:** ⚠️ INCIDENT (P2)
- **Incident:** Order dengan CC Pre-auth tidak terdispatch ke BBD
- **Root Cause:** Redis caching key tidak ditemukan untuk request order ke BBD
- **Impact:** 283 users affected
- **Hotfix:** Deployed 06:42 WIB
- **Notes:** Postmortem documented, lessons learned captured
- **Related:** [[2026-01-29-postmortem-cc-preauth]]

---

### 2026-01-29: Cross Payment Deployment
- **Team:** UPG
- **Feature:** Cross Payment Enhancement
- **Scope:** 
  - Service MPG2
  - Service Card Payment
  - Service Mrg Gateway
  - Bugfix CC CP
  - Add Voucher ID
  - Bugfix tcash error handling
- **Participating:** ✅ Yes
- **Start Time:** 22:00
- **End Time:** 02:00 (+1 day)
- **Duration:** 4 hours
- **Outcome:** ✅ SUCCESS
- **Notes:** All services deployed successfully, no incidents reported

---

## Monthly Statistics

| Metric | Value |
|--------|-------|
| **Working Days** | 23 (excl. weekends) |
| **Deployments Worked** | 3 |
| **Total Overtime Hours** | 8.5 hours |
| **Average Hours per Deployment** | 2.83 hours |
| **Night Deployments (>17:00)** | 3 |
| **Weekend Deployments** | 0 |
| **Success Rate** | 67% |
| **Incidents** | 1 (P2) |

---

## 🔗 Related
- [[deployment-overtime-summary]] (Master Dashboard)
- [[monthly-deployment-overtime-2026]] (2026 Monthly Summary)
