---
created: '2026-04-15'
status: active
tags:
  - tech-debt
  - upg
title: Tech Debt - UPG
updated: '2026-04-15'
---
# Tech Debt - UPG

> Last updated: 2026-04-15

## Daftar Tech Debt

### 1. Memindahkan Logic ke Service yang Semestinya
- Saat ini sebagian logic masih berada di API Gateway, bukan di service yang bertanggung jawab.
- **Contoh:** API Gateway masih mengandung business logic yang seharusnya ada di downstream service.
- Target: API Gateway hanya bertugas sebagai routing/auth proxy, tidak memiliki business logic.

### 2. Single Point of Responsibility
- Terkait dengan poin di atas: beberapa service belum mengikuti prinsip single responsibility.
- Logic yang tersebar perlu dikonsolidasi agar setiap service memiliki tanggung jawab yang jelas dan tidak overlapping.
- Refactor bertahap diperlukan dengan memastikan test coverage sebelum migrasi logic.
