# 📅 February 2026 - Deployment Log

**Month:** February 2026  
**Total Hours:** 14.5 hours  
**Total Deployments:** 4 (all completed)  

---

## Summary by Team

| Team | Deployments | Hours | Success | Incident |
|------|------------|-------|---------|----------|
| MRG | 3 | 14.0 | 3 | 0 |
| UPG | 1 | 0.5 | 1 | 0 |
| **TOTAL** | **4** | **14.5** | **4** | **0** |

---

## Deployment Details

### 2026-02-03: MRG Services Deployment ✅ SUCCESS
- **Team:** MRG
- **Feature:** Multiple Services Update
- **Scope:** 
  - Api Gateway
  - Order Processor
  - Payment Processor
  - Notification Center
- **Participating:** ✅ Yes
- **Start Time:** 22:00
- **End Time:** 02:00 (+1 day)
- **Duration:** 4 hours
- **Outcome:** ✅ SUCCESS
- **Notes:** All services deployed successfully

### 2026-02-12: UPG Cross Payment Feature ✅ SUCCESS
- **Team:** UPG
- **Feature:** Cross Payment Implementation & Bug Fixes
- **Scope:**
  - Bugfix platform fee ECV by scheduler
  - Pre-auth incremental
  - Bug fix cross payment
- **Participating:** ✅ Yes
- **Start Time:** 21:30
- **End Time:** 22:00
- **Duration:** 0.5 hours
- **Outcome:** ✅ SUCCESS
- **Notes:** Cross payment feature deployment with multiple bug fixes

### 2026-02-12: MRG Services Deployment ✅ SUCCESS
- **Team:** MRG
- **Feature:** Multiple Bug Fixes & Tech Debt
- **Scope:**
  - Exclude promo discount amount on total info code
  - Create monitoring for failed order creation to BBD
  - Handle empty response when delete fav location
  - Exclude softbanned user when retry get passcode
  - Reset Semarang and Yogyakarta fare
  - Tech debt: validate tipping, update panic in 1 line, comment update in request booking, update gRPC registration
- **Participating:** ✅ Yes
- **Start Time:** 22:00
- **End Time:** 02:00 (+1 day)
- **Duration:** 4.0 hours
- **Outcome:** ✅ SUCCESS
- **Notes:** Bug fixes and tech debt cleanup deployment

### 2026-02-19: MyBB Deployment (MYBB-5547) ✅ SUCCESS
- **Team:** MRG / UPG
- **Feature:** MyBB Release (MYBB-5547)
- **Scope:**
  - E-Wallet outstanding label fix (EN & ID language support)
  - Order GB Now with E-Wallet Dana, Add Extra, Recharge payment fix
  - CC charging fix (fare + extra)
  - Outstanding E-Wallet label di home, account, confirmation page
  - Comment update in request_booking & Update gRPC Registration
  - Cancel GB Now Immediate & Advance handling
- **PIC Dev:** @Kamal Firdaus
- **Participating:** ✅ Yes
- **Start Time:** 22:00
- **End Time:** 03:59 (+1 day, 2026-02-20)
- **Duration:** 5 hours 59 minutes (~6 hours)
- **Outcome:** ✅ SUCCESS (with partial rollback)
- **Rollback:** Promo Coret Whitelist Marketing
- **Notes:** Deployment successful, one feature rolled back (Promo Coret Whitelist Marketing). All other features deployed successfully including E-Wallet fixes and CC charging improvements.

---

## Monthly Statistics

| Metric | Value |
|--------|-------|
| **Working Days** | TBD |
| **Deployments Worked** | 4 |
| **Total Overtime Hours** | 14.5 |
| **Average Hours per Deployment** | 3.63 |
| **Night Deployments (>17:00)** | 4 |
| **Weekend Deployments** | 0 |
| **Success Rate** | 100% |
| **Incidents** | 0 |
| **Rollbacks** | 1 (partial) |

---

## 🔗 Related
- [[deployment-overtime-summary]] (Master Dashboard)
- [[monthly-deployment-overtime-2026]] (2026 Monthly Summary)
