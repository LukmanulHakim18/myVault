---
title: Multi-select Fleets RFC Index
tags:
  - rfc
  - mrg
  - multi-select-fleets
  - index
created: '2026-03-16'
updated: '2026-03-18'
---
# Multi-select Fleets RFC Index

**Status:** Draft  
**Created:** 2026-03-16  
**Updated:** 2026-03-18  
**Team:** MRG

---

## Overview

Dokumentasi RFC untuk fitur Multi-select Fleets yang memungkinkan user memilih lebih dari satu fleet dalam satu order.

---

## RFC Documents

| RFC | Title | Status | Description |
|-----|-------|--------|-------------|
| [[RFC-001-multi-select-fleets-overview]] | Overview | Draft | High-level architecture, flow, dan business context |
| [[RFC-002-session-manager-fleet-pricing]] | Session Manager | Draft | Fleet pricing storage (TTL 10 menit) |
| [[RFC-003-order-orchestrator-multi-fleet]] | Order Orchestrator | Draft | Database storage, dispatcher integration |
| [[RFC-004-payment-flow-multi-fleet]] | Payment Flow | Draft | Lock amount calculation, UPG integration |

---

## Key Design Decisions

### 1. Two-Phase Timeout
- **Fase 1 (Session Manager):** 10 menit — pricing validity
- **Fase 2 (Dispatcher):** 30 menit — retry mencari driver

### 2. Persistent Storage saat Create Order
- Fleet pricing di-copy ke `order_fleet_pricing` table
- Harga fixed selama fase retry
- Session Manager hanya untuk fase pencarian

### 3. Lock Amount = Harga Tertinggi
- Lock berdasarkan harga tertinggi dari selected fleets
- User eligible untuk semua fleet selection
- UPG handle partial charge + refund

### 4. Service Info → Price Engine
- Service Info yang call Price Engine (bukan Order Orchestrator)
- Service Info store ke Session Manager
- `item_id` + `price_code` dikirim ke Dispatcher untuk BBD translate

---

## Scope Summary

### MRG Scope

| Service | Changes |
|---------|---------|
| Service Info | Call Price Engine, store ke Session Manager |
| Session Manager | Store fleet pricing (TTL 10 menit) |
| Order Orchestrator | Copy ke `order_fleet_pricing`, lock highest amount, forward ke Dispatcher |
| UPG | Lock balance, partial charge, release sisa |

### Non-MRG Scope

| Service | Owner | Changes |
|---------|-------|---------|
| Price Engine | Tim Lain | Fleet eligibility, pricing, `item_id` + `price_code` |
| Dispatcher | Tim Lain | Scoring, winner selection, retry 30 menit |
| BBD | Tim Lain | Translate `item_id` + `price_code` via Price Engine |
| MyBB Apps | Tim Lain | UI multi-select checkbox |

---

## Database Schema

```sql
CREATE TABLE order_fleet_pricing (
    id              BIGSERIAL PRIMARY KEY,
    order_id        VARCHAR(36) NOT NULL,
    fleet_code      VARCHAR(20) NOT NULL,
    price_type      VARCHAR(10) NOT NULL,
    item_id         VARCHAR(50) NOT NULL,
    price_code      VARCHAR(50) NOT NULL,
    estimated_fare  BIGINT NOT NULL,
    platform_fee    BIGINT NOT NULL,
    total_price     BIGINT NOT NULL,
    created_at      TIMESTAMP DEFAULT NOW(),
    
    CONSTRAINT fk_order FOREIGN KEY (order_id) REFERENCES orders(id),
    CONSTRAINT uq_order_fleet UNIQUE (order_id, fleet_code)
);
```

---

## Open Items

| # | Item | Owner | Status |
|---|------|-------|--------|
| 1 | Tier grouping confirmation | Product | Pending |
| 2 | Dispatcher API contract | Tim Dispatcher | Pending |
| 3 | BB Prime polygon definition | Product | Pending |
| 4 | Price Engine integration | Tim Price Engine | Pending |
| 5 | CC settlement window untuk cancel | UPG | Pending |

---

## Timeline

TBD - Pending dependency resolution

---

## References

- Source Document: Multi-select Fleets PRD (Mar 2025)
- Related: [[dynamic-platform-fee]] (DPF untuk platform fee calculation)
- Related: [[fds-sequential-preauth]] (handling Argo actual > estimasi)
