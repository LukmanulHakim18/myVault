---
tags:
  - trading
  - task
  - json
created: '2026-01-20'
---
# Task JSON Format

Tasks are pulled from owner server before market opens.

---

## Task Constraints

| Rule | Enforced By |
|------|-------------|
| No duplicate emiten per account per day | ✅ Server |
| Valid price (within ARA/ARB) | ❌ Broker (robot submit as-is) |
| Valid lot | ❌ Broker (robot submit as-is) |

📌 **Server guarantees**: Tidak akan ada 2 task dengan emiten yang sama untuk 1 account dalam 1 hari.

---

## Task Structure

```json
{
  "account": "ACC_003",
  "tasks": [
    {
      "emiten": "BBCA",
      "buy": {
        "price": 2700,
        "lot": 20,
        "valid_from": "09:00",
        "valid_until": "10:30"
      },
      "take_profit": { "price": 3000 },
      "cut_loss": { "price": 2300 }
    }
  ]
}
```

---

## Field Descriptions

### Root Level

| Field | Type | Description |
|-------|------|-------------|
| `account` | string | Account identifier |
| `tasks` | array | List of trading tasks |

### Task Object

| Field | Type | Description |
|-------|------|-------------|
| `emiten` | string | Stock ticker symbol |
| `buy` | object | Buy order configuration |
| `take_profit` | object | TP configuration |
| `cut_loss` | object | CL configuration |

### Buy Object

| Field | Type | Description |
|-------|------|-------------|
| `price` | number | Buy price |
| `lot` | number | Number of lots |
| `valid_from` | string | Start time (HH:MM) |
| `valid_until` | string | End time (HH:MM) |

### Take Profit / Cut Loss

| Field | Type | Description |
|-------|------|-------------|
| `price` | number | Target price for TP/CL |

---

## Notes

- All times are in WIB (UTC+7)
- Robot pulls tasks once before market opens
- Tasks outside `valid_from`/`valid_until` window are ignored
