---
title: 'Spike: New CC Charging Flow'
date: '2026-02-12'
status: draft
tags:
  - spike
  - payment
  - cc
  - charging
  - mrg
  - upg
  - bbd
---
# Spike: New CC Charging Flow

**Date:** 2026-02-12  
**Updated:** 2026-02-12  
**Status:** Draft  
**Teams:** BBD Dispatching, MyBB Orchestrator, UPG Payment

---

## Background

### Current Flow
- User di-preauth dengan estimasi saat order
- Setelah dropoff, tunggu **30 menit** untuk charging (menunggu tips & extra dari driver)
- Charge total: argo + discount + platform fee + extra + tips

### Masalah
1. **Order gantung** - CC gagal charging, payment tersangkut, retry terus-menerus
2. **Fraud** - User matikan internet transaction di mobile sehingga charging gagal

### Solusi Sementara (Current)
- Multiple preauth selama trip
- BBD query MRG setiap 20 detik untuk check n2c (nontunai to cash)
- Jika actual argo > preauth terakhir → cancel preauth → preauth baru (+ fixed threshold)
- Jika preauth gagal → switch to cash → notif user + IoT "tagih tamu"

---

## Goals
1. Menghapus 30 menit waiting time
2. Memisahkan extra dan tips dari biaya perjalanan utama
3. Driver segera tau jika payment switch ke cash

---

## Proposed Flow

### Selama Trip
```
BBD query MRG setiap 20s
    → Compare actual argo vs preauth terakhir
    → Jika argo > preauth:
        → Cancel preauth lama
        → Preauth baru (+ fixed threshold)
    → Jika preauth gagal:
        → n2c = true
        → Switch to cash
        → Push notif ke user
        → IoT tampilkan "tagih tamu"
```

### Saat Complete (Driver Submit)
```
Driver di IoT:
    → Stop Argo
    → Input Extra (bisa 0)
    → Complete

BBD:
    → Stop interval query (prevent race condition)
    → Kirim Complete event ke MRG

MRG:
    → Jika total ≤ preauth terakhir:
        → Capture preauth (partial?) ← perlu konfirmasi UPG
    → Jika total > preauth terakhir:
        → Cancel preauth
        → Direct charge (biaya perjalanan + extra)
    → Jika charge gagal:
        → n2c = true
        → Switch to cash
        → Push notif ke user
        → IoT tampilkan "tagih tamu"
```

### Tips (Terpisah)
```
User input tips (dalam 24 jam)
    → Charge terpisah
    → Jika gagal → skip (tips optional)

Future consideration:
    → Tips bisa pakai payment method berbeda (e-wallet, dll)
```

---

## Key Changes

| Aspek        | Current                   | New                              |
| ------------ | ------------------------- | -------------------------------- |
| Waiting time | 30 menit setelah dropoff  | Langsung charge saat Complete    |
| Extra        | Digabung setelah 30 menit | Digabung saat Complete           |
| Tips         | Digabung setelah 30 menit | Terpisah, 24 jam timeout         |
| Preauth      | Capture di akhir          | Direct charge, preauth di-cancel |

---

## Race Condition Handling

**Concern:** Interval 20s bisa trigger bersamaan dengan driver Complete

**Solution (State-based di BBD):**
- BBD stop polling n2c **saat kirim event endtrip** ke MRG
- Event endtrip dikirim async ke MRG
- Tidak ada race karena polling sudah berhenti sebelum MRG proses charging

```
Selama Trip:
    BBD polling n2c setiap 20s

Saat Endtrip:
    BBD stop polling n2c 
    → BBD kirim event endtrip ke MRG (async)
    
MRG terima endtrip:
    → Direct charge (biaya perjalanan + extra)
    → Jika gagal:
        → Update payment method di MyBB
        → Info ke bb-trip
        → bb-trip eskalasi ke DA + IoT → "tagih tamu"
```

> **✅ Confirmed by BBD Architect (2026-02-12):**
> - Polling n2c akan diberhentikan ketika event endtrip dikirim ke MRG
> - Jika MRG gagal charging, MRG update payment method dan info ke bb-trip
> - bb-trip akan eskalasi ke tim DA (Driver Apps) dan tim IoT untuk notif driver "tagih tamu"

---

## Action Items

| No  | Task                                                              | PIC    | Status     |
| --- | ----------------------------------------------------------------- | ------ | ---------- |
| 1   | Konfirmasi ke UPG: bisa partial capture dari preauth?             | Lukman | **Done** ✅ |
| 2   | Konfirmasi ke BBD: feasible stop interval sebelum Complete event? | Lukman | **Done** ✅ |
| 3   | Testing rate limit preauth cancel → preauth baru                  | UPG    | Pending    |
| 4   | Define contract MRG → bb-trip untuk info payment switch           | Lukman | Pending    |

---

## UPG Confirmation (2026-02-12)

Enhancement untuk feature Complete Transaction:

| Scenario | Handling |
|----------|----------|
| authorize amount > capture amount | Sisa dikembalikan ke user, capture amount diamankan (experimental) |
| authorize amount < capture amount | Capture sesuai authorize amount, sisa jadi **outstanding payment** (partial charge) |

**Outstanding Payment Handling:**
- UPG scheduler akan retry charging untuk sisa outstanding
- Tidak ada push notif ke user
- Info outstanding ada di order history
- User tidak bisa order baru sampai outstanding selesai

---

## Order States Reference

```go
ORDER_STATE_NONE                                = -4 // for state order failed
ORDER_STATE_TIME_OUT                            = -3
ORDER_STATE_GOLDENBIRD_AIRPORT_TRANSFER_INITIAL_STATE = -3
ORDER_STATE_INITIAL_STATE                       = -2
ORDER_STATE_ORDER_FAILED                        = -1
ORDER_STATE_LOOKING_FOR_DRIVER                  = 0
ORDER_STATE_EN_ROUTE                            = 1
ORDER_STATE_CANCELED                            = 2
ORDER_STATE_NO_TAXI                             = 3
ORDER_STATE_ON_TRIP                             = 4 // telah sampai
ORDER_STATE_NO_SHOW                             = 5
ORDER_STATE_TRACKING                            = 6 // start tracking
ORDER_STATE_ECV_DROP_OFF                        = 7
ORDER_STATE_DROP_OFF                            = 8
ORDER_STATE_OFFLINE_DISPATCH                    = 9
```
