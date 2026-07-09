---
title: "OCBC SNAP Disbursement — Data Model Reference (Acuan Kontrak Internal)"
tags:
  - upg
  - disbursement
  - ocbc
  - snap
  - data-model
  - reference
  - contract
type: reference
status: active
owner: lukmanul.hakim
created: "2026-07-01"
updated: "2026-07-01"
source: "OCBC NISP Tech-Doc SNAP v1.10 (17 Jul 2025) — /mnt/d/project-bb/reference document/2. Tech-Doc-OCBC NISP ... (SNAP) V1.10.pdf"
related:
  - "[[disbursement-gateway-requirements]]"
  - "[[../../../../../Meetings/2026-04-27-OCBC-Disbursement-SNAP-API]]"
---

# OCBC SNAP Disbursement — Data Model Reference

> **Tujuan**: Menjadi **acuan data** untuk mendesain **kontrak internal** UPG (gRPC + fasad SNAP). Field yang diminta OCBC di hilir menentukan data apa yang **wajib dikumpulkan broker dari consumer** di hulu. Dokumen ini men-*derive* itu.
> **Sumber**: OCBC NISP SNAP Tech-Doc **v1.10** (17 Jul 2025), diverifikasi field-by-field langsung dari PDF (§5.2–§5.10).
> **Cara baca**: Bagian **§4 Unified Internal Contract Fields** adalah inti — di situ tiap field diberi label **sumber data** (consumer / broker-derived / consumer-config / OCBC-metadata / from-inquiry). Itu yang jadi dasar proto/kontrak kita.

---

## 1. Endpoint Inventory (13 endpoint disbursement)

| # | Method | Endpoint | Fungsi | Kategori |
|---|--------|----------|--------|----------|
| 1 | POST | `v1.0/authentication-signature/b2b` | Token Signature | Auth |
| 2 | POST | `v2.0/access-token/b2b` | Access Token (TTL 900s) | Auth |
| 3 | POST | `v1.0/transaction-signature/b2b` | Transaction Signature (Postman only) | Auth |
| 4 | POST | `corporate/v1.0/balance-inquiry` | Balance Inquiry | Query |
| 5 | POST | `corporate/v1.0/transaction-history-list` | History List | Query |
| 6 | POST | `corporate/v1.0/account-inquiry-internal` | Internal (Intrabank) Inquiry | Intrabank |
| 7 | POST | `corporate/v1.0/transfer-intrabank` | Intrabank Submit (Overbooking) | Intrabank |
| 8 | POST | `corporate/bifast/v1.0/account-inquiry-external` | BIFAST Inquiry | BIFAST |
| 9 | POST | `corporate/bifast/v1.0/transfer-interbank` | BIFAST Submit | BIFAST |
| 10 | POST | `corporate/olt/v1.0/account-inquiry-external` | Online Transfer (OLT) Inquiry | OLT |
| 11 | POST | `corporate/olt/v1.0/transfer-interbank` | OLT Submit | OLT |
| 12 | POST | `corporate/rtgs/v1.0/account-inquiry-external` | RTGS Inquiry | RTGS |
| 13 | POST | `corporate/v1.0/transfer-rtgs` | RTGS Submit | RTGS |
| 14 | POST | `corporate/skn/v1.0/account-inquiry-external` | SKN Inquiry | SKN |
| 15 | POST | `corporate/v1.0/transfer-skn` | SKN Submit | SKN |
| 16 | POST | `corporate/v1.0/transfer/status` | Transfer Status | Status |

> Pola universal semua channel: **Inquiry → simpan `referenceNo` → Submit (sertakan `referenceNo`) → (jika non-final) Transfer Status**.
>
> **Per PRD**: consumer-facing feature = **Balance Inquiry (#4), History List (#5), Transfer Account Inquiry, Transfer Submit, Transfer Status**. Jadi #4 & #5 **masuk kontrak consumer** (Client bisa cek saldo & histori), bukan internal-only.

---

## 2. Auth & Signature — Data yang Dibutuhkan

Detail mekanik (StringToSign, HMAC) ada di [[../../../../../Meetings/2026-04-27-OCBC-Disbursement-SNAP-API|catatan 27 Apr §1.1]]. Data yang harus **UPG simpan/kelola** (bukan dari consumer):

| Data | Sumber | Keterangan |
|---|---|---|
| `clientID` / `X-PARTNER-ID` | OCBC (setup) | ID partner |
| `clientSecret` | OCBC (setup) | Key HMAC_SHA512 (Transaction Signature) |
| `privateKey` (RSA) | UPG generate | Key SHA256withRSA (Token Signature) |
| `publicKey` | kirim ke OCBC | — |
| `accessToken` | derived per session | TTL 900s, auto-refresh |
| 4-digit prefix `partnerReferenceNo` | OCBC | Prefix wajib |

### Common Request Header (SEMUA transfer API)
| Header | Attr | Sumber | Nilai |
|---|---|---|---|
| `Content-Type` | M | fixed | `application/json` |
| `Authorization` | M | broker | `Bearer {accessToken}` |
| `X-PARTNER-ID` | M | config | = clientID |
| `X-SIGNATURE` | M | broker-derived | Transaction Signature per request |
| `X-TIMESTAMP` | M | broker-derived | ISO 8601 `yyyy-MM-ddTHH:mm:ss.SSSTZD` |
| `X-EXTERNAL-ID` | M | broker-derived | Numeric, **unik per hari** |
| `CHANNEL-ID` | M | fixed | `API` |

---

## 3. Legenda Sumber Data (label kolom "Sumber")

| Label | Arti | Implikasi kontrak internal |
|---|---|---|
| 🟦 **consumer** | Harus dikirim tim consumer per-request | Masuk request message kontrak internal |
| 🟩 **broker-derived** | Dibuat/dihitung broker | TIDAK di kontrak consumer (partnerReferenceNo, signature, X-EXTERNAL-ID) |
| 🟨 **consumer-config** | Konfigurasi per consumer (di-key oleh `channel`) — source account, sender, dll | Di registry consumer/channel, bukan per-request |
| 🟧 **OCBC-metadata** | Metadata bank statis dari attachment OCBC | Lookup table di broker (by bankCode) |
| 🟥 **from-inquiry** | Hasil Inquiry, dipakai di Submit | Internal broker (Inquiry→Submit chaining) |
| ⬜ **compliance** | Data PPATK (ultimate sender) | Dari consumer atau consumer-config, wajib regulasi |

---

## 4. Unified Internal Contract Fields (INTI)

Superset field dari semua channel, dikelompokkan. Kolom **Channel** = channel yang mewajibkan (I=Intrabank, B=BIFAST, O=OLT, R=RTGS, S=SKN). Kolom **Sumber** menentukan apakah field masuk kontrak consumer.

### 4.0 Consumer Identity & Routing (internal-only — TIDAK dikirim apa adanya ke OCBC)
> **Wajib**: broker harus tahu **consumer mana** yang melakukan disbursement — untuk audit ("dari mana dana keluar"), resolusi config/source account, isolasi multi-tenant, dan limit/policy per-consumer.

| Field (kontrak internal) | Wajib | Sumber | Catatan |
|---|---|---|---|
| `channel` | **M** | 🟦 consumer | **Identitas consumer/sub-divisi asal** disbursement (mis. `mrg-refund`, `reservation-payout`, `bbwallet-topup`). **Kunci utama** untuk resolve source account & config, audit, policy & limit per-consumer. Sudah cukup granular (level sub-divisi). ⚠️ **Beda dari OCBC `channel`** (device type) — lihat §4.4. |
| `tenant` | O | 🟦 consumer | **Opsional** — grouping/reporting (business unit di atas beberapa `channel`). Config di-resolve dari `channel`, bukan dari `tenant`. |
| `idempotencyKey` | **M** | 🟦 consumer | Anti double-disburse |
| `source_account_no` | **M** | 🟦 consumer | Rekening sumber (debit) yang dipilih client dari `GetAllowedRoutes`. **Provider/bank + api_code di-derive** dari rekening ini. **Divalidasi terhadap route channel** (B6). |
| `transferMethod` | **M** | 🟦 consumer | Rail transfer: `INTRABANK`/`BIFAST`/`OLT`/`RTGS`/`SKN`. **Divalidasi** — pasangan (`source_account_no`,`transferMethod`) harus ada di route (B6). Broker map ke endpoint OCBC. |

> **Entitlement (B6)**: broker sediakan endpoint `GetAllowedRoutes` → `{ routes: [{bank_name, bank_code, account_no, label, allowed_transfer_methods[]}] }` per channel. **Nested per REKENING sumber** (`source_account`) — tiap rekening bisa dukung rail berbeda. Client memilih **`source_account_no` + `transferMethod`**; request **ditolak** bila pasangan (`source_account_no`,`transferMethod`) di luar route channel. Entitlement dinormalisasi ke 4 tabel (provider/source_account/transfer_method/route) — lihat design §4.

> **Catatan tabrakan nama**: OCBC punya `additionalInfo.channel` = *device type* (opsional, mis. `mobilephone`). Di **kontrak internal kita**, `channel` bermakna *identitas consumer/sub-divisi*. Broker yang memetakan: `channel` (consumer) tercatat di audit log & tidak dikirim sebagai `additionalInfo.channel` OCBC; `additionalInfo.channel` OCBC di-default/di-set broker terpisah.
>
> **Source account & config di-resolve dari `channel`** (registry per-channel). `tenant` hanya lapisan grouping opsional di atasnya — tidak dipakai untuk resolusi teknis.

### 4.1 Beneficiary (penerima)
| Field OCBC | Wajib di channel | Sumber | Catatan untuk kontrak internal |
|---|---|---|---|
| `beneficiaryAccountNo` | I B O R S | 🟦 consumer | No rekening tujuan (len 34) |
| `beneficiaryBankCode` | B O R S | 🟦 consumer | BIC (BIFAST) / numerik (OLT/RTGS/SKN). Intrabank tidak perlu. |
| `beneficiaryAccountName` | B O R S (submit) | 🟥 from-inquiry | **Diisi dari hasil Inquiry**, bukan diketik consumer → validasi nama |
| `beneficiaryEmail` | C (jika notif) | 🟦 consumer | Wajib jika `sendEmailNotification=true` |
| `beneficiaryAddress` | O (B/O/S) | 🟦 consumer | Opsional |
| `beneficiaryCustomerResidence` | R S | 🟦 consumer | `1`=Indonesia `2`=Non |
| `beneficiaryCustomerType` | R S | 🟦 consumer | `1`=Individu `2`=Korp `3`=Govt |
| `beneficiaryAccountType` | (R resp) | 🟩 broker-derived | Default `D` (current account) |
| `receiverPhone` | O (R/S) | 🟦 consumer | Opsional |

### 4.2 Amount
| Field | Wajib | Sumber | Catatan |
|---|---|---|---|
| `amount.value` | semua | 🟦 consumer | String `16,2`, IDR = 2 desimal |
| `amount.currency` | semua | 🟩 broker-derived | `IDR` (fixed fase 1) |
| `currency` | semua | 🟩 broker-derived | `IDR` |

### 4.3 Source / Sender (pengirim)
| Field | Wajib | Sumber | Catatan |
|---|---|---|---|
| `sourceAccountNo` / `debitAccountNo` | semua | 🟨 consumer-config | Rekening debit UPG (per tenant / pooled → **OQ-1**) |
| `senderName` | R S | 🟨 consumer-config | Nama pengirim |
| `senderCustomerResidence` | O (R/S) | 🟨 consumer-config | `1`/`2` |
| `senderCustomerType` | O (R/S) | 🟨 consumer-config | `1`/`2`/`3` |
| `senderPhone` | O (R/S) | 🟨 consumer-config | — |
| `kodepos` | O (R/S) | 🟨 consumer-config | Kodepos pengirim |

### 4.4 Transaction Meta
| Field | Wajib | Sumber | Catatan |
|---|---|---|---|
| `partnerReferenceNo` | semua | 🟩 broker-derived | `{prefix}{unik-global}`. **Jangan** dari consumer |
| `transactionDate` | semua | 🟩 broker-derived | ISO 8601, bisa future date |
| `customerReference` | O | 🟦 consumer | Ref internal consumer (len 16) — untuk rekonsiliasi consumer |
| `remark` / `remarks` | O | 🟦 consumer | Deskripsi |
| `feeType` | O (O/R/S) | 🟨 consumer-config | `OUR`(default)/`BEN`/`SHA` |
| `trxPurpose` (inquiry) / `transactionPurpose` (submit) | B O | 🟦 consumer | `01`Investasi `02`Wealth `03`Purchase `99`Others |
| `sendEmailNotification` | M (B/O/R/S) | 🟨 consumer-config | Boolean |
| `deviceId`, `channel` (OCBC device type) | O | 🟩 broker-derived | Konteks device OCBC (mis. `mobilephone`). ⚠️ **Bukan** `channel` identitas consumer (§4.0) — di-set broker, default. |

### 4.5 Compliance — `originatorInfos[]` (Ultimate Sender, PPATK)
> **Conditional-Mandatory** semua transfer. OCBC menyarankan **selalu diisi** (Pasal 8 ayat 5 UU 3/2011). Untuk auto-s2s, sumber data ini **wajib disepakati** (→ OQ-4 di brief).

| Field | Wajib | Sumber | Catatan |
|---|---|---|---|
| `originatorCustomerNo` | M | ⬜ compliance (consumer/tenant) | No rekening ultimate sender (34) |
| `originatorCustomerName` | M | ⬜ compliance (consumer/tenant) | Nama ultimate sender (100) |
| `originatorBankCode` | M | ⬜ compliance (consumer-config) | Bank code ultimate sender (8) |

### 4.6 OCBC Bank Metadata (statis — lookup, BUKAN dari consumer)
> Dikirim OCBC sebagai attachment; broker simpan **lookup table by `beneficiaryBankCode`**. Consumer tidak perlu tahu ini.

| Field | Wajib | Channel |
|---|---|---|
| `beneficiaryBankCityCode` / `beneficiaryCityCode` | M | O R S |
| `beneficiaryBankAddress` / `beneficiarybankAddress` | M | O R S |
| `beneficiaryBankBranch` / `beneficiaryBankBranchName` | M | R S |
| `beneficiaryNetworkClearingId` / `beneficiaryBankNetworkClearingId` | M | O R S |

### 4.7 Regulatory & Inquiry-chaining (RTGS/SKN khusus)
| Field | Wajib | Sumber | Catatan |
|---|---|---|---|
| `referenceNo` (di `additionalInfo`) | B O R S (submit) | 🟥 from-inquiry | Dari response Inquiry |
| `bankRef` | R (submit) | 🟥 from-inquiry | RTGS: dari Inquiry `referenceNo` |
| `beneCategory` / `regulatorBeneCategory` | R S | 🟦 consumer / 🟨 config | Appendix 6.2 (A0/E0/…) |
| `remitterCategory` / `regulatorRemitterCategory` | R S | 🟨 consumer-config | Appendix 6.2 |
| `regulatorResidentStatus` | S | 🟦 consumer / 🟨 config | `Y`=Resident `N`=Non |
| `validationCheck` | S | 🟩 broker-derived | Default `N` |

---

## 5. Matriks Field Wajib per Channel (ringkas)

| Grup data | Intrabank | BIFAST | OLT | RTGS | SKN |
|---|:--:|:--:|:--:|:--:|:--:|
| Beneficiary acct+bank | ✅ | ✅ | ✅ | ✅ | ✅ |
| Beneficiary residence/type | — | — | — | ✅ | ✅ |
| trxPurpose | — | ✅ | ✅¹ | — | — |
| originatorInfos (PPATK) | ✅ | ✅ | ✅ | ✅ | ✅ |
| Bank metadata (city/addr/clearing) | — | — | ✅ | ✅ | ✅ |
| Regulatory category codes | — | — | — | ✅ | ✅ |
| senderName + sender detail | — | — | — | ✅ | ✅ |
| bankRef (from inquiry) | — | — | — | ✅ | ✅² |
| Working-hours guard | — | — | — | ✅ | ✅ |

<small>¹ OLT submit pakai `transactionPurpose`. ² SKN pakai `referenceNo` dari inquiry.</small>

**Kesimpulan desain kontrak internal**: kontrak consumer cukup minta **`channel` (wajib), idempotencyKey, beneficiary (acct+bank), amount, purpose, customerReference, identitas ultimate sender** (+ `tenant` opsional untuk grouping). Sisanya (source account, metadata bank, category codes, sender detail, referenceNo, signature, partnerReferenceNo, device channel OCBC) **di-resolve broker** dari registry `channel`/lookup/inquiry. Inilah yang membuat consumer tidak perlu tahu SNAP OCBC — sekaligus broker selalu tahu **dari consumer/channel mana** setiap disbursement berasal (audit & policy).

---

## 6. Status & Response Code (kanonik)

### `transactionStatus` / `latestTransactionStatus` (dari §5.8 & Transfer Status)
| Code | Status | Final? | Mapping broker |
|---|---|---|---|
| `00` | Success | ✅ | SUCCESS |
| `01` | Initiated | ❌ | PENDING (retry poll) |
| `02` | Paying/Suspect | ❌ | PENDING |
| `03` | Pending | ❌ | PENDING (poll max 15 mnt) |
| `04` | Refunded | ✅ | REFUNDED |
| `05` | Cancelled | ✅ | CANCELLED |
| `06` | Failed | ✅ | FAILED |
| `07` | Not Found | — | INVESTIGATE (cek History; >24H → kontak OCBC) |

### Response Code pattern
Format `{HTTP}{ServiceCode}{Case}` — mis. `2001800` (200-18-00 BIFAST OK), `4091801` (409-18-01 Duplicate partnerReferenceNo), `4091600` (409-16-00 Duplicate X-EXTERNAL-ID). **Service codes**: `17`=Intrabank, `18`=BIFAST/OLT, `22`=RTGS, `23`=SKN, `16`=Inquiry, `36`=Transfer Status.

**Idempotency signal penting:**
- `409...01` **Duplicate partnerReferenceNo** → submit sebelumnya sukses; **cek status, jangan retry**.
- `409...00` **Duplicate X-EXTERNAL-ID same day** → generate X-EXTERNAL-ID baru, aman retry.

---

## 7. Limit Transaksi (§1.5)

| Channel | Min | Max/Trx | Limit/Hari | Fee | Jam |
|---|---|---|---|---|---|
| Intrabank (Overbooking) | Rp1 | Unlimited | Unlimited | — | 24x7 |
| Online Transfer (OLT) | Rp10.000 | Rp100jt | Rp500jt | Rp6.500 | 24x7 |
| BIFAST | Rp1 | Rp250jt | Unlimited | Rp2.500 | 24x7 |
| SKN/LLG | Rp1 | Rp1M | Unlimited | Rp2.000 | Working hours |
| RTGS | Rp100.000.001 | Unlimited | Unlimited | Rp25.000 | Working hours |

Dipakai broker untuk **validasi limit teknis** method yang dipilih client (auto s2s; UPG validasi limit teknis + whitelist saja — **tidak** auto-pilih channel).

---

## 8. Contoh Minimum Request dari Consumer (target kontrak internal)

Konseptual — bukan proto final (itu di `/sa:design`). Menunjukkan bahwa consumer hanya kirim data bisnis, broker melengkapi sisanya:

```jsonc
// Consumer → UPG Disbursement Broker (provider-agnostic)
{
  "channel": "mrg-refund",               // 🟦 M → consumer/sub-divisi asal; kunci resolve source account, config, audit, limit
  "tenant": "mrg",                       // 🟦 O → grouping/reporting saja (opsional)
  "idempotencyKey": "mrg-refund-abc123", // 🟦 M anti double-disburse
  "source_account_no": "0123456789",     // 🟦 M → rekening sumber dari GetAllowedRoutes; provider/bank di-derive (B6)
  "transferMethod": "BIFAST",            // 🟦 M → pasangan (source_account_no,method) divalidasi vs route; broker map ke endpoint OCBC
  "beneficiary": {
    "accountNo": "722021990006",         // 🟦
    "bankCode": "014"                    // 🟦 → broker lookup metadata bank
  },
  "amount": "150000.00",                 // 🟦
  "purpose": "03",                       // 🟦 (opsional, default per use case)
  "customerReference": "ORDER-123",      // 🟦 rekonsiliasi consumer
  "ultimateSender": {                    // ⬜ PPATK (bisa dari consumer-config)
    "accountNo": "1234567886",
    "name": "Bluebird Group",
    "bankCode": "028"
  }
}
```
Broker meng-*derive*: `partnerReferenceNo`, `X-EXTERNAL-ID`, signature, `transactionDate`, metadata bank, category codes, `referenceNo` (via inquiry), sender detail. **Channel/rail dipilih client** (B6, divalidasi entitlement) — bukan auto-pilih broker.

---

## 9. Data Gaps / Open Questions (blocking desain kontrak)

- **OQ-1** ✅ **RESOLVED**: `sourceAccountNo` = **per-channel** (di registry channel). Fase 1 = 1 rekening (Finance).
- **OQ-4 (brief)** `originatorInfos` (ultimate sender): dari consumer per-request atau consumer-config (per-channel)? Untuk auto-s2s cenderung consumer-config.
- **DATA-1** ✅ **(2026-07-08)** Attachment OCBC (bankCityCode, bankAddress, networkClearingId, category codes) — **bukan blocker go-live**: ketersediaan method digerbang whitelist per-channel (CONF-3). Fase 1 whitelist **BIFAST + Intrabank** (tak butuh metadata); OLT/RTGS/SKN di-whitelist setelah metadata turun (metadata tetap prasyarat enablement method tsb).
- **DATA-2** `beneCategory`/`remitterCategory` (Appendix 6.2) — apakah consumer tahu kategori penerima, atau broker default? (mis. payout individu → `A0`).
- **DATA-3** `trxPurpose`/`transactionPurpose` — default per use case (refund=`03`? payout=`99`?) perlu disepakati product.

> **Konfirmasi PRD (SNAP API Disbursement OCBC NISP)**: PRD menandai DATA-1..3 yang sama sebagai *"need to confirm to OCBC"* — jadi gap ini diakui product, bukan asumsi kita. Consumer pertama = **tim Finance** (channel `finance`/`ho`); refund MyBB sekunder. Approval ada di **client**, bukan UPG (MoM). Sudah ada disbursement OCBC **host-to-host** existing — API = channel baru.

---

## 10. SNAP Coverage vs Broker Scope (Gap Analysis)

> **Pertanyaan kunci**: apakah OCBC SNAP sudah support semua use case kita, tinggal plug-and-play?
> **Jawaban**: **Tidak.** SNAP menyediakan **operasi primitif per-transaksi**; sebagian besar nilai tambah broker **dibangun di atasnya**. OCBC = **1 provider di belakang broker**, bukan produk jadi.

**Legenda**: ✅ native SNAP · ⚠️ parsial (broker melengkapi) · 🔴 broker bangun penuh (SNAP tak menyediakan)

| Kebutuhan | SNAP | Broker harus bangun |
|---|:--:|---|
| Transfer 5 channel (submit) | ✅ | Wiring + signature |
| Balance Inquiry / History List | ✅ | Pass-through |
| Account Inquiry (cek nama) | ✅ | Chaining `referenceNo`→submit |
| Transfer Status | ✅ | — |
| **Batch disbursement** (fase 1) | 🔴 | SNAP **single per call** — broker orkestrasi: loop N call, status & idempotency **per-item**, partial-failure |
| **Notifikasi async** (event/webhook) | 🔴 | v1.10 **polling-only**, tak ada callback OCBC — broker scheduler poll + emit event/webhook |
| **Multi-consumer / channel isolation** | 🔴 | OCBC lihat **1 partner** — registry channel, auth token, source account & PPATK per channel = 100% broker |
| **Kontrak provider-agnostic** | 🔴 | Translasi kontrak internal ↔ istilah OCBC |
| **Metadata bank** (OLT/RTGS/SKN) | 🔴 | Via attachment (DATA-1), bukan API — broker lookup by bankCode |
| **Idempotency dari consumer** | ⚠️ | Broker map `idempotencyKey`→`partnerReferenceNo` |
| **Category codes** RTGS/SKN | ⚠️ | Mapping/default (DATA-2) |
| **Working-hours guard** RTGS/SKN | 🔴 | Broker validasi jam operasional |
| **Channel selection** (B6) | 🔴 | **Client pilih bank+metode, dibatasi entitlement per-channel**; broker sediakan `GetAllowedRoutes` + validasi request (RESOLVED 2026-07-02) |
| **Velocity/anti-abuse** (OQ-5) | — | ❌ **Out of scope fase 1** (2026-07-08) — belum dibutuhkan; andalkan limit teknis OCBC + monitoring saldo. |
| **Refund** (MyBB) | ⚠️ | = transfer biasa ke rek customer; linking ke transaksi asal = broker/consumer |

**Estimasi kasar**: SNAP menutup **~30%** (primitif per-transaksi); **~70%** = engineering broker. Empat gap terbesar (paling jauh dari PNP): **(1) Batch**, **(2) Notifikasi async**, **(3) Multi-consumer**, **(4) Kontrak provider-agnostic + metadata**.

### ✅ Terkonfirmasi (2026-07-08) — worst-case, broker tanggung penuh
- **CONF-1** ✅ OCBC **TIDAK punya** endpoint bulk/batch → broker **orkestrasi batch sendiri** (loop per-item, idempotency & status per-item). Gap Batch tetap penuh.
- **CONF-2 (B1)** ✅ **Polling-only** — tak ada callback/webhook OCBC → broker **wajib scheduler poll** (worker); webhook ke consumer di-generate broker sendiri.
- **CONF-3 (DATA-1)** ✅ Ketersediaan channel digerbang **whitelist per-channel 2 tingkat** (bank + transferMethod, via `GetAllowedRoutes`) — client memilih dari yang di-whitelist. Ops hanya whitelist method yang metadata-nya siap → **BIFAST + Intrabank dulu**, OLT/RTGS/SKN di-whitelist saat metadata turun. **DATA-1 turun status: bukan blocker go-live, tapi prasyarat enablement per-method** (metadata tetap wajib sebelum method tsb di-whitelist).

> Status: **terkonfirmasi 2026-07-08**. Asumsi desain kini fakta: **single-only + polling-only**.

---

## Referensi
- OCBC NISP Tech-Doc SNAP **v1.10** — §5.2–§5.10 (field bodies), §1.5 (limit), §6 (appendix)
- [[../../../../../Meetings/2026-04-27-OCBC-Disbursement-SNAP-API|Field mapping & flows (27 Apr)]] — signature mechanics, sequence diagrams
- [[disbursement-gateway-requirements|Disbursement Gateway — Requirements Brief]]

---
_Data model reference — diverifikasi dari PDF sumber v1.10, 2026-07-01. Gap analysis (§10) + item konfirmasi OCBC ditambah 2026-07-02. Untuk desain kontrak, lanjut `/sa:design`._
