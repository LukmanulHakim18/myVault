---
tags:
  - trading
  - state-machine
  - design
created: '2026-01-20'
---
# State Machine Design

## 1️⃣ PRINSIP DASAR (WAJIB)

1. Setiap order punya state
2. State hanya boleh maju, tidak mundur
3. State disimpan (persisted) → survive restart
4. UI broker = source of truth terakhir

---

## 2️⃣ ORDER STATE MACHINE

```mermaid
stateDiagram-v2
    [*] --> PENDING
    PENDING --> SUBMITTED : submit order
    SUBMITTED --> FILLED : order matched
    SUBMITTED --> REJECTED : broker reject
    FILLED --> DONE : complete
    REJECTED --> FAILED : no retry
```

### Definisi State

| State | Makna |
|-------|-------|
| `PENDING` | Order belum dikirim (nunggu jam / harga) |
| `SUBMITTED` | Order sudah diklik & dikirim ke broker |
| `FILLED` | Order match (full / partial*) |
| `REJECTED` | Ditolak broker (harga, lot, dll) |
| `DONE` | Order selesai & valid |
| `FAILED` | Order gagal & tidak akan diulang |

📌 Partial fill diperlakukan sebagai `FILLED` + flag

---

## 3️⃣ TASK STATE MACHINE (LEVEL STRATEGY)

Satu task bisa punya 3 order:
- BUY
- SELL (TP)
- SELL (CL)

### Task State Flow

```mermaid
stateDiagram-v2
    [*] --> WAIT_ENTRY
    WAIT_ENTRY --> BUY_SUBMITTED : submit buy
    BUY_SUBMITTED --> BUY_FILLED : buy matched
    BUY_FILLED --> WAIT_EXIT : ready for exit
    WAIT_EXIT --> EXIT_SUBMITTED : submit TP/CL
    EXIT_SUBMITTED --> EXIT_FILLED : exit matched
    EXIT_FILLED --> DONE : complete
    DONE --> [*]
```

Mapping:
- BUY order → ORDER STATE
- EXIT order → ORDER STATE
- TASK state = agregasi

---

## 4️⃣ STRUKTUR STATE (DATA MODEL)

Contoh state yang disimpan lokal (file / DB kecil):

```json
{
  "account": "ACC_003",
  "emiten": "BBCA",
  "task_id": "TASK_20260120_01",
  "state": "WAIT_EXIT",
  "buy_order": {
    "price": 2700,
    "lot": 20,
    "state": "FILLED",
    "order_id": "SB123456"
  },
  "exit_order": {
    "type": "TP",
    "price": 3000,
    "state": "PENDING"
  }
}
```

📌 Ini single source of truth robot

---

## 5️⃣ TRANSISI STATE (RULE KERAS)

### ❌ Robot TIDAK BOLEH:
- Submit BUY jika state ≠ `WAIT_ENTRY`
- Submit EXIT jika BUY belum `FILLED`
- Submit ulang order yang sudah `SUBMITTED`

### ✅ Robot HANYA BOLEH:
- Advance state jika UI broker mengonfirmasi

---

## 6️⃣ DETEKSI STATE DARI UI

```mermaid
flowchart TD
    POLL["Poll UI<br/>(1-2 detik)"]
    
    POLL --> CHECK_SUBMIT{"Order muncul<br/>di Open Orders?"}
    CHECK_SUBMIT -->|Yes| SUBMITTED["✅ SUBMITTED"]
    
    POLL --> CHECK_FILL{"Order hilang dari<br/>Open Orders?"}
    CHECK_FILL -->|Yes| CHECK_HISTORY{"Cek Order History"}
    CHECK_HISTORY -->|"Status: Filled"| FILLED["✅ FILLED"]
    CHECK_HISTORY -->|"Status: Rejected"| REJECTED["❌ REJECTED"]
    
    POLL --> CHECK_REJECT{"Popup Error?"}
    CHECK_REJECT -->|Yes| REJECTED
```

📌 Robot poll UI (interval pendek, mis. 1–2 detik)

---

## 7️⃣ RESTART & CRASH RECOVERY

Saat robot restart:

```mermaid
flowchart TD
    START["🔄 Robot Restart"] --> LOAD["1. Load State File"]
    LOAD --> OPEN["2. Buka Broker UI"]
    OPEN --> CROSS["3. Cross-check"]
    
    CROSS --> CHECK_OPEN["Cek Open Orders"]
    CROSS --> CHECK_PORT["Cek Portfolio"]
    
    CHECK_OPEN --> RECONCILE["4. Reconcile"]
    CHECK_PORT --> RECONCILE
    
    RECONCILE --> R1{"Order ada &<br/>State = SUBMITTED"}
    R1 -->|Yes| A1["Lanjut Monitor"]
    
    RECONCILE --> R2{"Order tidak ada &<br/>State = SUBMITTED"}
    R2 -->|Yes| A2["Cek History"]
    
    RECONCILE --> R3{"Saham ada &<br/>State = BUY_SUBMITTED"}
    R3 -->|Yes| A3["Set FILLED"]
    
    RECONCILE --> R4{"Tidak ada apa-apa &<br/>State = SUBMITTED"}
    R4 -->|Yes| A4["Mark FAILED"]
```

📌 Tidak ada blind submit ulang

---

## 8️⃣ EVENT REPORTING

Setiap state change → kirim event ke server:

```json
{
  "event": "ORDER_STATE_CHANGED",
  "from": "SUBMITTED",
  "to": "FILLED",
  "account": "ACC_003",
  "emiten": "BBCA"
}
```

Server tidak menentukan state, hanya menerima.

---

## 9️⃣ EDGE CASES YANG SUDAH DI-COVER

- ✅ Partial fill
- ✅ Broker lag
- ✅ UI freeze
- ✅ Robot restart
- ✅ Server mati
- ✅ Internet drop sesaat

---

## 🔟 RANGKUMAN

- Order selalu punya state
- Task = kumpulan order
- State hanya maju
- UI broker = validator terakhir
- Persist state = aman dari double order

---

## ✅ STATUS DESAIN

| Area | Status |
|------|--------|
| Order State Machine | ✅ Final |
| Task State Machine | ✅ Final |
| Recovery Logic | ✅ Final |
| Safe from double order | ✅ Yes |
