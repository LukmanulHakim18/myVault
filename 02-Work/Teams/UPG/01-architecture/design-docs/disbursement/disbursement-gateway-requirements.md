---
title: "Disbursement Gateway (UPG) — Requirements Brief"
project_name: "Disbursement Gateway (UPG)"
service_name: "disbursement-service"
tags:
  - upg
  - disbursement
  - ocbc
  - snap
  - broker
  - requirements
type: requirements
status: draft
owner: lukmanul.hakim
created: "2026-07-01"
updated: "2026-07-01"
source:
  - "[[../../../../../Meetings/2026-04-17-meeting-prep-ocbc-disbursement-upg]]"
  - "[[../../../../../Meetings/2026-04-27-OCBC-Disbursement-SNAP-API]]"
  - "[[../../UPG-Helicopter-View]]"
---

# Disbursement Gateway (UPG) — Requirements Brief

> **Project**: Disbursement Gateway (UPG) · **Service (deploy)**: `disbursement-service` · **Provider fase 1**: OCBC (modul internal; calon `ocbc-service` saat split multi-provider).

> **Status**: Requirements discovery (brainstorm). Dokumen ini **belum** desain arsitektur.
> **Next step**: `/sa:design` untuk arsitektur service & kontrak API, lalu RFC-UPG-00X.
> **Deliverable brainstorm**: goal terklarifikasi, functional/non-functional requirements, acceptance criteria, project checklist, open questions.

---

## 1. Goal

Membangun **layanan disbursement terpusat** di bawah tim **UPG** yang berperan sebagai **Gateway / Broker**: satu pintu keluar dana untuk **seluruh tim Bluebird** (MRG, reservation, dll), dengan **OCBC NISP** sebagai provider bank pertama melalui **SNAP BI**.

**Success criteria (verifiable):**
- Sebuah tim consumer bisa melakukan disbursement end-to-end (refund/payout) via UPG tanpa tahu detail SNAP OCBC.
- Transaksi idempoten, statusnya bisa ditelusuri sampai final, dan terekonsiliasi dengan mutasi OCBC.
- Menambah provider bank kedua **tidak** mengubah kontrak yang dipakai consumer.

---

## 2. Keputusan yang Sudah Dikonfirmasi

| Topik | Keputusan | Implikasi |
|---|---|---|
| **Kontrak internal** | gRPC internal **+** fasad SNAP | Consumer Bluebird pakai gRPC (konsisten stack UPG); fasad SNAP tersedia untuk konsumen yang butuh format SNAP. |
| **Provider scope** | **Multi-provider sejak awal** | Wajib abstraksi `DisbursementProvider`; OCBC = implementasi pertama; kontrak consumer harus provider-agnostic. |
| **Arah arsitektur** | **Opsi B — Broker-shaped, OCBC-only** (2026-07-02) | Interface provider + kontrak provider-agnostic, **implementasi hanya OCBC, TANPA routing multi-bank** dulu. Memenuhi PRD (Finance cepat) tanpa over-engineer. Bank ke-2 = tambah implementasi, kontrak tetap. |
| **Approval & limit** | **Auto (system-to-system)** | Tidak ada maker-checker manual. UPG hanya validasi **limit teknis** (per channel, per tenant). Tanggung jawab bisnis di consumer. |
| **Use cases** | **Fase 1 = disbursement tim Finance via API** (per PRD); refund MyBB = sekunder | Consumer pertama = **Finance/HO** (bukan MRG). Payout/vendor/payroll menyusul. Desain tetap generic terhadap tipe disbursement. |
| **Baseline existing** | Sudah ada disbursement OCBC via **host-to-host** | Ini **bukan greenfield murni** — API adalah channel baru; H2H existing tetap jalan. |
| **Approval** | **Di sisi client, bukan UPG** (MoM 30 Jun) | Whitelist s/d approval CFO via ITRF ITRFD-621 di client. Menguatkan keputusan auto-s2s. |
| **Notifikasi status** (OQ-2) | **PubSub + polling + webhook** (2026-07-02) | 1 topic event + subscription **filtered per `channel`** (isolasi); consumer bisa poll; webhook opsional untuk realtime tanpa akses PubSub. |
| **Sumber PPATK** (OQ-4) | **Config per-channel** | `originatorInfos` (ultimate sender) diambil dari registry channel, bukan per-request — tak membebani consumer. |
| **Posting SAP/HO** (B2) | **Tidak** — broker emit event saja | Broker fokus eksekusi + record + event; posting HO (mis. refund) = subscriber terpisah. Finance/SAP sudah punya record sendiri. |
| **Bentuk transfer** (OQ-6) | **Batch sejak fase 1** | 1 request bisa banyak transfer. Wajib: partial success/failure, status & idempotency **per-item**, retry granular. |
| **Cek saldo** (B3) | **Single: tidak per-transaksi · Batch: ya (pra-validasi)** | Single → andalkan response OCBC (gagal = tak ada uang keluar) + monitoring saldo & alert. **Batch → validasi total nominal batch vs saldo tersedia** di gate pra-eksekusi (#2a). Caveat: saldo bisa bergeser (H2H/batch paralel) → eksekusi tetap best-effort. |
| **Semantik batch** (#2) | **Validasi-semua-dulu (termasuk saldo) → eksekusi best-effort per-item; hasil per-item sync + final via event** (2026-07-02) | (#2a) Validasi seluruh item (format, entitlement, limit, **saldo total**); ada yang invalid → tolak seluruh batch sebelum kirim. Lalu eksekusi per-item independen. (#2b) Submit balikin accepted/rejected per-item + `batchId`; status FINAL per-item via event/polling/webhook. |
| **Idempotency** (#3) | **Scope per-channel + retensi panjang (ikut record)** (2026-07-02) | Key unik dalam namespace `channel` (batch: key level-batch + per-item). Disimpan selama record disbursement ada (store broker #1) → dedup absolut, aman untuk jalur uang. |
| **Auth consumer** (B5) | **Service token / API key per channel** | Tiap channel punya kredensial sendiri → perlu secret management + rotasi. |
| **Pemilihan channel** (B6) | **Client memilih bank + metode transfer, dibatasi entitlement per-channel** (2026-07-02) | Endpoint baru **`GetAllowedRoutes`**: kembalikan `allowedBanks[]` + `allowedTransferMethods[]` untuk channel. Registry channel menyimpan entitlement. **Tiap request divalidasi** terhadap entitlement (bank/metode di luar daftar → tolak). Kontrak consumer membawa `bank` + `transferMethod` eksplisit. *(Asumsi: bank = provider/bank sumber; metode = BIFAST/OLT/RTGS/SKN/Intrabank.)* |
| **Data ownership / SSOT** (#1) | **Broker punya store sendiri + emit event** (2026-07-02) | DB broker untuk operasional (idempotency, state machine, batch, retry). Emit event ke PubSub → reporting/rekonsiliasi/unified-view dibangun downstream. **Tidak menyentuh service `transaction` (P1)**. Proyeksi ke SSOT terpusat = fase lanjut (bila diperlukan). |

> **OQ-5 — Velocity/anti-abuse guard per channel:** ❌ **Out of scope fase 1 (2026-07-08)** — belum dibutuhkan; tidak ada hard cap/rate-limit per channel. Andalkan limit teknis OCBC + monitoring saldo. Bisa ditambah belakangan tanpa ubah kontrak.

---

## 3. Scope

### In Scope (Fase 1)
- Service baru di landscape UPG: **`disbursement-service`** (project: *Disbursement Gateway (UPG)*).
- Abstraksi provider (interface) + **OCBC** sebagai provider pertama.
- 5 transfer type OCBC: **Intrabank, BIFAST, OLT, RTGS, SKN** (mekanik di [[../../../../../Meetings/2026-04-27-OCBC-Disbursement-SNAP-API|catatan 27 Apr]] + [[ocbc-snap-disbursement-data-model|data model]]).
- **Fitur consumer-facing (per PRD)**: Balance Inquiry, History List, Transfer Account Inquiry, Transfer Submit, Transfer Status.
- **Entitlement per-channel (B6)**: endpoint `GetAllowedRoutes` (list bank + metode transfer yang boleh untuk channel) + **validasi tiap request** terhadap entitlement; consumer kirim `bank` + `transferMethod` eksplisit.
- Kontrak consumer: **gRPC** (utama) + **fasad SNAP REST**.
- **Batch disbursement** (1 request → banyak transfer) dengan status & idempotency **per-item**.
- Notifikasi: **event PubSub + polling + webhook** opsional.
- Auth consumer: **service token / API key per channel**.
- Idempotency, status tracking sampai final, reconciliation, callback handling.
- Validasi limit teknis per channel + monitoring saldo & alert.
- Multi-consumer: identifikasi consumer via `channel` + source account mapping.
- **Admin API internal** (registrasi client/provider/source_account/route + penerbitan token) — internal-only; fase-1 boleh seed manual DB.

### Out of Scope (Fase 1)
- Maker-checker / approval workflow manual (keputusan: auto s2s).
- **Posting ke SAP/HO** — broker hanya emit event; posting = subscriber terpisah (B2).
- UI/dashboard operasional (bisa reuse `web-gateway` / BBD nanti).
- Provider bank kedua konkret (abstraksi disiapkan, implementasi belakangan).
- FX / multi-currency (IDR only).
- **Velocity/anti-abuse guard** (hard cap harian / rate-limit per channel) — belum dibutuhkan fase 1 (OQ-5, 2026-07-08). *(Pemilihan channel B6 sudah RESOLVED: client pilih dari whitelist.)*

### Asumsi (perlu divalidasi)
- Source-of-fund adalah rekening korporat OCBC milik Bluebird. **Model: per-channel** (registry channel → rekening sumber). Fase 1 = 1 rekening (Finance).
- Consumer bertanggung jawab atas keabsahan bisnis transaksi (auto s2s), UPG bertanggung jawab atas eksekusi teknis & idempotency.
- Semua consumer sudah/akan berjalan di jaringan internal Bluebird (gRPC reachable).

---

## 4. Functional Requirements

**Broker / Consumer-facing**
- **FR-1** Consumer submit request disbursement dengan **memilih `bank` + `transferMethod` secara eksplisit** (dari yang di-whitelist untuk channel-nya) + amount, beneficiary (account+bank), reference internal, ultimate sender. **Kontrol pemilihan penuh di client**; broker **tidak** auto-pilih channel.
- **FR-2** Broker hanya pegang **whitelist per-channel** (`allowedBanks[]` + `allowedTransferMethods[]`, via `GetAllowedRoutes`) dan **memvalidasi** tiap request: (a) `bank`/`transferMethod` ada di whitelist channel; (b) memenuhi **limit teknis** method tsb (OLT ≤100jt 24x7, BIFAST ≤250jt 24x7, SKN ≤1M working-hours, RTGS >100jt working-hours). Di luar whitelist/limit → **tolak** dengan error jelas. *(Broker validasi, bukan seleksi channel.)*
- **FR-3** Broker mengembalikan **status non-final** (accepted/pending) segera + cara consumer mengetahui status final (query API dan/atau event).
- **FR-4** Consumer dapat query status transaksi by reference (mapping ke `latestTransactionStatus` OCBC → status kanonik broker).
- **FR-5** Broker meng-emit **event** perubahan status (SUCCESS/FAILED/CANCELLED) via PubSub agar consumer bereaksi async.
- **FR-6** Setiap request punya **idempotency key** dari consumer; retry aman tanpa dobel transfer.
- **FR-7** Broker melakukan **beneficiary inquiry** (nama rekening) dan mengembalikannya ke consumer sebelum/at submit (konfirmasi penerima).
- **FR-8** Fasad **SNAP REST** mengekspos operasi setara untuk konsumen yang berbicara SNAP.

**Provider / OCBC-facing**
- **FR-9** Abstraksi `DisbursementProvider` dengan operasi minimal: `Inquiry`, `Submit`, `Status`, `HandleCallback`. OCBC = implementasi pertama.
- **FR-10** Implementasi auth OCBC SNAP: token B2B (TTL 900s) + **auto-refresh**, Transaction Signature per request (HMAC_SHA512).
- **FR-11** Generator `partnerReferenceNo` (globally unique, prefix OCBC) & `X-EXTERNAL-ID` (unik per hari).
- **FR-12** Alur Inquiry→Submit menyimpan & meneruskan `referenceNo`/`bankRef` sesuai channel (BIFAST/OLT/RTGS/SKN).
- **FR-13** Isi `originatorInfos` (kepatuhan PPATK) dari data yang di-supply broker/consumer.
- **FR-14** Handle callback async OCBC + **validasi signature**; fallback ke Transfer Status polling saat timeout/pending.

**Ops / Data**
- **FR-15** Simpan setiap transaksi (state machine: CREATED→PENDING→SUCCESS/FAILED/CANCELLED) + audit trail request/response.
- **FR-16** Job **status reconciliation** (mirip pola `upg-scheduler`) untuk transaksi pending > threshold; eskalasi manual bila >24 jam.
- **FR-17** Validasi limit teknis per channel & per tenant sebelum submit.

**Admin / Registrasi (internal-only, 2026-07-09)**
- **FR-18** **Admin API internal** (`DisbursementAdmin`, ClusterIP, tak di-expose publik) untuk: daftar client (`RegisterClient`), registrasi provider & rekening sumber (`RegisterProvider`/`RegisterSourceAccount`), kelola entitlement route (`AddRoute`/`DisableRoute`), master rail (`AddTransferMethod`). Fase-1 boleh seed manual DB.
- **FR-19** **Penerbitan token** (`IssueToken`/`RotateToken`): generate token acak opaque per channel, simpan **`api_token_hash` = SHA-256(token)**, kembalikan plaintext **sekali**; rotasi = baru + invalidate lama.
- **FR-20** Entitlement **opt-in eksplisit**: route dibuat admin per (client, source_account, method); tak ada auto-allow-all. `DisableRoute` = soft-disable (audit).

---

## 5. Non-Functional Requirements

| Kategori | Requirement |
|---|---|
| **Idempotency** | Tidak boleh ada double-disbursement walau retry/timeout/duplicate call. Mapping `idempotencyKey ↔ partnerReferenceNo ↔ status` persist di DB. |
| **Konsistensi** | Uang keluar = state SUCCESS. Tidak boleh "SUCCESS di broker tapi tidak di OCBC" atau sebaliknya → wajib reconciliation. |
| **Availability** | Target P1 (setara service UPG lain). Graceful degradation saat OCBC down (queue/retry, bukan gagal diam). |
| **Security** | `privateKey`/`secretKey`/`clientSecret` di secret store (vault/env), bukan di repo. mTLS ke OCBC. Validasi signature callback. Least-privilege source account. |
| **Auditability** | Semua transaksi + perubahan status ter-log & bisa ditelusuri (siapa tenant, kapan, berapa, ref OCBC). Kepatuhan PPATK. |
| **Observability** | Metrik per channel/tenant (volume, success rate, latency, saldo), tracing (Elastic APM), alert saldo rendah & pending menumpuk. |
| **Extensibility** | Tambah provider bank baru tanpa ubah kontrak consumer (Open/Closed). |
| **Multi-tenant isolation** | Limit, source account, dan reporting terpisah per tenant. |
| **SLA transparan** | Broker menyampaikan ekspektasi finalitas per channel (BIFAST/OLT real-time; RTGS/SKN bisa 1–3 hari). |

---

## 6. User Stories & Acceptance Criteria

- **US-1 — Consumer refund**
  *Sebagai* tim MRG, *saya ingin* mengirim refund ke rekening customer via satu call ke UPG, *sehingga* saya tidak perlu tahu detail SNAP OCBC.
  **AC:** call gRPC dengan (`bank` + `transferMethod` dari whitelist, amount, rek tujuan, ref internal) → dapat status accepted + txnId; broker validasi terhadap whitelist channel (bukan pilih channel); menerima event final SUCCESS/FAILED.

- **US-2 — Idempotent retry**
  *Sebagai* consumer, *saya ingin* retry aman saat timeout.
  **AC:** dua call dengan idempotency key sama → hanya satu transfer OCBC; call kedua mengembalikan status transaksi pertama.

- **US-3 — Beneficiary confirmation**
  **AC:** sebelum submit, broker mengembalikan `beneficiaryAccountName` dari inquiry; nama kosong/akun invalid → error terklasifikasi, tanpa submit.

- **US-4 — Status finality**
  **AC:** transaksi pending di-poll otomatis; status kanonik broker konsisten dengan `latestTransactionStatus` OCBC; final state tidak berubah lagi.

- **US-5 — Tambah provider**
  **AC:** implementasi provider baru memenuhi interface `DisbursementProvider` tanpa mengubah proto/kontrak consumer.

- **US-6 — Limit guard**
  **AC:** transaksi di luar whitelist (bank/method) atau melebihi limit teknis method ditolak sebelum ke OCBC, dengan error jelas (menyebut whitelist/limit yang berlaku).

- **US-7 — Reconciliation**
  **AC:** job harian mencocokkan transaksi broker vs mutasi/status OCBC; selisih dilaporkan; pending >24 jam masuk daftar eskalasi.

---

## 7. Project Checklist

> Checklist untuk *mencapai* project. Fase 0 = prasyarat non-teknis; sisanya urutan eksekusi teknis. Detail mekanik OCBC (auth, signature, field mapping per channel) ada di [[../../../../../Meetings/2026-04-27-OCBC-Disbursement-SNAP-API|catatan 27 Apr]] — jangan diulang, cukup dirujuk saat implementasi.

### Fase 0 — Procurement & Alignment (non-teknis, blocking)
- [ ] Finalisasi daftar use case fase 1 dengan **product** (refund / payout / vendor / payroll — pilih yang di-lock).
- [ ] Konfirmasi **model source-of-fund** (rekening OCBC pooled vs per-tenant) → jawab Open Question OQ-1.
- [ ] Request kredensial OCBC: `clientID`, `clientSecret` via `API_Solutions@ocbc.id`.
- [ ] Generate RSA key pair, kirim `publicKey` ke OCBC, simpan `privateKey` aman.
- [ ] Minta attachment OCBC: `bankCityCode`, `bankAddress`, `networkClearingId` (wajib OLT/RTGS/SKN) + **4-digit prefix** `partnerReferenceNo`.
- [ ] Dapatkan sertifikat **mTLS** + akses sandbox (`developer.ocbc.id/snap/apis`) & production (`snapapi.ocbc.id`).
- [ ] Tentukan nama service & posisinya di [[../../UPG-Helicopter-View|helicopter view]] (domain baru: *Disbursement*).

### Fase 1 — Foundation (broker skeleton, provider-agnostic)
- [ ] Definisikan **status kanonik broker** + state machine (mapping ke kode OCBC 00/01/03/05/06/07).
- [ ] Definisikan interface `DisbursementProvider` (`Inquiry`, `Submit`, `Status`, `HandleCallback`).
- [ ] Skema data transaksi + tabel idempotency (`idempotencyKey ↔ partnerReferenceNo ↔ status`).
- [ ] Kontrak **gRPC** consumer-facing (provider-agnostic, tanpa bocor istilah OCBC).
- [ ] Modul validasi **limit teknis** per channel & per tenant.
- [ ] Registry tenant + mapping source account.

### Fase 2 — OCBC Provider Implementation
- [ ] Auth layer: token B2B + **auto-refresh** (<900s) + Transaction Signature (HMAC_SHA512).
- [ ] Generator `partnerReferenceNo` (unik global) & `X-EXTERNAL-ID` (unik/hari).
- [ ] Implementasi per channel: **BIFAST, OLT, RTGS, SKN** (Inquiry→Submit→Status) + field `originatorInfos` (PPATK).
- [ ] Validasi request terhadap **whitelist per-channel** (bank + transferMethod) + **limit teknis** method (amount + jam operasional) + guard working-hours RTGS/SKN. *(Broker validasi, bukan auto-pilih channel.)*
- [ ] Handler **callback** OCBC + validasi signature; fallback Transfer Status polling.

### Fase 3 — Broker Surface & Multi-Tenant
- [ ] Fasad **SNAP REST** (operasi setara gRPC) untuk konsumen SNAP.
- [ ] Event publishing perubahan status via **PubSub** (pola UPG existing).
- [ ] Query status API (by consumer reference / txnId).
- [ ] Error taxonomy terklasifikasi (retriable vs final vs needs-escalation).

### Fase 4 — Reliability & Reconciliation
- [ ] Retry policy: retry hanya pada PENDING/timeout (cek status dulu, **jangan** retry FAILED).
- [ ] Job status reconciliation (pola `upg-scheduler`): poll pending, eskalasi >24 jam.
- [ ] Rekonsiliasi saldo/mutasi OCBC vs record broker (History List / statement).
- [ ] Handling EOD window BIFAST/OLT (00.00–05.01) & finalitas RTGS/SKN 1–3 hari.

### Fase 5 — Security & Observability
- [ ] Secret management (`privateKey`/`secretKey`/`clientSecret` di vault) + rotasi.
- [ ] Metrik per channel/tenant + alert (saldo rendah, pending menumpuk, success rate turun).
- [ ] Tracing Elastic APM + audit log lengkap (PPATK-ready).

### Fase 6 — Testing & Go-Live
- [ ] Test semua channel di **sandbox** (success/pending/timeout/not-found).
- [ ] Verifikasi signature via Postman (`/v1.0/transaction-signature/b2b`).
- [ ] Selesaikan **ASPI Functional Scenario & Developer Site Testing** (prasyarat produksi).
- [ ] Uji idempotency & concurrency (double-submit, duplicate `X-EXTERNAL-ID`/`partnerReferenceNo`).
- [ ] Onboard 1 consumer pilot (mis. refund MRG) → validasi end-to-end → go-live bertahap.

---

## 8. Open Questions

- **OQ-1** ✅ **RESOLVED (2026-07-02)**: Source account **per-channel** (tiap channel punya rekening sumber di registry). Fase 1 praktis 1 rekening (Finance), tapi model per-channel untuk atribusi & rekonsiliasi bersih.
- **OQ-2** ✅ **RESOLVED (2026-07-02)**: PubSub (topic tunggal + subscription filtered per channel) + polling + webhook opsional.
- **OQ-3** ✅ **RESOLVED (PRD)**: Balance Inquiry & History List = **consumer-facing** (bukan internal-only).
- **OQ-4** ✅ **RESOLVED (2026-07-02)**: `originatorInfos` (ultimate sender PPATK) = **config per-channel**.
- **OQ-5** ❌ **DROPPED (2026-07-08)** — velocity/anti-abuse guard **out of scope fase 1**, belum dibutuhkan. Andalkan limit teknis OCBC + monitoring saldo.
- **OQ-6** ✅ **RESOLVED (2026-07-02)**: **Batch sejak fase 1** (partial failure & status per-item).
- **OQ-7** Kandidat provider bank kedua (untuk memvalidasi abstraksi) — sudah ada arah?
- **B6** ✅ **RESOLVED (2026-07-02)**: Client memilih bank + metode transfer, dibatasi **entitlement per-channel** (endpoint `GetAllowedRoutes` + validasi request).

---

## 9. Next Steps

1. ✅ OQ-1 & arah arsitektur (Opsi B) sudah diputuskan — **siap masuk desain**.
2. Jalankan Fase 0 (procurement OCBC) secara paralel — ini lead-time panjang.
3. `/sa:design` → arsitektur service, kontrak gRPC/SNAP, skema DB, state machine (Opsi B: OCBC-only, provider-agnostic).
4. Tulis **RFC-UPG-00X** (pola [[../../rfcs/RFC-UPG-001-ho-report-queue-service|RFC-UPG-001]]) untuk sign-off arsitek.

---

## 10. Changelog / Session Log

### 2026-07-01 — Discovery & Reference
- `/sa:brainstorm` → requirements brief dibuat (goal, scope, FR/NFR, user stories, checklist 6 fase).
- Keputusan awal: gRPC + fasad SNAP · multi-provider · auto s2s.
- Data model reference dibuat & diverifikasi field-by-field dari OCBC SNAP v1.10.
- Enrichment dari **PRD + MoM ClickUp**: consumer pertama = **Finance**, baseline = OCBC **host-to-host** (API = channel baru), approval di **client**, Balance/History = consumer-facing.
- Field kontrak: `channel` wajib, `tenant` opsional.

### 2026-07-02 — Keputusan Arsitektur & OQ Round
- **Arah arsitektur**: **Opsi B — Broker-shaped, OCBC-only** (provider-agnostic, tanpa routing multi-bank).
- **OQ-1** ✅ source account **per-channel**.
- **OQ round** (8 item): OQ-2 notif = PubSub+polling+webhook · OQ-4 PPATK = config per-channel · **B2** = tidak posting SAP/HO (emit event) · **OQ-6** = **batch fase 1** · **B3** = tanpa cek saldo per-trx (monitoring+alert) · **B5** auth = token/API key per channel.
- **TODO bisnis**: B6 (pemilihan channel) · OQ-5 (velocity guard) — opsi tersimpan di §2.
- **Perlu konfirmasi OCBC**: B1 (callback vs polling-only) · DATA-1 (attachment metadata bank).
- **Status**: fase Requirements/Analysis **SELESAI** → siap `/sa:design`.

### 2026-07-02 (lanjutan) — Reorg, Gap Analysis & Keputusan Pre-Design
- **Reorg**: dokumen dipindah ke folder `design-docs/disbursement/` (brief + data model), link relatif diperbaiki.
- **Gap analysis** (data model §10): OCBC SNAP **bukan plug-and-play** — tutup ~30% (primitif per-transaksi); ~70% engineering broker. Gap terbesar: batch, notifikasi async, multi-consumer, kontrak provider-agnostic + metadata.
- **Item konfirmasi OCBC** (data model §10): **CONF-1** endpoint bulk/batch? · **CONF-2** callback vs polling-only? · **CONF-3** kapan attachment metadata bank? — *user follow-up ke tim OCBC*.
- **B6** ✅ **RESOLVED**: client memilih **bank + transferMethod**, dibatasi **entitlement per-channel** → endpoint `GetAllowedRoutes` + validasi request. Field kontrak baru: `bank`, `transferMethod`.
- **Grey area pre-design** ✅: **#1 SSOT** = broker punya store sendiri + emit event (tidak sentuh `transaction` P1) · **#2 batch** = validasi-semua-dulu (termasuk saldo total) → best-effort per-item, hasil sync per-item + final via event · **#3 idempotency** = scope per-channel + retensi panjang (ikut record).
- **Refine B3**: single = tanpa pre-flight balance · batch = validasi total vs saldo di gate.
- **Masih TODO bisnis**: **OQ-5** velocity/anti-abuse guard (belum diputuskan).
- **Status**: semua blocker desain bersih → **siap `/sa:design`** (OQ-5 & CONF OCBC jalan paralel).

### 2026-07-08 — Konfirmasi OCBC & OQ-5 Ditutup
- **CONF-1** ✅ OCBC **tidak punya** batch endpoint → broker orkestrasi batch sendiri (loop per-item).
- **CONF-2 (B1)** ✅ **Polling-only** → worker poller wajib; tak ada callback OCBC.
- **CONF-3 (DATA-1)** ✅ Ketersediaan channel digerbang **whitelist 2 tingkat** (bank + transferMethod, via `GetAllowedRoutes`); ops whitelist method yang siap → **BIFAST + Intrabank dulu**. DATA-1 turun status: bukan blocker go-live, tapi prasyarat enablement OLT/RTGS/SKN.
- **OQ-5** ❌ **DROPPED** — velocity/anti-abuse guard out of scope fase 1 (belum dibutuhkan).
- **Status**: worst-case (single-only + polling-only) kini **fakta terkonfirmasi**; desain terkunci → siap RFC-UPG-00X + `/sa:implement`.

---

## Referensi
- [[disbursement-gateway-delivery-checklist|Delivery Checklist to Production]] — **checklist eksekusi M0→M10** (sign-off → go-live), 2 track (BIFAST+Intrabank dulu → OLT/RTGS/SKN setelah DATA-1)
- [[disbursement-gateway-design|Disbursement Gateway — Design]] — **arsitektur, kontrak gRPC/SNAP, skema DB, state machine** (framework: skeleton-api-go)
- [[ocbc-snap-disbursement-data-model|OCBC SNAP Disbursement — Data Model Reference]] — **acuan data untuk kontrak internal** (field per channel, sumber data, minimum request consumer)
- [[../../../../../Meetings/2026-04-17-meeting-prep-ocbc-disbursement-upg|Meeting Prep OCBC Disbursement (17 Apr)]]
- [[../../../../../Meetings/2026-04-27-OCBC-Disbursement-SNAP-API|OCBC Disbursement SNAP API — field mapping & flows (27 Apr)]]
- [[../../UPG-Helicopter-View|UPG Helicopter View]]
- OCBC NISP Tech-Doc SNAP v1.10 (17 Jul 2025)
- **PRD**: [SNAP API Disbursement OCBC NISP](https://bluebirdgroup.clickup.com/9018711461/v/dc/8crx7d5-412698/8crx7d5-134498) (ClickUp) — target user Finance, timeline Jan 2026
- **MoM**: [Discuss SNAP OCBC NISP](https://bluebirdgroup.clickup.com/9018711461/v/dc/8crx7d5-442318/8crx7d5-210578) — approval di client, assessment SAP di UPG+Architect
- **ITRF**: ITRFD-621 (whitelist s/d approval CFO, sisi client) · **PARR**: 9 Des 2025

---
_Requirements brief — dihasilkan via `/sa:brainstorm` 2026-07-01. Belum desain arsitektur._
