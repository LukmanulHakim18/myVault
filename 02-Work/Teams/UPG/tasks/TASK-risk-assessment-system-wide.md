---
created: '2026-04-20'
priority: high
status: todo
tags:
  - task
  - risk
  - assessment
  - mrg
  - upg
  - bbd
title: 'TASK: System-Wide Risk Assessment'
updated: '2026-04-20'
---
# TASK: System-Wide Risk Assessment

> **Status:** TODO  
> **Priority:** High  
> **Created:** 2026-04-20  
> **Owner:** Lukmanul Hakim

---

## Tujuan

Mengidentifikasi dan mempresentasikan 2–3 risiko terbesar pada sistem saat ini, mencakup tim MRG, UPG, dan BBD. Setiap team member diminta mempresentasikan risiko yang mereka ketahui agar dapat diassess bersama.

---

## Action Items

- [ ] Minta setiap team member (MRG, UPG, BBD) untuk menyiapkan presentasi singkat tentang risiko terbesar di area masing-masing
- [ ] Jadwalkan sesi risk review bersama ketiga tim
- [ ] Dokumentasikan hasil assessment ke ADR atau Risk Register

---

## Kandidat Risiko untuk Diassess

### 🔴 Risk 1: Kafka Single-Zone (MRG)

**Deskripsi:**  
Kafka belum dikonfigurasi multi-zone. Jika satu zone Huawei Cloud (HWC) mengalami failure, seluruh messaging pipeline MRG akan terdampak — termasuk alur booking, ride event, dan notifikasi.

**Dampak potensial:** Downtime P1, semua transaksi real-time terganggu.  
**Owner untuk presentasi:** Team MRG (Infrastructure lead)

---

### 🔴 Risk 2: Business Logic di API Gateway (UPG)

**Deskripsi:**  
Sebagian business logic UPG masih berada di API Gateway, bukan di downstream service yang bertanggung jawab. Ini menyebabkan coupling tinggi, sulit di-test secara independen, dan rawan regression saat ada perubahan.

**Dampak potensial:** Bug tersembunyi saat cross-payment atau fitur baru, bottleneck di satu titik.  
**Owner untuk presentasi:** Team UPG (Backend lead)

---

### 🔴 Risk 3: JWT Auth Token — Stateless vs Invalidation Policy (MRG)

**Deskripsi:**  
Implementasi JWT saat ini tidak bisa di-invalidate sebelum expired. Jika kebijakan logout dari stakeholder diaktifkan, tidak ada mekanisme untuk membatalkan token yang sudah diterbitkan. Risiko keamanan nyata terutama jika akun user dikompromikan.

**Dampak potensial:** Security breach, non-compliance terhadap kebijakan internal.  
**Owner untuk presentasi:** Team MRG (Auth service owner)

---

## Format Presentasi per Tim

Setiap presentasi diharapkan mencakup:
1. **Deskripsi risiko** — apa yang bisa salah
2. **Probabilitas** — seberapa mungkin terjadi
3. **Dampak** — severity jika terjadi
4. **Mitigasi yang sudah ada** (jika ada)
5. **Rekomendasi** — langkah mitigasi ke depan

---

## Referensi

- [[02-Work/Teams/MRG/tech-debt]]
- [[02-Work/Teams/UPG/tech-debt]]
