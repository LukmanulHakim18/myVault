# Postmortem: Order CC Pre-auth Tidak Terdispatch

**Date:** 2026-01-29  
**Priority:** P2  
**Postmortem Owner:** MRG Team  
**Affected Services:** MyBB  

---

## Summary

Order dengan pembayaran Credit Card Pre-auth tidak terdispatch ke BBD, menyebabkan order gantung pada state finding driver atau halaman schedule order.

---

## Impact

- **Users Affected:** 283 users
- **Duration:** ~2 jam (04:39 - 06:42)
- **Severity:** P2

---

## Timeline (WIB)

| Time | Event |
|------|-------|
| 04:39 | Laporan isu dari CRC/Helpdesk |
| 05:04 | Investigasi & Fixing dimulai |
| 06:42 | Deployment hotfix |
| 06:42 - 08:00 | Monitoring |

---

## Root Cause

**Redis caching key tidak ditemukan untuk keperluan request order ke BBD.**

Improvement pada booking flow menyebabkan order dengan CC yang di-pre-auth gagal terdispatch karena caching key yang diperlukan tidak tersedia.

---

## Detection

- **Method:** Laporan dari Helpdesk/CRC
- **Improvement:** Perlu monitoring order state 0 dari MyBB ke BBD

---

## Resolution

Hotfix pada order menggunakan CC pre-auth untuk memastikan caching key tersedia saat request ke BBD.

---

## Lessons Learned

1. ✅ Semua deployment harus tercatat di deployment plan dan dibahas pada risk analysis
2. ✅ Melengkapi PIC standby pada deployment plan
3. ✅ CC dengan kondisi pre-auth dimasukkan ke dalam scenario testing
4. ✅ Perlu monitoring order dari MyBB ke BBD untuk cek apakah order state 0 masuk ke BBD atau tidak

---

## Follow-up Tasks

| Task | Owner | Status | Tracking |
|------|-------|--------|----------|
| Update deployment plan template dengan risk analysis | TBD | 🔲 TODO | - |
| Add PIC standby requirement | TBD | 🔲 TODO | - |
| Add CC pre-auth to test scenarios | QA | 🔲 TODO | - |
| Implement monitoring order state 0 MyBB → BBD | MRG | 🔲 TODO | - |

---

## Related

- [[2026-01-deployment-log]] (Deployment Log)
- [[deployment-overtime-summary]] (Master Dashboard)

---

**Tags:** #postmortem #incident #mrg #p2 #cc-preauth
