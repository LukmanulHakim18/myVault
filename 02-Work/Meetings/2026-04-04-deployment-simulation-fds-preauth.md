---
date: '2026-04-04'
status: in-progress
tags:
  - deployment
  - fds
  - preauth
  - mrg
  - upg
  - bbd
type: meeting
---
# Deployment Simulation — FDS Pre-auth

**Date:** 4 April 2026 | **Commander:** Lukmanul
**Teams:** MRG · UPG · BBD
**Dependency:** UPG → MRG → BBD

---

## PRE-DEPLOYMENT

- [x] Konfirmasi semua PIC standby: Eko, Abdul, Alfian M, Aldhonie, Nuur Rohman, Nadila, Dian, Eka
- [x] Verifikasi pre-requisite: **Service MyBB bisa dihit dari BBD**
- [x] Pastikan rollback plan sudah siap di masing-masing squad
- [x] Konfirmasi QA siap sebelum QA Test dimulai

---

## Step 1 — QA Test Preparation — PIC: Abdul, Nadila, Dian

- [x] Prepare Order Gantung
- [x] Konfirmasi ke commander: QA preparation selesai ✅

---

## Step 2 — UPG: Indexing — PIC: Nuur Rohman

- [x] Change indexing table `charge_tracking`
- [x] Konfirmasi ke commander: indexing selesai ✅

---

## Step 3 — UPG Deploy — PIC: Eko

Bug Fix Partial Paid CC:
- [x] Add whitelist BBID
- [x] Deploy `card-payment`
- [x] Deploy `mrg-gateway`
- [x] Deploy `service-transaction`
- [x] Konfirmasi ke commander: UPG deploy selesai ✅

## Step 4 — MyBB Deploy — PIC: Alfian M

- [x] Update tagging version to **6.24**
- [x] Add whitelist BBID
- [x] Konfirmasi ke commander: MyBB deploy selesai ✅

---

## Step 5 — [QA] UPG — PIC: Abdul

- [x] UPG: Test CC non-whitelist
- [x] Konfirmasi ke commander: UPG QA pass ✅

## Step 6 — [QA] BBD Pre-switch — PIC: Bagus

- [ ] BBD: Create order e-wallet untuk end trip sebelum switching
- [ ] Konfirmasi ke commander: pre-switch order created ✅
skiped IOT not working

---

## Step 7 — BBD Deploy — PIC: Aldhonie

- [x] Deploy `payment-watch`
- [x] Change flagging to **V2** & exclude order GB
- [x] Konfirmasi ke commander: BBD deploy selesai ✅

---

## Step 8 — [QA] BBD Post-switch — PIC: Abdul, Nadila, Eka

- [ ] BBD: End trip order after switch
- [ ] Konfirmasi ke commander: BBD post-switch QA pass ✅
skiped IOT not working

---

## Step 9 — BBD: Whitelist Payment CC — PIC: Nuur Rohman

- [x] Add whitelist payment CC
- [x] Konfirmasi ke commander: whitelist CC selesai ✅

---

## Step 10 — [QA] Full Flow — PIC: Abdul, Nadila, Dian, Eka

**MyBB:**
- [x] Initial Pre-Auth & Increment berjalan normal

**BBD:**
- [x] N2C berjalan dengan benar
- [x] VINI monitoring alert watch aktif

**UPG:**
- [x] Pre-auth CC berjalan normal
- [x] Non-pre-auth CC berjalan normal
- [x] Pre-auth E-wallet berjalan normal
- [x] Non-pre-auth E-wallet berjalan normal

- [ ] Konfirmasi ke commander: Full flow QA pass ✅

---

## Step 11 — Monitoring Error after Deployment

**PIC:** Eko (UPG) · Alfian M (MyBB) · Aldhonie (BBD)

- [x] Monitor error rate UPG — Kibana/Grafana
- [x] Monitor error rate MyBB — Kibana/Grafana
- [x] Monitor error rate BBD — Kibana/Grafana
- [x] Tidak ada spike error abnormal dalam 30 menit pertama
- [ ] Commander declare: **DEPLOYMENT SUCCESS** / **ROLLBACK**

---

## ROLLBACK TRIGGERS

> Jika salah satu kondisi ini terjadi, segera eskalasi ke commander:

- [ ] Error rate naik > threshold normal
- [ ] Order gagal untuk whitelisted user
- [ ] Non-whitelisted user terdampak (order gagal)
- [ ] Service down salah satu komponen
