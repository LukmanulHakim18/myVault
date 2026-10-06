---
title: Tech Debt - MRG
tags:
  - tech-debt
  - mrg
created: '2026-04-15'
updated: '2026-04-15'
status: active
---
# Tech Debt - MRG

> Last updated: 2026-04-15

## Daftar Tech Debt

### 1. Kafka Multi-Zone
- Kafka belum dikonfigurasi untuk multi-zone availability.
- Risiko: single zone failure dapat mempengaruhi keseluruhan messaging pipeline.

### 2. Pub/Sub Spike Region Jakarta
- Terdapat spike yang belum diinvestigasi/dimitigasi pada region Jakarta.
- Perlu analisis root cause dan solusi mitigasi (rate limiting, scaling, atau redistribusi).

### 3. JWT Auth Token
- Implementasi JWT saat ini bertolak belakang dengan kepentingan stakeholder terkait kebijakan logout.
- Jika stakeholder tidak mengizinkan logout, stateless JWT menjadi conflict karena token tidak bisa di-invalidate sebelum expired.
- Perlu alignment dengan stakeholder untuk menentukan pendekatan: blacklist token, short-lived token + refresh token, atau kebijakan lain.

### 4. Ride Revamp
- Revamp alur ride masih memiliki sisa tech debt yang belum diselesaikan.
- Detail item spesifik perlu di-breakdown lebih lanjut.

### 5. Logger Implementation Standardisation
- Implementasi logger belum standar di seluruh service MRG.
- Perlu definisi standar: format log (JSON/text), level (info/warn/error), correlation ID, dan library yang digunakan.
