---
title: Dynamic Platform Fee - Test Scenarios
tags:
  - test
  - mrg
  - dynamic-platform-fee
  - qa
created: '2026-02-18'
status: draft
team: MRG
updated: '2026-03-02'
---
# Dynamic Platform Fee — Test Scenarios

**Updated:** 2026-03-02  
**Status:** Draft  
**Owner:** MRG Architecture Team

---

## 1. Overview

Dokumen ini berisi test scenarios untuk validasi implementasi Dynamic Platform Fee (DPF).

---

## 2. Test Configuration

### 2.1. Service Type Config

| service_type | base_platform_fee | cap_platform_fee |
|---|---|---|
| RIDE_BLUE_ARGO | 4.000 | 50.000 |
| RIDE_BLUE_FIXED | 4.000 | 50.000 |
| RIDE_SILVER_ARGO | 7.000 | 70.000 |
| RIDE_GOLDEN_NOW | 5.000 | 60.000 |
| RIDE_GOLDEN_AT | 5.000 | 60.000 |
| RENT_BLUE_HOURLY | 4.000 | 50.000 |
| RENT_SILVER_HOURLY | 7.000 | 70.000 |
| DELIVERY_BLUE_KIRIM | 0 | 40.000 |
| SHUTTLE_CITITRANS_REGULAR | 4.000 | 50.000 |

### 2.2. Global Config

| config_key | value |
|---|---|
| new_user_order_threshold | 1 |
| foreign_order_threshold | 6 |
| foreign_fee_increment | 5.000 |
| corporate_surcharge | 1.000 |
| amex_surcharge | 500 |
| bin_country_surcharge | 1.000 |
| loyalty_discount_explorer | 500 |
| loyalty_discount_traveler | 1.000 |
| loyalty_discount_voyager | 1.500 |
| loyalty_discount_elite | 2.000 |

---

## 3. New User Scenarios

### 3.1. TC-NEW-001: New User (0 orders)

**Input:**
```json
{
    "bbid": "BB000001",
    "is_foreign": false,
    "total_completed_order": 0,
    "service_type": "RIDE_BLUE_ARGO",
    "payment_method": {"type": "cash"}
}
```

**Expected:**
```json
{
    "final_platform_fee": 0,
    "breakdown": [
        {"modifier": "new_user_exemption", "amount": 0}
    ],
    "metadata": {"is_new_user": true}
}
```

### 3.2. TC-NEW-002: New User (1 order)

**Input:**
```json
{
    "bbid": "BB000002",
    "is_foreign": true,
    "total_completed_order": 1,
    "service_type": "RIDE_SILVER_ARGO",
    "payment_method": {"type": "cc", "principal": "AMEX", "country_code": "US"}
}
```

**Expected:**
```json
{
    "final_platform_fee": 0,
    "breakdown": [
        {"modifier": "new_user_exemption", "amount": 0}
    ],
    "metadata": {"is_new_user": true}
}
```

> **Note:** New user exemption overrides ALL modifiers termasuk foreign, AMEX, dll.

### 3.3. TC-NEW-003: Not New User (2 orders)

**Input:**
```json
{
    "bbid": "BB000003",
    "is_foreign": false,
    "total_completed_order": 2,
    "service_type": "RIDE_BLUE_ARGO",
    "payment_method": {"type": "cash"}
}
```

**Expected:**
```json
{
    "base_platform_fee": 4000,
    "final_platform_fee": 4000,
    "metadata": {"is_new_user": false}
}
```

---

## 4. Foreign User Scenarios

### 4.1. TC-FRG-001: Foreign User Below Threshold

**Input:**
```json
{
    "bbid": "BB000004",
    "is_foreign": true,
    "total_completed_order": 5,
    "service_type": "RIDE_BLUE_ARGO",
    "payment_method": {"type": "cash"}
}
```

**Expected:**
```json
{
    "base_platform_fee": 4000,
    "final_platform_fee": 4000,
    "breakdown": [],
    "metadata": {"is_foreign": true, "foreign_threshold": 6}
}
```

> **Note:** `order (5) < threshold (6)` → no foreign surcharge

### 4.2. TC-FRG-002: Foreign User At Threshold

**Input:**
```json
{
    "bbid": "BB000005",
    "is_foreign": true,
    "total_completed_order": 6,
    "service_type": "RIDE_BLUE_ARGO",
    "payment_method": {"type": "cash"}
}
```

**Expected:**
```json
{
    "base_platform_fee": 4000,
    "raw_fee": 9000,
    "rounded_fee": 9000,
    "final_platform_fee": 9000,
    "breakdown": [
        {"modifier": "foreign_surcharge", "amount": 5000}
    ]
}
```

### 4.3. TC-FRG-003: Foreign User Above Threshold

**Input:**
```json
{
    "bbid": "BB000006",
    "is_foreign": true,
    "total_completed_order": 10,
    "service_type": "RIDE_BLUE_ARGO",
    "payment_method": {"type": "cash"}
}
```

**Expected:**
```json
{
    "final_platform_fee": 9000,
    "breakdown": [
        {"modifier": "foreign_surcharge", "amount": 5000}
    ]
}
```

### 4.4. TC-FRG-004: Local User (not foreign)

**Input:**
```json
{
    "bbid": "BB000007",
    "is_foreign": false,
    "total_completed_order": 100,
    "service_type": "RIDE_BLUE_ARGO",
    "payment_method": {"type": "cash"}
}
```

**Expected:**
```json
{
    "final_platform_fee": 4000,
    "breakdown": [],
    "metadata": {"is_foreign": false}
}
```

---

## 5. Corporate User Scenarios

### 5.1. TC-CORP-001: Corporate with ECV

**Input:**
```json
{
    "bbid": "BB000008",
    "is_foreign": false,
    "total_completed_order": 10,
    "service_type": "RIDE_BLUE_ARGO",
    "payment_method": {"type": "ECV"}
}
```

**Expected:**
```json
{
    "base_platform_fee": 4000,
    "raw_fee": 5000,
    "final_platform_fee": 5000,
    "breakdown": [
        {"modifier": "corporate_surcharge", "amount": 1000}
    ],
    "metadata": {"is_corporate": true}
}
```

### 5.2. TC-CORP-002: Corporate with Trip Voucher

**Input:**
```json
{
    "bbid": "BB000009",
    "is_foreign": false,
    "total_completed_order": 10,
    "service_type": "RIDE_BLUE_ARGO",
    "payment_method": {"type": "trip_voucher"}
}
```

**Expected:**
```json
{
    "final_platform_fee": 5000,
    "metadata": {"is_corporate": true}
}
```

### 5.3. TC-CORP-003: Corporate with CC_CP

**Input:**
```json
{
    "bbid": "BB000010",
    "is_foreign": false,
    "total_completed_order": 10,
    "service_type": "RIDE_BLUE_ARGO",
    "payment_method": {"type": "cc_cp"}
}
```

**Expected:**
```json
{
    "final_platform_fee": 5000,
    "metadata": {"is_corporate": true}
}
```

### 5.4. TC-CORP-004: Retail with CC (not corporate)

**Input:**
```json
{
    "bbid": "BB000011",
    "is_foreign": false,
    "total_completed_order": 10,
    "service_type": "RIDE_BLUE_ARGO",
    "payment_method": {"type": "cc", "principal": "VISA", "country_code": "ID"}
}
```

**Expected:**
```json
{
    "final_platform_fee": 4000,
    "metadata": {"is_corporate": false}
}
```

---

## 6. Loyalty Tier Scenarios

### 6.1. TC-LTY-001: Explorer Tier

**Input:**
```json
{
    "bbid": "BB000012",
    "is_foreign": false,
    "total_completed_order": 10,
    "loyalty_tier": "EXPLORER",
    "service_type": "RIDE_BLUE_ARGO",
    "payment_method": {"type": "cash"}
}
```

**Expected:**
```json
{
    "base_platform_fee": 4000,
    "raw_fee": 3500,
    "rounded_fee": 4000,
    "final_platform_fee": 4000,
    "breakdown": [
        {"modifier": "loyalty_discount", "label": "EXPLORER Discount", "amount": -500},
        {"modifier": "rounding", "amount": 500}
    ]
}
```

> **Note:** Floor applied — final tidak boleh kurang dari base

### 6.2. TC-LTY-002: Traveler Tier

**Input:**
```json
{
    "bbid": "BB000013",
    "loyalty_tier": "TRAVELER",
    "service_type": "RIDE_BLUE_ARGO"
}
```

**Expected:**
```json
{
    "raw_fee": 3000,
    "final_platform_fee": 4000,
    "breakdown": [
        {"modifier": "loyalty_discount", "amount": -1000}
    ]
}
```

### 6.3. TC-LTY-003: Voyager Tier

**Input:**
```json
{
    "bbid": "BB000014",
    "loyalty_tier": "VOYAGER",
    "service_type": "RIDE_BLUE_ARGO"
}
```

**Expected:**
```json
{
    "raw_fee": 2500,
    "final_platform_fee": 4000,
    "breakdown": [
        {"modifier": "loyalty_discount", "amount": -1500}
    ]
}
```

### 6.4. TC-LTY-004: Elite Tier

**Input:**
```json
{
    "bbid": "BB000015",
    "loyalty_tier": "ELITE",
    "service_type": "RIDE_BLUE_ARGO"
}
```

**Expected:**
```json
{
    "raw_fee": 2000,
    "final_platform_fee": 4000,
    "breakdown": [
        {"modifier": "loyalty_discount", "amount": -2000}
    ]
}
```

---

## 7. Payment Scenarios

### 7.1. TC-PAY-001: Foreign Card (BIN Country)

**Input:**
```json
{
    "bbid": "BB000016",
    "is_foreign": false,
    "total_completed_order": 10,
    "service_type": "RIDE_BLUE_ARGO",
    "payment_method": {"type": "cc", "principal": "VISA", "country_code": "US"}
}
```

**Expected:**
```json
{
    "raw_fee": 5000,
    "final_platform_fee": 5000,
    "breakdown": [
        {"modifier": "bin_country", "amount": 1000}
    ]
}
```

### 7.2. TC-PAY-002: AMEX Card

**Input:**
```json
{
    "bbid": "BB000017",
    "is_foreign": false,
    "total_completed_order": 10,
    "service_type": "RIDE_BLUE_ARGO",
    "payment_method": {"type": "cc", "principal": "AMEX", "country_code": "ID"}
}
```

**Expected:**
```json
{
    "raw_fee": 4500,
    "rounded_fee": 5000,
    "final_platform_fee": 5000,
    "breakdown": [
        {"modifier": "card_principal", "amount": 500},
        {"modifier": "rounding", "amount": 500}
    ]
}
```

### 7.3. TC-PAY-003: Double Surcharge (Foreign + AMEX)

**Input:**
```json
{
    "bbid": "BB000018",
    "is_foreign": true,
    "total_completed_order": 10,
    "service_type": "RIDE_BLUE_ARGO",
    "payment_method": {"type": "cc", "principal": "AMEX", "country_code": "US"}
}
```

**Expected:**
```json
{
    "base_platform_fee": 4000,
    "raw_fee": 11500,
    "rounded_fee": 12000,
    "final_platform_fee": 12000,
    "breakdown": [
        {"modifier": "foreign_surcharge", "amount": 5000},
        {"modifier": "bin_country", "amount": 1000},
        {"modifier": "card_principal", "amount": 500},
        {"modifier": "rounding", "amount": 500}
    ]
}
```

---

## 8. Rounding Scenarios

### 8.1. TC-RND-001: Rounding Up

**Input:**
```json
{
    "base_platform_fee": 4000,
    "adjustments": [
        {"modifier": "card_principal", "amount": 500}
    ]
}
```

**Expected:**
```
raw = 4000 + 500 = 4500
rounded = ceil(4500 / 1000) * 1000 = 5000
rounding_adjustment = 500
```

### 8.2. TC-RND-002: No Rounding Needed

**Input:**
```json
{
    "base_platform_fee": 4000,
    "adjustments": [
        {"modifier": "foreign_surcharge", "amount": 5000}
    ]
}
```

**Expected:**
```
raw = 4000 + 5000 = 9000
rounded = 9000
rounding_adjustment = 0
```

### 8.3. TC-RND-003: Rounding Before Cap

**Input:**
```json
{
    "base_platform_fee": 4000,
    "cap_platform_fee": 50000,
    "raw_fee": 49500
}
```

**Expected:**
```
raw = 49500
rounded = ceil(49500 / 1000) * 1000 = 50000
final = min(50000, 50000) = 50000 ✅
```

---

## 9. Cap & Floor Scenarios

### 9.1. TC-CAP-001: Cap Applied

**Input:**
```json
{
    "bbid": "BB000019",
    "is_foreign": true,
    "total_completed_order": 100,
    "service_type": "RIDE_BLUE_ARGO",
    "payment_method": {"type": "cc", "principal": "AMEX", "country_code": "US"},
    "many_surcharges": "simulate cap exceeded"
}
```

**Expected (if raw > 50000):**
```json
{
    "final_platform_fee": 50000,
    "cap_applied": true
}
```

### 9.2. TC-FLR-001: Floor Applied

**Input:**
```json
{
    "bbid": "BB000020",
    "is_foreign": false,
    "total_completed_order": 10,
    "loyalty_tier": "ELITE",
    "service_type": "RIDE_BLUE_ARGO",
    "payment_method": {"type": "cash"}
}
```

**Expected:**
```json
{
    "base_platform_fee": 4000,
    "raw_fee": 2000,
    "final_platform_fee": 4000,
    "breakdown": [
        {"modifier": "loyalty_discount", "amount": -2000}
    ]
}
```

> **Note:** Floor = base_platform_fee, sehingga final = 4000 bukan 2000

---

## 10. Service Type Scenarios

### 10.1. TC-SVC-001: Silverbird (Higher Base)

**Input:**
```json
{
    "service_type": "RIDE_SILVER_ARGO",
    "base": 7000,
    "cap": 70000
}
```

### 10.2. TC-SVC-002: Delivery (Zero Base)

**Input:**
```json
{
    "service_type": "DELIVERY_BLUE_KIRIM",
    "base": 0,
    "cap": 40000
}
```

**Expected:** Base = 0, so new user also gets 0

### 10.3. TC-SVC-003: Unknown Service Type (Fallback)

**Input:**
```json
{
    "service_type": "RIDE_UNKNOWN_TYPE"
}
```

**Expected:**
```json
{
    "base_platform_fee": 4000,
    "note": "Fallback to default"
}
```

---

## 11. Stacking Scenarios

### 11.1. TC-STK-001: Full Stack (All Modifiers)

**Input:**
```json
{
    "bbid": "BB000021",
    "is_foreign": true,
    "total_completed_order": 10,
    "loyalty_tier": "ELITE",
    "service_type": "RIDE_BLUE_ARGO",
    "payment_method": {"type": "ECV", "principal": "AMEX", "country_code": "US"}
}
```

**Calculation:**
```
Base PF (RIDE_BLUE_ARGO)  :  4.000
Foreign surcharge         : +5.000
Corporate surcharge       : +1.000
BIN country surcharge     : +1.000
AMEX surcharge            :   +500
Loyalty ELITE             : -2.000
─────────────────────────────────
Raw                       :  9.500
Round up (nearest 1000)   : 10.000
Cap check (50.000)        :    ✅
─────────────────────────────────
Final PF                  : 10.000
```

**Expected:**
```json
{
    "base_platform_fee": 4000,
    "raw_fee": 9500,
    "rounded_fee": 10000,
    "final_platform_fee": 10000,
    "breakdown": [
        {"modifier": "foreign_surcharge", "amount": 5000},
        {"modifier": "corporate_surcharge", "amount": 1000},
        {"modifier": "bin_country", "amount": 1000},
        {"modifier": "card_principal", "amount": 500},
        {"modifier": "loyalty_discount", "amount": -2000},
        {"modifier": "rounding", "amount": 500}
    ]
}
```

---

## 12. Fallback Scenarios

### 12.1. TC-FB-001: Order Orchestrator Timeout

**Condition:** Order Orchestrator tidak respond

**Expected:**
```json
{
    "total_completed_order": 0,
    "is_new_user": true,
    "final_platform_fee": 0,
    "note": "Fallback to new user"
}
```

### 12.2. TC-FB-002: Loyalty Service Timeout

**Condition:** Loyalty Service tidak respond

**Expected:**
```json
{
    "loyalty_tier": "",
    "loyalty_discount": 0,
    "note": "Skip loyalty adjustment"
}
```

### 12.3. TC-FB-003: Service Type Not Found

**Condition:** Service type tidak ada di dpf_service_config

**Expected:**
```json
{
    "base_platform_fee": 4000,
    "cap_platform_fee": 50000,
    "note": "Fallback to default"
}
```

---

## 13. Related Documents

- [[RFC-001-dynamic-platform-fee-service]]
- [[rule-engine-implementation]]
