---
title: "Disbursement Gateway (UPG) — Delivery Checklist to Production"
project_name: "Disbursement Gateway (UPG)"
service_name: "disbursement-service"
tags:
  - upg
  - disbursement
  - ocbc
  - snap
  - checklist
  - delivery
  - go-live
type: checklist
status: active
owner: lukmanul.hakim
created: "2026-07-08"
updated: "2026-07-09"
related:
  - "[[disbursement-gateway-requirements]]"
  - "[[disbursement-gateway-design]]"
  - "[[ocbc-snap-disbursement-data-model]]"
---

# Disbursement Gateway (UPG) — Delivery Checklist to Production

> **Tujuan**: satu checklist eksekusi dari **sign-off → live production**. Diturunkan dari [[disbursement-gateway-design|Design]] + [[disbursement-gateway-requirements|Requirements]], mencerminkan keputusan terkunci per **2026-07-08**.
>
> **Prinsip yang dikunci:**
> - **Opsi B** — broker-shaped, OCBC-only, provider-agnostic.
> - **Client kontrol penuh** pilih `bank`+`transferMethod`; broker hanya **whitelist per-channel + validasi** (bukan auto-pilih).
> - **CONF-1** OCBC tak ada batch endpoint → broker orkestrasi batch. **CONF-2** polling-only → worker poller wajib. **CONF-3** ketersediaan method via whitelist.
> - **OQ-5 velocity guard DROPPED** (out of scope fase 1).
>
> **2 track eksekusi:**
> - 🟢 **Track A (jalur cepat go-live)**: BIFAST + Intrabank — **tidak** butuh bank-metadata (DATA-1).
> - 🟡 **Track B (menyusul)**: OLT / RTGS / SKN — **blocked** sampai DATA-1 (attachment metadata bank) turun.

**Legenda**: 🔴 blocking · ⏳ long lead-time · 🟢 Track A · 🟡 Track B · ⭐ prasyarat produksi

> **Status implementasi kode (2026-07-09):** repo `git.bluebird.id/upg/disbursementservice`, 4 commit lokal (belum push — nunggu default branch GitLab). **12/16 usecase GREEN via TDD** (3 read-only + 8 Admin API + `Disburse`). Sisa panic: `DisburseBatch`, `InquiryBeneficiary`, `BalanceInquiry`, `HistoryList`. **OCBC adapter nyata (HTTP/signature/mTLS) belum ada** — semua test lulus via mock; M3 baru "orkestrasi tervalidasi", bukan "OCBC benar-benar bisa dihubungi".

---

## M0 — Sign-off & Alignment

- [ ] **RFC-UPG-00X** ditulis (pola [[../../rfcs/RFC-UPG-001-ho-report-queue-service|RFC-UPG-001]]) 🔴
- [ ] RFC direview & **di-approve arsitek** 🔴
- [ ] Lock daftar use case fase 1 dengan **product** (konfirmasi: Finance = consumer #1; refund MyBB sekunder)
- [ ] Assessment teknis + kontrak **SAP/HO** (Eko Nugroho + Lukmanul) — output: apakah broker cukup emit event (B2)
- [ ] Sepakati default **DATA-2** (beneCategory/remitterCategory) & **DATA-3** (trxPurpose per use case) dengan product

---

## M1 — Fase 0 Procurement & Prasyarat (OCBC + Infra) ⏳

> Non-teknis, **lead-time panjang** — **mulai paralel secepatnya**, tidak menunggu RFC approve.

### OCBC credentials & security material
- [ ] Request `clientID` + `clientSecret` via `API_Solutions@ocbc.id` 🔴⏳
- [ ] Generate **RSA key pair**; kirim `publicKey` ke OCBC; simpan `privateKey` di vault 🔴
- [ ] Dapatkan **4-digit prefix** `partnerReferenceNo` dari OCBC 🔴
- [ ] Sertifikat **mTLS** + akses **sandbox** (`developer.ocbc.id/snap/apis`) & **production** (`snapapi.ocbc.id`) 🔴⏳
- [ ] Request **attachment metadata bank** (bankCityCode, bankAddress, networkClearingId, category codes) — **DATA-1** 🟡⏳ *(blocker Track B, bukan Track A)*
- [ ] Konfirmasi ke OCBC: whitelist rekening sumber (Finance) + limit teknis per-channel di sisi OCBC

### Client-side approval (di luar UPG)
- [ ] **ITRFD-621** — whitelist s/d approval CFO diselesaikan di sisi client

### Infra provisioning (UPG)
- [ ] PostgreSQL DB terpisah **`disbursement`**
- [ ] **Redis** (token OCBC, distributed lock, cache saldo)
- [ ] **GCP PubSub** topic `disbursement.status.changed` + skema subscription per-channel
- [ ] **Flipt** feature-flag namespace (rollout per channel/method)
- [ ] **ArgoCD** — namespace/deploy `upg-*` (GCP) + Huawei CCE
- [ ] Secret store (vault) untuk `privateKey`/`secretKey`/`clientSecret` + Elastic APM wiring

---

## M2 — Foundation (broker skeleton, provider-agnostic) — 🟢 SEBAGIAN BESAR SELESAI

- [x] Scaffold service via **`eneraplus init-service`** (skill `enera`; layout: `usecase/`+`transport/`+`repository/db/`+`repository/repoiface/`) 🔴
- [x] Tabel **`channel_registry`** — entitlement dinormalisasi ke 4 tabel (`provider`/`source_account`/`transfer_method`/`route`, bukan kolom `allowed_banks`/`allowed_transfer_methods` — lihat design §4), originator PPATK, `api_token_hash`, webhook (velocity columns **TIDAK** dibuat — OQ-5 dropped)
- [x] Tabel **`disbursements`** (`UNIQUE(channel, idempotency_key)`, `partner_reference_no` unik, raw req/resp jsonb audit) — DDL manual (`model/schema.sql`, belum ada tool migrasi)
- [x] Tabel **`disbursement_batches`** + **`bank_metadata`** (kosong, diisi DATA-1) + **`outbox`**
- [x] **Admin API internal** (`DisbursementAdmin` RPC di-merge ke `service Disbursement` — standar eneraplus 1-proto-1-service): `RegisterClient`, `IssueToken`/`RotateToken`, `RegisterProvider`, `RegisterSourceAccount`, `AddRoute`, `DisableRoute`, `AddTransferMethod` — **8/8 usecase GREEN via TDD**. REST `/internal/admin/*`
- [x] **Penerbitan token**: generate opaque acak (32-byte base62) → `api_token_hash`=SHA-256(hex) → plaintext sekali; `RotateToken` = baru+invalidate — teruji
- [x] Entitlement **opt-in** (`AddRoute` eksplisit per client×source_account×method, `DisableRoute`=soft-disable; tak ada auto-allow-all) — teruji
- [ ] **Auth** middleware per channel (B5): cocokkan `api_token_hash` dari metadata/header → set `channel` context — token issuance **sudah ada**, middleware intersepsi request **belum**
- [x] **Idempotency logic** (`Disbursement.GetByIdempotencyKey` + `model.ErrNotFound` sentinel; idempotent replay di `Disburse`) — teruji · [ ] **Redis distributed lock** anti double-submit — **belum** (Redis interface masih `HealthCheck` saja)
- [x] **Domain model** + **status FSM** enum (CREATED→SUBMITTED→PENDING→SUCCESS/FAILED/CANCELLED/REFUNDED/NEEDS_INVESTIGATION/REJECTED) — dipakai konsisten di `Disburse`/`GetStatus`; guard transisi FSM formal belum ada
- [ ] **Validator limit teknis** (amount min/max + jam operasional per method) — entitlement (B6) sudah ✅ via `findRoute`, limit teknis **belum**
- [x] Kontrak **gRPC** 8 RPC consumer (`GetAllowedRoutes`, `InquiryBeneficiary`, `Disburse`, `DisburseBatch`, `GetStatus`, `GetBatchStatus`, `BalanceInquiry`, `HistoryList`) — proto+REST lengkap; **usecase**: `GetAllowedRoutes`/`GetStatus`/`GetBatchStatus`/`Disburse` GREEN, 4 sisanya masih panic

---

## M3 — OCBC Provider Adapter — Track A (BIFAST + Intrabank) 🟢 — 🟡 ORKESTRASI ✅, ADAPTER NYATA ⏳

> Usecase `Disburse` sudah orkestrasi lengkap (Inquiry→Submit chaining, mapping status, error handling) **teruji via mock**. Yang **belum**: adapter OCBC sungguhan (HTTP + signature + mTLS) — `repository/ocbc/implementor.go` masih `panic("unimplemented")`.

- [ ] Interface **`DisbursementProvider`** — **2/5 method** (`InquiryBeneficiary`, `Submit` di `Ocbc` interface, dipakai `Disburse`); `GetStatus`/`BalanceInquiry`/`HistoryList` menyusul sesuai usecase terkait digarap
- [ ] OCBC adapter `repository/ocbc/`: token B2B (Redis, **auto-refresh <900s**), **Transaction Signature HMAC_SHA512** (`pkg/signature`), **mTLS** (`pkg/tls`) — **stub panic, belum diimplement**
- [ ] Generator **`partnerReferenceNo`** + **`X-EXTERNAL-ID`** — fungsi ada (`usecase/reference.go`) tapi pakai **prefix placeholder "BRK"**, bukan 4-digit prefix OCBC asli (pending M1 procurement)
- [x] Chaining **Inquiry → simpan `referenceNo` → Submit** — orkestrasi di usecase `Disburse` teruji (`submitToProvider`)
- [x] Isi **`originatorInfos`** (PPATK) dari `channel_registry` — `resolveUltimateSender` fallback ke `ch.Originator` (OQ-4), teruji
- [ ] Implement **BIFAST** (Inquiry-external → transfer-interbank) 🟢 — HTTP call nyata belum ada
- [ ] Implement **Intrabank/Overbooking** 🟢 — HTTP call nyata belum ada
- [ ] Mapping response code (00/01/02/03/04/05/06/07) + sinyal dedup `409...01`/`409...00` — target mapping (`model.TxnStatus`) sudah dipakai orkestrasi, tapi belum ada kode nyata yang memetakan response OCBC asli
- [ ] **Circuit breaker** + retry (retry **hanya** dari PENDING, tidak dari FAILED/REJECTED)

---

## M4 — Batch (broker-orchestrated — CONF-1: tak ada batch endpoint OCBC)

- [ ] **DisburseBatch**: validate-all-first (format, entitlement, limit, **total nominal vs saldo** di gate) → tolak seluruh batch bila ada invalid
- [ ] Eksekusi **per-item best-effort** (loop N call OCBC), `partner_reference_no` per-item, **idempotency batch-level + per-item**
- [ ] Ack sinkron: per-item ACCEPTED/REJECTED + `batchId`; status final per-item via event/polling/webhook
- [ ] `GetBatchStatus` (agregat + per-item)

---

## M5 — Async Status & Notifikasi (CONF-2: polling-only · k8s CronJob)

> **Mekanisme cron (keputusan 2026-07-08):** k8s CronJob → endpoint internal service (ClusterIP, internal-only, aman via jaringan Bluebird — tanpa auth tambahan). **Bukan** in-process `upg-scheduler`.

- [ ] **Endpoint internal** `POST /internal/jobs/poll-status` + `POST /internal/jobs/reconcile-history` (`delivery/worker`) — ClusterIP, **tidak** di-expose ke gateway publik 🔴
- [ ] **k8s CronJob poll-status**: `schedule: "* * * * *"`, `concurrencyPolicy: Forbid`, `startingDeadlineSeconds`, container ringan hit endpoint 🔴
- [ ] **k8s CronJob reconcile-history**: `schedule: "0 2 * * *"` (harian)
- [ ] **Klaim baris PENDING via `SELECT ... FOR UPDATE SKIP LOCKED`** (batch `limit`) — anti double-poll/double-emit (WAJIB, correctness) 🔴
- [ ] Poller: ambil PENDING > threshold → OCBC **Transfer Status** → update FSM → emit event
- [ ] Eskalasi: kode `07` / pending >24 jam → **NEEDS_INVESTIGATION** + alert
- [ ] **Reconciler** harian: OCBC **History List** vs DB → laporkan selisih (**termasuk vs H2H existing**)
- [ ] **Event publishing** ke PubSub `disbursement.status.changed` (atribut `channel`) via **outbox** (reliable publish)
- [ ] Subscription **filtered per `channel`** (isolasi multi-consumer)
- [ ] **Webhook dispatcher** (opsional per `channel_registry.webhook_url`): sign pakai `webhook_secret`, retry + DLQ

---

## M6 — Broker Surface & Query

- [ ] **Fasad SNAP REST** (`delivery/http`) — mirror operasi gRPC untuk konsumen SNAP
- [ ] **BalanceInquiry** + **HistoryList** pass-through (consumer-facing per PRD)
- [ ] **Error taxonomy** terklasifikasi (retriable vs final vs needs-escalation)
- [ ] `GetAllowedRoutes` end-to-end (client bisa tahu whitelist bank+method-nya)

---

## M7 — Security & Observability ⭐

- [ ] Secret management final (`privateKey`/`secretKey`/`clientSecret` di vault) + **rotasi**
- [ ] Validasi signature + mTLS ke `snapapi.ocbc.id` terverifikasi
- [ ] **Elastic APM** tracing end-to-end
- [ ] Metrik per channel/method (volume, success rate, latency, **saldo**)
- [ ] **Alert**: saldo rendah · pending menumpuk · success-rate turun · worker lag
- [ ] **Audit log** lengkap (channel, waktu, nominal, ref OCBC) — **PPATK-ready**

---

## M8 — Testing ⭐

- [~] Unit test usecase — **12/16 GREEN via TDD** (`GetAllowedRoutes`/`GetStatus`/`GetBatchStatus`/`Disburse` + 8 Admin API). Sisa panic: `DisburseBatch`, `InquiryBeneficiary`, `BalanceInquiry`, `HistoryList`. Validator/FSM/adapter integration test **belum** (adapter masih stub)
- [ ] **Sandbox** test channel Track A (success / pending / timeout / not-found)
- [ ] Verifikasi signature via **Postman** (`/v1.0/transaction-signature/b2b`)
- [ ] Test **idempotency & concurrency** (double-submit, duplicate `X-EXTERNAL-ID` & `partnerReferenceNo`)
- [ ] Test **batch** partial-failure (sebagian item invalid / gagal submit)
- [ ] Test **reconciliation** (selisih broker vs OCBC History)
- [ ] ⭐ **ASPI Functional Scenario & Developer Site Testing** (prasyarat produksi OCBC) 🔴

---

## M9 — Go-Live (Track A dulu) ⭐

- [ ] Deploy production (ArgoCD) — feature flag **OFF** untuk semua channel
- [ ] Whitelist channel **Finance** (bank=OCBC, method=BIFAST+Intrabank) di `channel_registry`
- [ ] **Onboard pilot** consumer (Finance) — 1 channel
- [ ] Validasi **end-to-end di prod** (nominal kecil) → cek FSM, event, reconciliation
- [ ] Handling **EOD window** BIFAST/OLT (00.00–05.01) terverifikasi
- [ ] **Runbook** + on-call + eskalasi OCBC
- [ ] Rollout bertahap (naikkan flag per channel/method)
- [ ] ✅ **GO-LIVE Track A**

---

## M10 — Post Go-Live / Track B & Fase Lanjut 🟡

- [ ] **DATA-1** attachment metadata bank diterima → isi tabel `bank_metadata`
- [ ] Implement **OLT** (transfer + metadata + `transactionPurpose`) 🟡
- [ ] Implement **RTGS** (metadata + regulatory category + sender detail + working-hours guard + `bankRef` from inquiry) 🟡
- [ ] Implement **SKN** (metadata + regulatory + `referenceNo` from inquiry + working-hours guard) 🟡
- [ ] Working-hours guard RTGS/SKN + finalitas 1–3 hari ter-handle
- [ ] Whitelist OLT/RTGS/SKN per channel yang membutuhkan → rollout bertahap
- [ ] **Onboard consumer ke-2** (mis. refund MRG) end-to-end
- [ ] *(Opsional/future)* Provider bank **kedua** (validasi abstraksi `DisbursementProvider`) — kontrak consumer tetap
- [ ] *(Opsional/future)* Unified reporting via **event projection** (SSOT terpusat)
- [ ] *(Ditunda)* **Velocity/anti-abuse guard** (OQ-5) — hanya bila kebutuhan bisnis muncul; tambah kolom + enforcement tanpa ubah kontrak

---

## Ringkasan Blocker & Dependency

| Blocker | Menahan | Mitigasi |
|---|---|---|
| RFC approve (M0) | Mulai implementasi | Tulis RFC segera |
| OCBC credential + mTLS (M1) ⏳ | Semua submit ke OCBC | Mulai procurement paralel, lead-time panjang |
| ASPI Testing (M8) | Go-live produksi | Jadwalkan setelah sandbox stabil |
| **DATA-1** metadata (M1/M10) | **Hanya Track B** (OLT/RTGS/SKN) | Go-live Track A (BIFAST+Intrabank) duluan, tak nunggu DATA-1 |

---

## Jalur Kritis (critical path)

```
RFC approve ──┐
              ├─► M2 Foundation ─► M3 Track A ─► M4 Batch ─► M5 Poller/Event ─► M8 Test ─► ASPI ─► M9 GO-LIVE (Track A)
OCBC cred ────┘                                                                                         │
DATA-1 ─────────────────────────────────────────────────────────────────────────────► M10 Track B ◄───┘
```

---
_Delivery checklist — dibuat 2026-07-08 dari design + requirements. Update centang seiring eksekusi._
