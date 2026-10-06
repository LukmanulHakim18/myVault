---
title: Meeting Prep - OCBC Disbursement Integration UPG
date: '2026-04-17'
team: UPG
tags:
  - meeting-prep
  - ocbc
  - disbursement
  - api
  - upg
status: draft
---
# Meeting Prep - OCBC Disbursement Integration UPG

**Tanggal:** 2026-04-17  
**Tim:** UPG  
**Topik:** Integrasi API REST OCBC untuk Disbursement

---

## Background

OCBC (PT Bank OCBC NISP, Tbk) akan menjadi channel disbursement pertama di UPG. Integrasi menggunakan **API REST** (Connect2OCBC platform). Semua tipe disbursement akan diintegrasikan:

- Real-time single transfer
- Bulk/batch disbursement
- Transfer internal sesama OCBC
- Transfer ke rekening bank lain (BI-FAST/RTGS)

---

## Mekanisme API REST Disbursement

### 1. Authentication

```
POST /oauth/token
Body: client_id + client_secret
Response: access_token (Bearer), expires_in
```

Token disimpan di DB beserta `expires_at`. Refresh otomatis saat expired.

### 2. Transfer Request

```
POST /disbursement/transfer
Header: Authorization: Bearer {token}
Body:
{
  "transaction_id": "UPG-TXN-001",   // idempotency key
  "amount": 150000,
  "currency": "IDR",
  "source_account": "xxx",
  "destination_account": "yyy",
  "destination_bank_code": "028",    // kalau ke bank lain
  "beneficiary_name": "John Doe",
  "description": "Refund order #123"
}
```

### 3. Response Flow

**Synchronous** (response langsung final):
```
← 200 OK { "status": "SUCCESS" | "FAILED", "bank_reference": "OCBC-REF-XYZ" }
```

**Asynchronous** (via callback):
```
← 200 OK { "status": "PENDING", "ref": "..." }
↓
OCBC → POST {callback_url}/disbursement/notify
       { "status": "SUCCESS" | "FAILED", "ref": "..." }
```

### 4. Status Inquiry

```
GET /disbursement/status?transaction_id=UPG-TXN-001
← { "status": "SUCCESS" | "PENDING" | "FAILED" }
```

---

## Sequence Diagram

```mermaid
sequenceDiagram
    autonumber
    participant App as MyBluebird App
    participant UPG as UPG Service
    participant DB as UPG Database
    participant OCBC as OCBC API

    rect rgb(240, 248, 255)
        Note over UPG, OCBC: Authentication
        UPG->>OCBC: POST /oauth/token (client_id, client_secret)
        OCBC-->>UPG: access_token (Bearer)
        UPG->>DB: Store token + expires_at
    end

    rect rgb(240, 255, 240)
        Note over App, OCBC: Disbursement Request
        App->>UPG: POST /v1/disburse (amount, destination, etc)
        UPG->>DB: Insert txn (status=PENDING, transaction_id=UPG-TXN-001)
        UPG->>OCBC: POST /disbursement/transfer (Bearer token)
        alt Token expired
            OCBC-->>UPG: 401 Unauthorized
            UPG->>OCBC: POST /oauth/token (refresh)
            OCBC-->>UPG: new access_token
            UPG->>OCBC: POST /disbursement/transfer (retry)
        end
        OCBC-->>UPG: 200 PENDING (bank_ref: OCBC-REF-XYZ)
        UPG->>DB: Update txn (status=PENDING, bank_ref)
        UPG-->>App: 200 Accepted (txn_id, status=PENDING)
    end

    rect rgb(255, 255, 240)
        Note over UPG, OCBC: Async Callback
        OCBC->>UPG: POST /callback/disbursement (status, bank_ref, signature)
        UPG->>UPG: Validate signature
        alt Signature valid
            UPG->>DB: Update txn (status=SUCCESS|FAILED)
            UPG-->>OCBC: 200 OK
        else Signature invalid
            UPG-->>OCBC: 400 Bad Request
        end
    end

    rect rgb(255, 240, 240)
        Note over UPG, OCBC: Timeout Handling
        UPG->>UPG: Scheduler - cek txn PENDING > X menit
        UPG->>OCBC: GET /disbursement/status?transaction_id=UPG-TXN-001
        OCBC-->>UPG: status SUCCESS | FAILED | PENDING
        UPG->>DB: Update txn status
    end

    rect rgb(248, 240, 255)
        Note over App, DB: Status Polling (opsional)
        App->>UPG: GET /v1/disburse/{txn_id}/status
        UPG->>DB: Query txn status
        DB-->>UPG: txn record
        UPG-->>App: status (SUCCESS | PENDING | FAILED)
    end
```

---

## Hal Kritis di Sisi UPG

| Concern | Handling |
|--------|---------|
| **Timeout** | Selalu inquiry ulang sebelum retry |
| **Idempotency** | `transaction_id` unik per transaksi |
| **Callback keamanan** | Validasi signature dari OCBC |
| **Reconciliation** | Match internal record vs status OCBC |
| **Retry logic** | Hanya retry jika status `PENDING`, bukan `FAILED` |

---

## Pertanyaan untuk Dikonfirmasi ke OCBC

1. Apakah API disbursement sudah **SNAP BI compliant** atau masih proprietary?
2. Response transfer **synchronous** atau **asynchronous** (via callback)?
3. Mekanisme **signature/HMAC** untuk validasi callback
4. Format `transaction_id` — ada constraint panjang/karakter?
5. Limit transaksi per request dan per hari
6. Apakah ada **sandbox environment** untuk testing?
7. Metode transfer yang didukung: BI-FAST, RTGS, SKN — beda endpoint atau parameter?
8. SLA waktu proses per metode transfer

---

## Referensi

- Kode Bank OCBC NISP: `028`
- SWIFT Code: `NISPIDJAXXX`
- Platform API: Connect2OCBC (`api.ocbc.com`)
- Regulasi: SNAP BI (SK Gubernur BI No. 23/10/KEP.GBI/2021)
