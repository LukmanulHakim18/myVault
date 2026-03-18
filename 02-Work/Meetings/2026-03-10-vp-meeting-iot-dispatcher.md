---
date: '2026-03-10'
type: meeting
attendees:
  - VP
tags:
  - meeting
  - iot
  - dispatcher
  - mybb
  - da
---
# Meeting Notes - VP: IoT & Dispatcher Direction

**Date:** 2026-03-10  
**With:** VP  

---

## Key Points

### 1. Dispatcher — Tidak Relevan Lagi
- Dispatcher sudah **deprecated/ditolak** karena mengandalkan IoT
- Isu utama: IoT bisa mati (tidak reliable) → dispatcher di-reject

### 2. IoT — Harus Relevan dengan DA (Driver Apps)
- IoT hanya valid jika terhubung dan relevan dengan **Driver Apps (DA)**
- IoT standalone tanpa konteks DA = tidak acceptable

### 3. Pesan MyBB & DA — Semua Dinamis
- Pesan di **MyBluebird (MyBB)** dan **Driver Apps (DA)** harus bersifat **dinamis**
- Tidak boleh hardcoded/statis

---

## Action Items
- [ ] Review dampak keputusan ini terhadap arsitektur MRG
- [ ] Identifikasi komponen yang masih bergantung pada dispatcher
- [ ] Alignment dengan tim terkait dynamic messaging untuk MyBB & DA
