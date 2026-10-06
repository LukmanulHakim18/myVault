---
date: '2026-04-14'
type: meeting-notes
team: UPG
topic: Standarisasi SNAP
status: action-items-pending
---
# Meeting: Standarisasi SNAP - UPG
**Tanggal:** 2026-04-14  
**Tim:** UPG  
**Topic:** Standarisasi SNAP Response & Guidelines

---

## Output Meeting

### SNAP Response Standard

```json
{
    "reference_no": "12321312",
    "partner_reference_no": "ORD-12321321321",
    "status": "SUCCESS",
    "transaction_message": "Deny By Shopeepay"
}
```

**Notes:**
- `status`: `SUCCESS` atau `FAILED`
- `transaction_message`: pesan dari provider (tidak perlu di-wrap)

---

## Action Items (PR Tim UPG → Report ke Lukman)

| Priority | Task |
|----------|------|
| 🔴 HIGH | Create standarization SNAP based on UPG Behavior using Flipt (feature flag) |
| 🔴 HIGH | Create new repo for handling proto file (for guidelines / standard) |
| 🟡 MED | Don't need to wrap error message |
| 🟡 MED | Create SNAP with specific suffix endpoint: `snap_{func}_provider` |

---

## Keputusan Teknis

- **Feature flag:** menggunakan **Flipt**
- **Error message:** tidak perlu di-wrap, kirim langsung dari provider
- **Naming convention endpoint:** `snap_{func}_provider`
  - Contoh: `snap_charge_shopeepay`, `snap_charge_gopay`
- **Proto standardization:** dibuatkan repo dedicated baru

---

## Follow-up

- [ ] Tim UPG buat standarisasi SNAP + Flipt integration
- [ ] Tim UPG buat repo proto file baru
- [ ] Report progress ke Lukman
