---
title: 'Meeting: OCBC Disbursement SNAP API - Tim UPG'
date: '2026-04-27'
tags:
  - meeting
  - ocbc
  - snap
  - disbursement
  - upg
  - integration
status: active
source: Tech-Doc OCBC NISP SNAP v1.10
---
# Meeting: OCBC Disbursement SNAP API — Tim UPG
**Tanggal**: 2026-04-27
**Dokumen Referensi**: Tech-Doc OCBC NISP Standar Nasional Open API Pembayaran (SNAP) V1.10
**Scope**: Greenfield implementation — BIFAST, Online Transfer (OLT), RTGS, SKN
**Context**: Diskusi Form OCBC Disbursement untuk implementasi di tim UPG

---

## BAGIAN 1: INTEGRATION FLOW DIAGRAM

### 1.1 Authentication Flow

```mermaid
sequenceDiagram
    participant P as Partner (UPG)
    participant APIM as OCBC APIM
    participant V as Velocity

    Note over P,V: STEP 1 — Setup Credential (sekali saja)
    P->>APIM: GET /v1.0/authentication-credential/b2b
    APIM-->>P: { privateKey, publicKey }
    P->>APIM: GET /v1.0/transaction-credential/b2b
    APIM-->>P: { secretKey }
    Note over P: Kirim publicKey ke OCBC<br/>Simpan privateKey & secretKey

    Note over P,V: STEP 2 — Per Session: Get Token (TTL 900s)
    P->>APIM: POST /v1.0/authentication-signature/b2b<br/>{ clientID, timestamp, privateKey }
    APIM->>APIM: Generate signature (SHA256withRSA)
    APIM-->>P: { signature }

    P->>APIM: POST /v2.0/access-token/b2b<br/>Header: X-CLIENT-KEY, X-SIGNATURE, X-TIMESTAMP<br/>Body: { grantType, scope }
    APIM->>V: Processing
    V-->>APIM: Generate accessToken
    APIM-->>P: { accessToken, expiresIn: 900, tokenType: "Bearer" }

    Note over P: ⚠️ Refresh token sebelum 900 detik
```

**StringToSign (Token Signature):**
```
stringToSign = client_ID + "|" + X-TIMESTAMP
HmacValue    = SHA256withRSA(SecretKey, stringToSign)
```

**StringToSign (Transaction Signature per request):**
```
stringToSign = HTTPMethod + ":" + EndpointUrl + ":" + AccessToken + ":"
             + Lowercase(HexEncode(SHA-256(minify(RequestBody)))) + ":" + TimeStamp
HmacValue    = HMAC_SHA512(clientSecret, stringToSign)
```

---

### 1.2 BIFAST Transfer Flow (Inquiry → Submit → Status)

```mermaid
sequenceDiagram
    participant U as User/App
    participant P as Partner (UPG)
    participant APIM as OCBC APIM
    participant V as Velocity

    U->>P: Input beneficiary account + amount

    Note over P,V: PHASE 1 — Inquiry Beneficiary
    P->>P: Build Transaction Signature (HMAC_SHA512)
    P->>APIM: POST corporate/bifast/v1.0/account-inquiry-external<br/>{ beneficiaryBankCode, beneficiaryAccountNo,<br/>  partnerReferenceNo, additionalInfo.debitAccountNo }
    APIM->>V: Save log & forward
    V-->>APIM: Processing
    APIM-->>P: { referenceNo, beneficiaryAccountName,<br/>  beneficiaryBankName, currency }
    P->>U: Tampilkan nama & bank beneficiary

    Note over P,V: PHASE 2 — Submit Transfer
    U->>P: Konfirmasi transfer
    P->>P: Build new Transaction Signature
    P->>APIM: POST corporate/bifast/v1.0/transfer-interbank<br/>{ partnerReferenceNo, amount, beneficiaryAccountNo,<br/>  beneficiaryAccountName, beneficiaryBankCode,<br/>  sourceAccountNo, transactionDate,<br/>  originatorInfos, additionalInfo.referenceNo }
    APIM->>V: Save log & forward
    V-->>APIM: Processing
    APIM-->>P: { transactionStatus, referenceNo }

    alt transactionStatus = "00" SUCCESS
        P->>U: Tampilkan status SUKSES
    else transactionStatus = "01" Initiated / timeout
        Note over P,V: PHASE 3 — Cek Transfer Status
        P->>APIM: POST corporate/v1.0/transfer/status<br/>{ originalPartnerReferenceNo,<br/>  serviceCode: "18", transactionDate, amount }
        APIM->>V: Processing
        APIM-->>P: { latestTransactionStatus, transactionStatusDesc }
        P->>U: Update status ke user
    end
```

---

### 1.3 Online Transfer (OLT) Flow

```mermaid
sequenceDiagram
    participant U as User/App
    participant P as Partner (UPG)
    participant APIM as OCBC APIM
    participant V as Velocity

    U->>P: Input beneficiary + amount (max Rp100jt)

    Note over P,V: PHASE 1 — OLT Inquiry
    P->>APIM: POST corporate/olt/v1.0/account-inquiry-external<br/>{ beneficiaryBankCode, beneficiaryAccountNo,<br/>  additionalInfo.debitAccountNo, additionalInfo.amount }
    APIM->>V: Save log & forward
    APIM-->>P: { referenceNo, beneficiaryAccountName,<br/>  beneficiaryBankCode, beneficiaryBankName }
    P->>U: Tampilkan info beneficiary

    Note over P,V: PHASE 2 — OLT Submit
    U->>P: Konfirmasi
    P->>APIM: POST corporate/olt/v1.0/transfer-interbank<br/>{ partnerReferenceNo, amount, beneficiaryAccountName,<br/>  beneficiaryAccountNo, beneficiaryBankCode,<br/>  sourceAccountNo, originatorInfos,<br/>  additionalInfo.referenceNo,<br/>  additionalInfo.beneficiaryBankCityCode,<br/>  additionalInfo.beneficiaryBankAddress,<br/>  additionalInfo.beneficiaryBankNetworkClearingId }
    APIM->>V: Save log & forward
    APIM-->>P: { transactionStatus, referenceNo }

    alt transactionStatus != "00"
        P->>APIM: POST corporate/v1.0/transfer/status<br/>{ serviceCode: "18" }
        APIM-->>P: { latestTransactionStatus }
    end
    P->>U: Update status
```

---

### 1.4 RTGS Transfer Flow (Working Hours Only)

```mermaid
sequenceDiagram
    participant U as User/App
    participant P as Partner (UPG)
    participant APIM as OCBC APIM
    participant V as Velocity

    U->>P: Input transfer > Rp100jt (RTGS)
    P->>P: Validasi jam operasional (working hours)

    Note over P,V: PHASE 1 — RTGS Inquiry
    P->>APIM: POST corporate/rtgs/v1.0/account-inquiry-external<br/>{ beneficiaryBankCode, beneficiaryAccountNo,<br/>  additionalInfo.debitAccountNo }
    APIM->>V: Save log & forward
    APIM-->>P: { referenceNo, beneficiaryAccountName,<br/>  beneficiaryBankCode, beneficiaryBankName }
    P->>U: Tampilkan info beneficiary

    Note over P,V: PHASE 2 — RTGS Submit
    U->>P: Konfirmasi
    P->>APIM: POST corporate/v1.0/transfer-rtgs<br/>{ partnerReferenceNo, amount,<br/>  beneficiaryAccountName, beneficiaryAccountNo,<br/>  beneficiaryCustomerResidence, beneficiaryCustomerType,<br/>  sourceAccountNo, originatorInfos,<br/>  additionalInfo.senderName,<br/>  additionalInfo.bankRef (dari Inquiry referenceNo),<br/>  additionalInfo.beneficiaryNetworkClearingId,<br/>  additionalInfo.beneCategory, additionalInfo.remitterCategory }
    APIM->>V: Save log & forward
    APIM-->>P: { transactionStatus, referenceNo }

    Note over P: ⚠️ Final status bisa 1–3 hari
    loop Poll Transfer Status
        P->>APIM: POST corporate/v1.0/transfer/status<br/>{ serviceCode: "22" }
        APIM-->>P: { latestTransactionStatus }
        alt latestTransactionStatus = "00"
            P->>U: SUKSES
        else still pending
            P->>P: Tunggu, poll lagi
        end
    end
```

---

### 1.5 SKN Transfer Flow (Working Hours Only)

```mermaid
sequenceDiagram
    participant U as User/App
    participant P as Partner (UPG)
    participant APIM as OCBC APIM
    participant V as Velocity

    U->>P: Input transfer via SKN (max Rp1M)
    P->>P: Validasi jam operasional (working hours)

    Note over P,V: PHASE 1 — SKN Inquiry
    P->>APIM: POST corporate/skn/v1.0/account-inquiry-external<br/>{ beneficiaryBankCode, beneficiaryAccountNo,<br/>  additionalInfo.debitAccountNo,<br/>  additionalInfo.amount: "1.00" (default) }
    APIM->>V: Save log & forward
    APIM-->>P: { referenceNo, beneficiaryAccountName,<br/>  beneficiaryBankCode, beneficiaryBankName }
    P->>U: Tampilkan info beneficiary

    Note over P,V: PHASE 2 — SKN Submit
    U->>P: Konfirmasi
    P->>APIM: POST corporate/v1.0/transfer-skn<br/>{ partnerReferenceNo, amount,<br/>  beneficiaryAccountName, beneficiaryAccountNo,<br/>  beneficiaryCustomerResidence, beneficiaryCustomerType,<br/>  sourceAccountNo, originatorInfos,<br/>  additionalInfo.senderName,<br/>  additionalInfo.referenceNo (dari Inquiry),<br/>  additionalInfo.regulatorResidentStatus,<br/>  additionalInfo.regulatorRemitterCategory,<br/>  additionalInfo.regulatorBeneCategory,<br/>  additionalInfo.beneficiaryBankCityCode,<br/>  additionalInfo.beneficiaryBankNetworkClearingId }
    APIM->>V: Save log & forward
    APIM-->>P: { transactionStatus, referenceNo }

    Note over P: ⚠️ Final status bisa sampai 1–3 hari
    P->>APIM: POST corporate/v1.0/transfer/status<br/>{ serviceCode: "23" }
    APIM-->>P: { latestTransactionStatus }
    P->>U: Update status
```

---

### 1.6 Transfer Status — Decision Flow

```mermaid
sequenceDiagram
    participant P as Partner (UPG)
    participant APIM as OCBC APIM

    Note over P: Trigger: Timeout / "Request In Progress" / Non-final status

    P->>APIM: POST corporate/v1.0/transfer/status<br/>{ originalPartnerReferenceNo,<br/>  serviceCode, transactionDate, amount }
    APIM-->>P: Response

    alt responseMessage = "Successful"
        Note over P: Cek latestTransactionStatus
        alt latestTransactionStatus = "00" & desc = "SUCCESS"
            P->>P: ✅ FINAL — Update DB sebagai SUCCESS
        else latestTransactionStatus = "01" Initiated
            P->>P: Retry poll dalam beberapa menit
        else latestTransactionStatus = "03" Pending
            P->>P: Retry poll, max 15 menit
        else latestTransactionStatus = "05" Cancelled
            P->>P: ✅ FINAL — Update DB sebagai CANCELLED
        end
    else responseMessage = "Transaction Not Found"
        P->>P: Cek statement / History List API
        Note over P: Jika > 24H masih not found:<br/>Hubungi OCBC
    end
```

---

### 1.7 Error Handling Flow (Best Practice OCBC)

```mermaid
sequenceDiagram
    participant P as Partner (UPG)
    participant APIM as OCBC API Service
    participant BE as OCBC API Backend

    Note over P,BE: PRE-TRANSACTION
    P->>APIM: Hit Transfer Submit API
    APIM->>BE: API Validation
    alt Validation FAILED
        BE-->>APIM: Error response
        APIM-->>P: Error (4xx) — Final Status
        P->>P: Create new transaction if needed
    else Validation SUCCESS
        BE->>BE: Process Transaction
        BE-->>APIM: Request in Progress
        APIM-->>P: Response (200/202)
    end

    Note over P,BE: POST-TRANSACTION PROCESSING
    loop Check Status (max 15 menit interval)
        P->>APIM: Hit Transfer Status API
        APIM->>BE: Check status
        alt Status = Final (SUCCESS/CANCELLED)
            BE-->>APIM: Transaction Status Result
            APIM-->>P: Final status
            P->>P: Update DB & notify user
        else Still "Request in Progress" < 24H
            BE-->>APIM: Request in Progress
            APIM-->>P: Still processing
            P->>P: Wait & retry
        else Still pending > 24H
            P->>P: Contact OCBC<br/>Update transaction status manually
        end
    end
```

---

## BAGIAN 2: FIELD MAPPING PER CHANNEL

### Common Request Header (SEMUA Transfer API)

| Header | Attr | Keterangan |
|---|---|---|
| `Content-Type` | **M** | `application/json` |
| `Authorization` | **M** | `Bearer {accessToken}` |
| `X-PARTNER-ID` | **M** | Sama dengan ClientID dari OCBC |
| `X-SIGNATURE` | **M** | Dari Transaction Signature (HMAC_SHA512) |
| `X-TIMESTAMP` | **M** | ISO 8601: `yyyy-MM-ddTHH:mm:ss.SSSTZD` |
| `X-EXTERNAL-ID` | **M** | Numeric string, **unik per hari** |
| `CHANNEL-ID` | **M** | `"API"` |

---

### 2.1 BIFAST Inquiry
`POST corporate/bifast/v1.0/account-inquiry-external`

| Field | Type | Attr | Length | Keterangan |
|---|---|---|---|---|
| `beneficiaryBankCode` | String | **M** | 8 | BIC Code bank tujuan |
| `beneficiaryAccountNo` | String | **M** | 34 | No rekening tujuan |
| `partnerReferenceNo` | String | **M** | 64 | 4-digit prefix OCBC + ID unik |
| `additionalInfo.debitAccountNo` | String | **M** | 34 | Rekening sumber |
| `additionalInfo.amount` | String | O | 16,2 | Opsional di v1.10 |
| `additionalInfo.trxPurpose` | String | **M** | 2 | `01`=Investasi `02`=Transfer Wealth `03`=Purchase `99`=Others |

> **Response kunci**: `referenceNo` → **WAJIB disimpan** untuk dipakai di Submit

---

### 2.2 BIFAST Submit
`POST corporate/bifast/v1.0/transfer-interbank` | Max Rp250jt | Fee Rp2.500 | 24x7

| Field | Type | Attr | Length | Keterangan |
|---|---|---|---|---|
| `partnerReferenceNo` | String | **M** | 64 | Harus unik global |
| `amount.value` | String | **M** | 16,2 | Max Rp 250.000.000 |
| `amount.currency` | String | **M** | 3 | `IDR` |
| `beneficiaryAccountName` | String | **M** | 100 | Dari Inquiry response |
| `beneficiaryAccountNo` | String | **M** | 34 | |
| `beneficiaryBankCode` | String | **M** | 8 | BIC |
| `beneficiaryBankName` | String | O | 50 | |
| `beneficiaryEmail` | String | **C** | 50 | Wajib jika `sendEmailNotification=true` |
| `currency` | String | **M** | 3 | `IDR` |
| `customerReference` | String | O | 16 | Ref internal Partner |
| `sourceAccountNo` | String | **M** | 19 | Rekening debit |
| `transactionDate` | String | **M** | 25 | ISO 8601 (bisa future date) |
| `originatorInfos[0].originatorCustomerNo` | String | **M** | 34 | PPATK — ultimate sender acct no |
| `originatorInfos[0].originatorCustomerName` | String | **M** | 100 | PPATK — ultimate sender name |
| `originatorInfos[0].originatorBankCode` | String | **M** | 8 | PPATK — bank code |
| `additionalInfo.referenceNo` | String | **M** | 64 | **Dari BIFAST Inquiry response** |
| `additionalInfo.transactionPurpose` | String | **M** | 2 | `01`–`03`, `99` |
| `additionalInfo.remark` | String | O | 50 | |
| `additionalInfo.sendEmailNotification` | Boolean | **M** | — | `true`/`false` |

---

### 2.3 Online Transfer (OLT) Inquiry
`POST corporate/olt/v1.0/account-inquiry-external`

| Field | Type | Attr | Length | Keterangan |
|---|---|---|---|---|
| `beneficiaryBankCode` | String | **M** | 8 | Kode numerik bank |
| `beneficiaryAccountNo` | String | **M** | 34 | |
| `partnerReferenceNo` | String | **M** | 64 | |
| `additionalInfo.debitAccountNo` | String | **M** | 34 | Rekening sumber |
| `additionalInfo.amount` | String | **M** | 16,2 | Transfer amount |
| `additionalInfo.customerReference` | String | O | 16 | |

---

### 2.4 OLT Submit
`POST corporate/olt/v1.0/transfer-interbank` | Max Rp100jt/trx, Rp500jt/hari | Fee Rp6.500 | 24x7

> Field sama dengan BIFAST Submit, dengan **tambahan khusus OLT**:

| Field | Type | Attr | Keterangan |
|---|---|---|---|
| `additionalInfo.beneficiaryBankCityCode` | String | **M** | CITY_ID — **minta attachment ke tim OCBC** |
| `additionalInfo.beneficiaryBankAddress` | String | **M** | **Minta ke tim OCBC** |
| `additionalInfo.beneficiaryBankNetworkClearingId` | String | **M** | BI_CLEARING_ID — **minta ke tim OCBC** |
| `additionalInfo.referenceNo` | String | **M** | Dari OLT Inquiry response |
| `feeType` | String | O | `OUR` (default) / `BEN` / `SHA` |

---

### 2.5 RTGS Inquiry
`POST corporate/rtgs/v1.0/account-inquiry-external` | Min Rp100.000.001 | Working hours only

| Field | Type | Attr | Length | Keterangan |
|---|---|---|---|---|
| `beneficiaryBankCode` | String | **M** | 8 | |
| `beneficiaryAccountNo` | String | **M** | 34 | |
| `partnerReferenceNo` | String | **M** | 64 | |
| `additionalInfo.debitAccountNo` | String | **M** | 34 | |

> **Response kunci**: `referenceNo` → dipakai sebagai **`bankRef`** di RTGS Submit

---

### 2.6 RTGS Submit
`POST corporate/v1.0/transfer-rtgs` | Min Rp100.000.001 | Fee Rp25.000 | Working hours

> Field dasar sama dengan BIFAST Submit, dengan **tambahan RTGS-specific**:

| Field | Type | Attr | Keterangan |
|---|---|---|---|
| `beneficiaryCustomerResidence` | String | **M** | `1`=Indonesia, `2`=Non-Indonesia |
| `beneficiaryCustomerType` | String | **M** | `1`=Individual, `2`=Corp, `3`=Govt |
| `kodepos` | String | O | Kodepos pengirim |
| `senderCustomerResidence` | String | O | `1`/`2` |
| `senderCustomerType` | String | O | `1`/`2`/`3` |
| `additionalInfo.senderName` | String | **M** | Nama pengirim |
| `additionalInfo.beneficiaryBankBranch` | String | **M** | **Minta ke OCBC** |
| `additionalInfo.beneficiaryBankAddress` | String | **M** | **Minta ke OCBC** |
| `additionalInfo.beneficiaryNetworkClearingId` | String | **M** | RTGS Member Code — **minta ke OCBC** |
| `additionalInfo.beneficiaryCityCode` | String | **M** | 4 digit — **minta ke OCBC** |
| `additionalInfo.beneCategory` | String | **M** | Appendix 6.2 (A0=Perorangan, E0=Perusahaan...) |
| `additionalInfo.remitterCategory` | String | **M** | Appendix 6.2 |
| `additionalInfo.bankRef` | String | **M** | Dari RTGS Inquiry `referenceNo` |

---

### 2.7 SKN Inquiry
`POST corporate/skn/v1.0/account-inquiry-external` | Max Rp1M | Working hours

> Sama seperti RTGS Inquiry. `additionalInfo.amount` default `1.00`.

---

### 2.8 SKN Submit
`POST corporate/v1.0/transfer-skn` | Max Rp1M | Fee Rp2.000 | Working hours

> Field dasar sama dengan BIFAST Submit + **tambahan SKN-specific** (paling banyak field regulasi):

| Field | Type | Attr | Keterangan |
|---|---|---|---|
| `beneficiaryCustomerResidence` | String | **M** | `1`/`2` |
| `beneficiaryCustomerType` | String | **M** | `1`/`2`/`3` |
| `additionalInfo.senderName` | String | **M** | Nama pengirim |
| `additionalInfo.beneficiaryBankCityCode` | String | **M** | **Minta ke OCBC** |
| `additionalInfo.beneficiaryBankBranchName` | String | **M** | **Minta ke OCBC** |
| `additionalInfo.beneficiaryBankAddress` | String | **M** | **Minta ke OCBC** |
| `additionalInfo.beneficiaryBankNetworkClearingId` | String | **M** | BI_CLEARING_ID — **minta ke OCBC** |
| `additionalInfo.regulatorResidentStatus` | String | **M** | `Y`=Resident, `N`=Non-resident |
| `additionalInfo.regulatorRemitterCategory` | String | **M** | Appendix 6.2 |
| `additionalInfo.regulatorBeneCategory` | String | **M** | Appendix 6.2 |
| `additionalInfo.validationCheck` | String | **M** | Default `N` |
| `additionalInfo.referenceNo` | String | **M** | Dari SKN Inquiry response |

---

### 2.9 Transfer Status API
`POST corporate/v1.0/transfer/status`

| Field | Type | Attr | Keterangan |
|---|---|---|---|
| `originalPartnerReferenceNo` | String | **M** | `partnerReferenceNo` dari Submit |
| `originalReferenceNo` | String | O | `referenceNo` dari Submit response |
| `originalExternalId` | String | O | `X-EXTERNAL-ID` dari header Submit |
| `serviceCode` | String | **M** | `17`=Intrabank, `18`=BIFAST/OLT, `22`=RTGS, `23`=SKN |
| `transactionDate` | String | **M** | ISO 8601 |
| `amount.value` | String | **M** | |
| `amount.currency` | String | **M** | `IDR` |

**`latestTransactionStatus` values:**

| Code | Status | Keterangan |
|---|---|---|
| `00` | **SUCCESS** | Final |
| `01` | Initiated | Masih proses |
| `02` | Suspect/Paying | — |
| `03` | Pending | — |
| `05` | Cancelled | Final |
| `06` | Failed | Final |
| `07` | Not Found | Contact OCBC |

### Service Code Reference

| Code | Layanan |
|---|---|
| 11 | Balance Inquiry |
| 12 | History List |
| 15 | Internal Inquiry |
| 16 | BIFAST / OLT / RTGS / SKN Inquiry |
| 17 | Intrabank Submit |
| 18 | BIFAST & OLT Submit |
| 22 | RTGS Submit |
| 23 | SKN Submit |
| 36 | Transfer Status |

---

## BAGIAN 3: IMPLEMENTATION CHECKLIST

### A. Pre-Development Setup
- [ ] Hubungi `API_Solutions@ocbc.id` untuk dapatkan `clientID` dan `clientSecret`
- [ ] Generate RSA key pair: `openssl genrsa -out rsa.private 1024` → export public key
- [ ] Kirim `publicKey` ke OCBC, simpan `privateKey` aman di sistem sendiri
- [ ] **Minta attachment dari OCBC**: bankCityCode, bankAddress, networkClearingId (wajib OLT, RTGS, SKN)
- [ ] **Minta 4-digit prefix** untuk `partnerReferenceNo` dari OCBC
- [ ] Inject certificate **mTLS** dari OCBC ke backend (domain `snapapi.ocbc.id` produksi)
- [ ] Sandbox URL: `https://developer.ocbc.id/snap/apis/`
- [ ] Production URL: `https://snapapi.ocbc.id/`

### B. Authentication Layer
- [ ] Implementasi Token generation (4 step)
- [ ] Implementasi **auto-refresh** token sebelum 900 detik habis
- [ ] Implementasi Transaction Signature per request (HMAC_SHA512)
- [ ] Simpan `secretKey` secara aman (env variable / vault)

### C. Request Building
- [ ] Generator `partnerReferenceNo` yang **globally unique** — format: `{prefix}{timestamp}{uuid}`
- [ ] Generator `X-EXTERNAL-ID` yang **unik per hari** (reset setiap hari / rolling)
- [ ] `transactionDate` format ISO 8601: `yyyy-MM-ddTHH:mm:ss.SSSTZD`
- [ ] **BIFAST/OLT**: Inquiry → simpan `referenceNo` → kirim ke `additionalInfo.referenceNo` di Submit
- [ ] **RTGS**: Inquiry → simpan `referenceNo` → kirim sebagai `additionalInfo.bankRef` di Submit
- [ ] **SKN**: Inquiry → simpan `referenceNo` → kirim ke `additionalInfo.referenceNo` di Submit
- [ ] Selalu isi `originatorInfos` (kepatuhan PPATK — pasal 8 ayat 5 UU No.3/2011)

### D. Error Handling & Retry Logic
- [ ] **Timeout / 504**: Jangan buat transaksi baru — hit Transfer Status API dulu
- [ ] **Request In Progress (202)**: Polling Transfer Status tiap X menit, max 15 menit sebelum eskalasi
- [ ] **Duplicate `partnerReferenceNo` (409-01)**: Submit sebelumnya sudah sukses — cek status, jangan retry
- [ ] **Duplicate `X-EXTERNAL-ID` (409-00)**: Generate `X-EXTERNAL-ID` baru
- [ ] Status **Request In Progress > 24 jam**: hubungi OCBC
- [ ] Implementasi **idempotency**: simpan mapping `partnerReferenceNo ↔ status` di database

### E. RTGS & SKN Khusus
- [ ] Validasi jam operasional di backend sebelum submit (working hours only)
- [ ] EOD window BIFAST/OLT (00.00–05.01): akan return "Request In Progress" — inform user
- [ ] Final status RTGS/SKN bisa 1–3 hari — jangan retry sebelum cek Transfer Status
- [ ] Siapkan data regulasi SKN: `regulatorResidentStatus`, `regulatorRemitterCategory`, `regulatorBeneCategory`
- [ ] Siapkan `beneCategory` + `remitterCategory` RTGS dari Appendix 6.2

### F. Testing & Go-Live
- [ ] Test semua channel di Sandbox
- [ ] Selesaikan **ASPI Functional Scenario & Developer Site Testing** di ASPI Portal sebelum produksi
- [ ] Test Transfer Status API untuk skenario: success, pending, timeout, not found
- [ ] Verifikasi signature calculation dengan Postman (`/v1.0/transaction-signature/b2b` — testing only)

---

## BAGIAN 4: FLOW DIAGRAMS — Mermaid Flowchart (dari PDF)

> Konversi dari diagram asli PDF OCBC SNAP v1.10

---

### 4.1 Token Flow to Service (Hal. 10)

```mermaid
flowchart TD
    Start([Start]) --> A1

    subgraph ObtainAccess["Obtain Access to OCBC API Services"]
        A1["① Hit API Token Signature\nPOST /v1.0/authentication-signature/b2b"]
        A1 --> A2["② APIM: Processing"]
        A2 --> A3["③ APIM: Generate Signature\nSHA256withRSA"]
        A3 --> A4["④ Partner: Obtain signature\nfrom OCBC"]
        A4 --> A5["⑤ Hit API: Token\nPOST /v2.0/access-token/b2b"]
        A5 --> A6["⑥ APIM: Processing"]
        A6 --> A7["⑦ APIM/Velocity: Generate accessToken\nTTL = 900 detik"]
        A7 --> A8["⑧ Partner: Obtain accessToken\nfrom OCBC ✅"]
    end

    A8 --> A9

    subgraph AccessService["Proceed to Access Service"]
        A9["⑨ Hit API: History List\nPOST corporate/v1.0/transaction-history-list"]
        A9 --> A10["⑩ APIM: Save to log\n& forward to Velocity"]
        A10 --> A11["⑪ Velocity: Processing"]
        A11 --> A12["⑫ APIM: Respond to Partner\nwith Response"]
        A12 --> A13["⑬ Partner: Show Transaction\nHistory List in Platform"]
    end

    A13 --> End([End])

    style ObtainAccess fill:#fff3cd,stroke:#f0ad4e,color:#000
    style AccessService fill:#d4edda,stroke:#28a745,color:#000
    style Start fill:#dc3545,color:#fff,stroke:#dc3545
    style End fill:#28a745,color:#fff,stroke:#28a745
```

---

### 4.2 Balance Inquiry Flow (Hal. 21)

```mermaid
flowchart TD
    Start([Start]) --> B1

    subgraph BalanceFlow["Balance Inquiry Flow"]
        B1["① Partner: Hit API Balance Inquiry\nPOST corporate/v1.0/balance-inquiry"]
        B1 --> B2["② APIM: Save to log\n& forward to Velocity"]
        B2 --> B3["③ Velocity: Processing"]
        B3 --> B4["④ APIM: Respond to Partner\n{ availableBalance, ledgerBalance }"]
        B4 --> B5["⑤ Partner: Show Account Balance\nin Partner Platform"]
    end

    B5 --> End([End])

    style BalanceFlow fill:#cce5ff,stroke:#004085,color:#000
    style Start fill:#dc3545,color:#fff,stroke:#dc3545
    style End fill:#28a745,color:#fff,stroke:#28a745
```

---

### 4.3 History List Flow (Hal. 25)

```mermaid
flowchart TD
    Start([Start]) --> H1

    subgraph HistoryFlow["History List Flow"]
        H1["① Partner: Hit API History List\nPOST corporate/v1.0/transaction-history-list"]
        H1 --> H2["② APIM: Save to log\n& forward to Velocity"]
        H2 --> H3["③ Velocity: Processing"]
        H3 --> H4["④ APIM: Respond to Partner\n{ detailData, status, type }"]
        H4 --> H5["⑤ Partner: Show Transaction\nHistory List in Partner Platform"]
    end

    H5 --> End([End])

    style HistoryFlow fill:#e2d9f3,stroke:#6f42c1,color:#000
    style Start fill:#dc3545,color:#fff,stroke:#dc3545
    style End fill:#28a745,color:#fff,stroke:#28a745
```

---

### 4.4 API Transfer Flow — Generic (Hal. 31)

> Berlaku untuk: **BIFAST, Online Transfer, SKN, RTGS, Intrabank**

```mermaid
flowchart TD
    Start([Start]) --> T1

    subgraph InquiryPhase["Phase 1 — Inquiry Beneficiary"]
        T1["① Partner: Hit API Inquiry\n(BIFAST/OLT/RTGS/SKN/Internal)\nPOST ...account-inquiry-external"]
        T1 --> T2["② APIM: Save to log\n& forward to Velocity"]
        T2 --> T3["③ Velocity: Processing\n(validasi rekening tujuan)"]
        T3 --> T4["④ APIM: Respond to Partner\n{ referenceNo, beneficiaryAccountName,\nbenefiiciaryBankName }"]
        T4 --> T5["⑤ Partner: Tampilkan nama &\nbank beneficiary di Platform\n✅ Simpan referenceNo!"]
    end

    T5 --> T6

    subgraph SubmitPhase["Phase 2 — Proceed Transfer"]
        T6["⑥ Partner: Hit API Submit\n(BIFAST/OLT/RTGS/SKN/Intrabank)\nSertakan referenceNo dari Inquiry"]
        T6 --> T7["⑦ APIM: Save to log\n& forward to Velocity"]
        T7 --> T8["⑧ Velocity: Processing\n(eksekusi transfer)"]
        T8 --> T9["⑨ APIM: Respond to Partner\n{ transactionStatus, referenceNo }"]
        T9 --> T10["⑩ Partner: Tampilkan Status Transfer\ndi Platform"]
    end

    T10 --> End([End])

    style InquiryPhase fill:#fff3cd,stroke:#f0ad4e,color:#000
    style SubmitPhase fill:#d4edda,stroke:#28a745,color:#000
    style Start fill:#dc3545,color:#fff,stroke:#dc3545
    style End fill:#28a745,color:#fff,stroke:#28a745
```

---

### 4.5 Transfer Status Flow + Decision Tree (Hal. 78)

```mermaid
flowchart TD
    Start([Hit Transfer Status API\nPOST corporate/v1.0/transfer/status]) --> CheckResp{responseMessage?}

    CheckResp -->|"✅ Successful"| CheckStatus{"Cek\nlatestTransactionStatus\n& transactionStatusDesc"}
    CheckResp -->|"❌ Transaction Not Found"| NotFound["Check Statement /\nConfirm to OCBC\nvia History List API"]

    CheckStatus -->|"00 & SUCCESS"| FinalSuccess(["✅ FINAL STATUS\nSUKSES"])
    CheckStatus -->|"01 & Initiated"| Retry["🔄 Retry Poll\n(beberapa menit)"]
    CheckStatus -->|"03 & Pending"| Retry
    CheckStatus -->|"05 & Cancelled"| FinalCancel(["❌ FINAL STATUS\nCANCELLED"])

    Retry --> Start

    NotFound --> Confirm{Status\nterverifikasi?}
    Confirm -->|"Ya"| FinalSuccess
    Confirm -->|"Tidak"| ContactOCBC["📞 Hubungi OCBC\nClient Service"]

    style Start fill:#0d6efd,color:#fff,stroke:#0d6efd
    style FinalSuccess fill:#28a745,color:#fff,stroke:#28a745
    style FinalCancel fill:#dc3545,color:#fff,stroke:#dc3545
    style ContactOCBC fill:#fd7e14,color:#fff,stroke:#fd7e14
```

---

### 4.6 Error Handling Flow for Partner (Hal. 84)

```mermaid
flowchart TD
    Start([Start]) --> Perform

    subgraph PreTrx["Pre-Transaction"]
        Perform["Partner: Perform Fund Transfer"] --> HitSubmit["Partner: Hit API\nTransfer Submit"]
        HitSubmit --> Validation{"OCBC Backend:\nAPI Validation"}
        Validation -->|"❌ Failed / Error"| ErrorResp["OCBC: Error Response\n(4xx — Final Status)"]
        Validation -->|"✅ Success"| ProcessTrx["OCBC: Process Transaction"]
        ProcessTrx --> InProgress["Status: Request in Progress"]
    end

    ErrorResp --> CreateNew{"Buat Transaksi\nBaru?"}
    CreateNew -->|"Ya (jika error pre-trx)"| Perform
    CreateNew -->|"Tidak"| End

    InProgress --> CheckStatus

    subgraph PostTrx["Post-Transaction Processing"]
        CheckStatus["Partner: Perform Check\nStatus of Transaction"] --> HitStatus["Partner: Hit API\nTransfer Status"]
        HitStatus --> StatusResult{"Transaction\nStatus Result?"}
        StatusResult -->|"✅ Final Status"| UpdateStatus["Partner: Update\nTransaction Status"]
        StatusResult -->|"🔄 Request in Progress\n< 15 menit"| CheckStatus
        StatusResult -->|"⏳ Request in Progress\n15 menit – 24 jam"| WaitLonger["Tunggu &\nRetry Berkala"]
        WaitLonger --> HitStatus
        StatusResult -->|"⚠️ Still Pending\n> 24 jam"| ContactOCBC["📞 Contact OCBC\nClient Service"]
    end

    ContactOCBC --> UpdateStatus
    UpdateStatus --> End([End])

    style PreTrx fill:#fff3cd,stroke:#f0ad4e,color:#000
    style PostTrx fill:#cce5ff,stroke:#004085,color:#000
    style Start fill:#dc3545,color:#fff,stroke:#dc3545
    style End fill:#28a745,color:#fff,stroke:#28a745
    style ContactOCBC fill:#fd7e14,color:#fff,stroke:#fd7e14
    style UpdateStatus fill:#d4edda,stroke:#28a745,color:#000
```

---

### 4.7 Transaction Signature Generation Flow (Hal. 18-20)

```mermaid
flowchart TD
    Start([Sebelum setiap API Request]) --> TS1

    subgraph TxnSig["Transaction Signature Calculation"]
        TS1["Ambil komponen:"] --> TS2
        TS2["• HTTPMethod: POST\n• EndpointUrl: /corporate/...\n• AccessToken: Bearer ...\n• RequestBody (JSON)\n• TimeStamp: ISO 8601"]

        TS2 --> TS3["Step 1: Minify RequestBody\n(hapus whitespace, \\r, \\n, \\t)"]
        TS3 --> TS4["Step 2: SHA-256 hash\nMinified RequestBody"]
        TS4 --> TS5["Step 3: HexEncode → Lowercase"]
        TS5 --> TS6["Step 4: Build StringToSign\nHTTPMethod + ':' + EndpointUrl + ':'\n+ AccessToken + ':'\n+ Lowercase(HexEncode(SHA256(body))) + ':'\n+ TimeStamp"]
        TS6 --> TS7["Step 5: HMAC_SHA512\n(clientSecret, StringToSign)\n→ X-SIGNATURE"]
    end

    TS7 --> TS8["Masukkan ke Request Header:\nX-SIGNATURE: {hasil}"]
    TS8 --> End([Kirim API Request])

    style TxnSig fill:#f8d7da,stroke:#721c24,color:#000
    style Start fill:#0d6efd,color:#fff,stroke:#0d6efd
    style End fill:#28a745,color:#fff,stroke:#28a745
```

---

### 4.8 Ringkasan Alur Lengkap — End-to-End Disbursement

```mermaid
flowchart TD
    Start([Mulai Disbursement]) --> Auth

    subgraph Auth["1. Authentication"]
        GetSig["GET Token Signature"] --> GetToken["GET Access Token\n(valid 900 detik)"]
    end

    Auth --> SelectChannel

    subgraph SelectChannel["2. Pilih Channel"]
        Ch{Pilih berdasarkan\namount & SLA}
        Ch -->|"≤ Rp100jt\n24x7"| OLT["Online Transfer (OLT)"]
        Ch -->|"≤ Rp250jt\n24x7"| BIFAST["BIFAST"]
        Ch -->|"≤ Rp1M\nWorking Hours"| SKN["SKN/LLG"]
        Ch -->|"> Rp100jt\nWorking Hours"| RTGS["RTGS"]
    end

    OLT & BIFAST & SKN & RTGS --> Inquiry

    subgraph Inquiry["3. Inquiry Beneficiary"]
        InqAPI["POST ...account-inquiry-external\n→ Dapat referenceNo + nama rekening"]
    end

    Inquiry --> Submit

    subgraph Submit["4. Submit Transfer"]
        SubAPI["POST transfer endpoint\nSertakan referenceNo dari Inquiry\n+ originatorInfos (PPATK)\n+ field spesifik per channel"]
    end

    Submit --> CheckResp{Response?}

    CheckResp -->|"00 - SUCCESS"| Done(["✅ Transfer Sukses"])
    CheckResp -->|"01 - Initiated\natau Timeout"| StatusCheck

    subgraph StatusCheck["5. Transfer Status Check"]
        PollStatus["POST corporate/v1.0/transfer/status\nserviceCode: 18/22/23"]
        PollStatus --> FinalCheck{latestTransactionStatus?}
        FinalCheck -->|"00 SUCCESS"| Done
        FinalCheck -->|"Pending"| PollStatus
        FinalCheck -->|"> 24H"| Escalate["📞 Hubungi OCBC"]
    end

    style Auth fill:#fff3cd,stroke:#f0ad4e,color:#000
    style SelectChannel fill:#cce5ff,stroke:#004085,color:#000
    style Inquiry fill:#e2d9f3,stroke:#6f42c1,color:#000
    style Submit fill:#d4edda,stroke:#28a745,color:#000
    style StatusCheck fill:#f8d7da,stroke:#721c24,color:#000
    style Done fill:#28a745,color:#fff,stroke:#28a745
    style Start fill:#dc3545,color:#fff,stroke:#dc3545
    style Escalate fill:#fd7e14,color:#fff,stroke:#fd7e14
```

---

## REMITTER / BENEFICIARY CATEGORY CODE (Appendix 6.2)

| Code | Keterangan |
|---|---|
| A0 | Perorangan |
| B0 | Pemerintah |
| C0 | Bank Central |
| C1 | Bank Pelapor di Dalam Negeri |
| C9 | Bank Lainnya |
| D0 | Lembaga Keuangan Non-Bank |
| E0 | Perusahaan |
| F1 | Lembaga/Organisasi Internasional Berbentuk Bank |
| F2 | Lembaga/Organisasi Internasional Bukan Bank |
| I0 | Pelaku Identik |
| Z9 | Lainnya |

---

## KEY DECISIONS — Pertanyaan untuk Meeting

| Topik | Yang Perlu Diputuskan |
|---|---|
| **Token refresh strategy** | Background job atau lazy refresh? |
| **`partnerReferenceNo` format** | Convention yang disepakati tim UPG? |
| **`originatorInfos` source** | Data ultimate sender diambil dari mana di sistem UPG? |
| **Bank metadata attachment** | Kapan file attachment dari OCBC bisa diterima? |
| **Error escalation SLA** | Berapa lama sebelum kontak OCBC jika status stuck >24H? |
| **SKN regulatory fields** | `regulatorResidentStatus` hardcode atau dari data nasabah? |
| **Email notification** | `sendEmailNotification` selalu `false` atau configurable? |
| **RTGS/SKN working hours guard** | Validasi di UI atau backend? |
| **Transfer channel selection** | Logika pemilihan channel otomatis atau pilihan user? |

---

## TRANSACTION LIMIT SUMMARY

| Channel | Min | Max/Trx | Limit/Hari | Fee | Jam Operasi |
|---|---|---|---|---|---|
| Intrabank (Overbooking) | Rp1 | Unlimited | Unlimited | — | 24x7 |
| Online Transfer (OLT) | Rp10.000 | Rp100.000.000 | Rp500.000.000 | Rp6.500 | 24x7 |
| BIFAST | Rp1 | Rp250.000.000 | Unlimited | Rp2.500 | 24x7 |
| SKN/LLG | Rp1 | Rp1.000.000.000 | Unlimited | Rp2.000 | Working hours |
| RTGS | Rp100.000.001 | Unlimited | Unlimited | Rp25.000 | Working hours |

---

*Sumber: OCBC NISP Tech-Doc SNAP v1.10 (17 Jul 2025)*
*Reference document: "D:\reference document\2. Tech-Doc-OCBC NISP Standar Nasional Open API Pembayaran (SNAP) V1.10 (1).pdf"
*Ditulis otomatis via Claude Code — 2026-04-27*
