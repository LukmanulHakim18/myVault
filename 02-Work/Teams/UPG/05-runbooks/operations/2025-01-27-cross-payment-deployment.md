# UPG Cross Payment Deployment - 2025-01-27

**Status:** 🔄 Pending  
**Deployment Time:** 21:30 (9:30 PM) Tonight  
**Date:** January 27, 2025

---

## Deployment Details

### Feature: Cross Payment (Full Rollout)
Rollout scope: **All corporate accounts (100% deployment)**

### Services Being Deployed
1. **Service GoPay** - Handle GoPay transaction data
2. **Service ShopeePay** - Handle ShopeePay transaction data  
3. **Service Transaction** - Payment change tracking & cross-payment logging
4. **Service bbecv-adapter** - Electronic Company Voucher (ECV) handling

### Payment Methods Supported
- **ECV** (Electronic Company Voucher) - B2B corporate payment method issued by Bluebird
- **GoPay** - Digital wallet payment
- **ShopeePay** - Digital wallet payment

### Business Context
- B2B transactions between corporate accounts
- Corporate customers rely on payment reliability
- High business impact if failures occur (service suspension risk)
- Reconciliation & audit trail critical for B2B operations

---

## Pre-Deployment Checklist
- [ ] Idempotency tracking in Transaction service validated
- [ ] ECV flow & routing logic tested
- [ ] Reconciliation queries prepared
- [ ] Monitoring/alerting dashboard active
- [ ] Incident response team on standby
- [ ] Rollback strategy confirmed

---

## Deployment Personnel

**Lukmanul Hakim Involvement:** ✅ Yes
- **Participating:** Yes
- **Role:** Deployment Lead/Engineer

---

## Deployment Timeline & Hours Tracking

**Start Time:** 21:30 (January 27, 2025)  
**End Time:** 23:00 (January 27, 2025)  
**Duration:** 1 hour 30 minutes (1.5 hours)

**Operational Hours Summary:**
- Start: 21:30
- End: 23:00
- **Total Hours:** 1.5 hours
- **Date(s) Spanned:** January 27, 2025 (Same day)

---

## Deployment Results
✅ **SUCCESS**

**Outcome:** ✅ Successful Deployment
- **Success/Failure:** SUCCESS
- **Status Code:** All Green
- **Details:** Deployment completed safely with all 4 services online
- **Timestamp:** 23:00 on January 27, 2025
- **Notes:** Smooth rollout to all corporate accounts, no incidents reported

---

## Key Contacts & Notes
- Full rollout to all corporate accounts simultaneously (high-risk, high-impact)
- Will receive update from Lukmanul after deployment completes

---

## Related Documents
- [[UPG-services-overview]]
- [[Payment-Gateway-Architecture]]
- [[ECV-System-Design]]
- [[monthly-deployment-overtime-2025]] (Monthly Tracking)
- [[2025-01-deployment-log]] (January Log)
