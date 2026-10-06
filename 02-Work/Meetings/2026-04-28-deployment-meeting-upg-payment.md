---
title: '[2026.04.06] Deployment Meeting — UPG / Payment'
date: '2026-04-28'
time: '22:00 WIB'
type: meeting
tags:
  - deployment
  - upg
  - payment
  - production
project: UPG / Payment
status: scheduled
tech_lead: Eko Nugroho
pm: Rayan Nurbadi
---
# [2026.04.06] Deployment Meeting — UPG / Payment

## Overview

| Field | Details |
|-------|---------|
| **Project** | UPG / Payment |
| **Deployment Type** | New Release |
| **Deployment Date** | 2026-04-28, pukul **22:00 WIB** |
| **Tech Lead** | Eko Nugroho |
| **Product Manager** | Rayan Nurbadi |
| **Target Environment** | Production |
| **ClickUp Tasks** | [UPGN-3753](https://app.clickup.com/t/9018711461/UPGN-3753) · [UPGN-3752](https://app.clickup.com/t/9018711461/UPGN-3752) |

---

## Team Members

| Nama | Role |
|------|------|
| Eko Nugroho | Tech Lead |
| Rayan Nurbadi | Product Manager |
| Muhammad Marwan Faisal | BE |
| Sahrul Ramdoni | QA |
| Abdul Karman | QA |
| Adam Haniif | DevOps |

---

## Pre-Deployment Checklist

### QA Notes

| No | Note | PIC | Status |
|----|------|-----|--------|
| 1 | Testing on staging env passed | Sahrul Ramdoni / Abdul Karman | ✅ Ready to deploy |

### Code Quality

| No | Service | Coverage | Duplication | Quality Gate |
|----|---------|----------|-------------|--------------|
| 1 | scheduler | 85.1% | 1.8% | BB Level 4 |
| 2 | MPG2 | 27.9% | 11.7% | BB Level 0 |

### Product Manager Checklist

- [x] PM sudah aware terhadap QA Notes dan Risk Analysis
- [x] Stakeholder sudah diberitahu (Email / WA)

### Pre-requisites / Issue yang Perlu Diperhatikan

- ⚠️ `pq: cannot execute INSERT in a read-only transaction`

---

## Deployment Steps

| No | Service | MR Link | PIC | Status |
|----|---------|---------|-----|--------|
| 1 | scheduler | [MR #1183](https://git.bluebird.id/upg/mrg-gateway/-/merge_requests/1183) | Eko Nugroho | ⬜ |
| 2 | mpg2 | [MR #1183](https://git.bluebird.id/upg/mrg-gateway/-/merge_requests/1183) | Eko Nugroho | ⬜ |
| 3 | PVT / Test After Live | [Scenario Test](https://bluebirdgroup365.sharepoint.com/:x:/s/UPG/ETTPqKbH4RRLuUO6CrKYTL0BB3gtjBwR6FCfymQtRtIcWA?e=zi9h8Y) | Abdul Karman | ⬜ |
| 4 | Monitoring semua service post-deploy | KMS · GMS | All member | ⬜ |

---

## Rollback Plan

| No | Action | PIC | Status |
|----|--------|-----|--------|
| 1 | Rollback Image | DevOps | ⬜ |
| 2 | Monitoring all services setelah rollback (KMS, GMS) | All member | ⬜ |

---

## Meeting Notes

> _Isi saat meeting berlangsung_

- 
