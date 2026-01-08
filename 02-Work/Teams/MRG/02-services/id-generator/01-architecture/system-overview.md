# ID Generator System - Overview

## Project Information

**Project:** Two-Level ID Generation System for Bluebird Transportation Platform
**Status:** Draft (Awaiting Approval)
**Created:** December 29, 2025
**Platform:** Web Reservasi & MyBB Rent Revamp
**Team:** Platform Engineering Team

## Executive Summary

Sistem Two-Level ID Generation yang menggantikan sequential database ID dengan ID yang user-friendly, secure, dan analytics-ready untuk platform transportasi Bluebird Group.

### Key Components

1. **Booking ID** (Parent Level) - Format: `CCT-TTTTTHHHHHHHHH` (18 chars)
2. **Order ID** (Child Level) - Format: `CCPP-TTTTTHHHHHHHHH` (18 chars)

## Problem Statement

Sistem lama menggunakan sequential database ID yang:
- ❌ **Tidak secure**: Competitor bisa estimate order volume
- ❌ **Tidak user-friendly**: Sulit dikomunikasikan via telepon
- ❌ **Tidak informative**: Tidak ada context tentang channel/product
- ❌ **Tidak scalable**: Single-level tracking tidak mendukung multi-item bookings

## Solution

### Two-Level System Architecture

```
┌─────────────────────────────────────┐
│         Booking Level               │
│  CCT-TTTTTHHHHHHHHH (18 chars)     │
│  ├─ Channel Code (2 chars)          │
│  ├─ Booking Type (1 char)           │
│  ├─ Time Hash (5 chars)             │
│  └─ DB Hash (8 chars)               │
└─────────────────────────────────────┘
              │
              ├──────────┬──────────┐
              ▼          ▼          ▼
    ┌──────────────┐ ┌──────────────┐
    │ Order Level  │ │ Order Level  │
    │ CCPP-TTTTT.. │ │ CCPP-TTTTT.. │
    │ (18 chars)   │ │ (18 chars)   │
    └──────────────┘ └──────────────┘
```

## Format Specifications

### Booking ID Format
```
MYS-NQBMEK7P2QF01
││└─┘└──┬──┘└───┬────┘
││ │    │       └─ DB Hash (8 chars)
││ │    └─ Time Hash (5 chars base36)
││ └─ Booking Type: S=Single, M=Multi
│└─ Channel Code: MY=MyBB, WB=Web, etc
```

### Order ID Format
```
MYSN-NQBMEK7P2QF01
││└┬┘└──┬──┘└───┬────┘
││ │    │       └─ DB Hash (8 chars)
││ │    └─ Time Hash (5 chars base36)
││ └─ Product Code: SN=Silverbird Rent
│└─ Channel Code: MY=MyBB
```

## Code Mappings

### Channel Codes (6 channels)
| Code | Channel | Description |
|------|---------|-------------|
| MY | MyBB | MyBB Mobile Application |
| WB | Web | Web Booking Platform |
| CR | Corporate | Corporate Portal |
| AP | API | Third-party API Partners |
| CT | Call Center | Call Center Booking |
| WA | WhatsApp | WhatsApp Booking Bot |

### Booking Types (2 types)
| Code | Type | Items | Description |
|------|------|-------|-------------|
| S | Single | 1 | Single order booking |
| M | Multi | 2+ | Multi-item booking (packages) |

**Total Booking Prefixes**: 6 × 2 = **12 combinations**

### Vehicle Types (4 types)
| Code | Vehicle | Description |
|------|---------|-------------|
| B | BlueBird | Standard taxi service |
| S | Silverbird | Premium taxi service |
| G | Goldenbird | Luxury taxi service |
| C | Citytrans | Shuttle bus service |

### Service Types (7 types)
| Code | Service | Description |
|------|---------|-------------|
| A | Airport | Airport transfer service |
| D | Delivery | Package delivery |
| N | Rent (general) | General rental service |
| R | Ride (P2P) | Point-to-point ride |
| H | Hourly | Hourly rental |
| L | Daily | Daily rental |
| S | Shuttle | Shuttle bus service |

**Total Product Combinations**: 4 × 7 = **28 combinations**
**Total Order Prefixes**: 6 × 28 = **168 combinations**

## Key Features

### 1. User-Friendly
- ✅ Easy to communicate via phone: "M-Y-S-N dash N-Q-B-M-E..."
- ✅ Human-readable format with semantic meaning
- ✅ Consistent 18-character length

### 2. Security & Privacy
- ✅ Database ID obfuscated with XOR encryption
- ✅ Competitor cannot estimate order volume
- ✅ Reversible encoding for internal use

### 3. Analytics-Ready
- ✅ Channel embedded in ID: `WHERE exposed_id LIKE 'MY%'`
- ✅ Product tracking without JOIN: `SUBSTRING(exposed_id, 1, 4)`
- ✅ Time-based partitioning support

### 4. High Performance
- ✅ Sub-millisecond generation (<0.3ms)
- ✅ 10,000+ TPS per instance
- ✅ Stateless architecture for horizontal scaling

## ID Recognition Logic

**Simple Rule**: Count characters before dash (-)

```go
prefixLen := len(strings.Split(exposedID, "-")[0])

if prefixLen == 3 {
    return "booking"  // CCT-...
} else if prefixLen == 4 {
    return "order"    // CCPP-...
}
```

## Hash Generation

### Time Hash (5 characters)
```go
// Format: YYMMDDHH in base36
// Example: 25122614 → "NQBME"
timestamp := time.Now()
combined := (year%100)*1000000 + month*10000 + day*100 + hour
timeHash := ToBase36(combined, 5)
```

**Coverage**: 
- Years: 2000-2099 (100 years)
- Precision: Per hour
- Collision window: 1 hour

### DB Hash (8 characters)
```go
// XOR obfuscation with secret key
obfuscated := dbID ^ HashSecret(secret)
dbHash := ToBase36(obfuscated, 8)
```

**Properties**:
- Space: 36^8 = 2.8 trillion combinations
- Security: XOR with secret salt
- Reversible: Can decode to database ID

## Implementation Status

### Web Reservasi
- ✅ Booking ID implemented
- ✅ Order ID implemented
- ✅ Database migration completed

### Rent Revamp MyBB
- ✅ Booking ID implemented
- ✅ Order ID implemented
- ✅ Multi-item booking support

## Performance Metrics

- **Generation Time**: <0.3ms average
- **Throughput**: 10,000+ IDs/second per instance
- **Database Index Size**: ~50% smaller than UUID
- **Collision Rate**: 0% (guaranteed by DB constraint)

## Tags

#architecture #id-generation #bluebird #transportation #system-design #web-reservasi #rent-revamp #mybb