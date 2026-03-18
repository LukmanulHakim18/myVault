---
title: CC Charging Flow - Project Documentation
type: index
status: active
team: MRG
created: '2026-02-12'
tags:
  - cc
  - charging
  - payment
  - n2c
  - mrg
  - upg
  - bbd
---
# CC Charging Flow - Project Documentation

**Last Updated:** 2026-02-12

---

## Overview

Dokumentasi lengkap untuk flow charging Credit Card di MyBluebird, mencakup:
1. **N2C (Ngecharge to Cloud)** - Pre-auth validation selama trip
2. **New Charging Flow** - Perubahan flow saat Complete (menghapus 30 menit waiting)

---

## Documents

| Document | Description | Status |
|----------|-------------|--------|
| [[01-n2c-flow-documentation]] | N2C service: pre-auth, cancel pre-auth, switch to cash | Approved |
| [[02-spike-new-charging-flow]] | Spike: direct charge saat Complete, pisahkan tips | Draft |
| [[03-cc-charging-sequence-diagrams]] | Sequence diagrams: preauth polling, stop argo, endtrip | Draft |
| [[04-bbd-source-of-truth-force-cash]] | BBD source of truth: force-to-cash mechanism, SLA timeout, reporting | Draft |

---

## Teams Involved

- **BBD Dispatching** - Trigger n2c validation, order state management
- **MRG (MyBB Orchestrator)** - N2C service, fare calculation
- **UPG Payment** - Pre-auth, cancel, charge execution

---

## Key Goals

1. Prevent fraud dari user yang matikan internet transaction
2. Menghapus 30 menit waiting time setelah dropoff
3. Memisahkan extra dan tips dari biaya perjalanan utama
4. Driver segera tau jika payment switch ke cash

---

## Related Links

- MRG Team Docs: [[../../MRG-SPOF-Documentation-Hub]]
