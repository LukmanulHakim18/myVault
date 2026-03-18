---
title: UPG Contract Alignment Summary
tags:
  - design-doc
  - mrg
  - dynamic-platform-fee
  - upg
  - alignment
created: '2026-02-18'
status: final
team: MRG
updated: '2026-03-02'
---
# UPG Contract Alignment Summary

**Updated:** 2026-03-02  
**Status:** Alignment Completed  
**Owner:** MRG Architecture Team

---

## 1. UPG Actual Response Contract

**Endpoint:** `GET /payment/single/{identifier}`

**Actual Response Fields:**
```json
{
    "data": {
        "type": "cc",
        "principal": "VISA",
        "country_code": "ID",
        "card_type": "DEBIT",
        "cc_type": "visa",
        "category": "credit_card",
        "display": "46169948*****8440",
        "status": "inactive"
    }
}
```

---

## 2. Field Name Standardization

### ✅ Adopted from UPG (Standardized)

| DPF Field | UPG Field | Type | Values |
|-----------|-----------|------|--------|
| `type` | `type` | string | `"cc"`, `"cash"`, `"ewallet"`, `"ECV"`, `"trip_voucher"`, `"cc_cp"` |
| `principal` | `principal` | string | `"VISA"`, `"MASTERCARD"`, `"AMEX"` |
| `country_code` | `country_code` | string | `"ID"`, `"US"`, `"SG"`, etc |
| `card_type` | `card_type` | string | `"DEBIT"`, `"CREDIT"` |

---

## 3. Payment Type Categories

### 3.1. Corporate Payment Types

DPF mendeteksi corporate user berdasarkan `payment_method.type`:

| Payment Type | Category | Surcharge Applied |
|--------------|----------|-------------------|
| `ECV` | Corporate | ✅ Yes |
| `trip_voucher` | Corporate | ✅ Yes |
| `cc_cp` | Corporate | ✅ Yes |
| `cc` | Retail | ❌ No |
| `cash` | Retail | ❌ No |
| `ewallet` | Retail | ❌ No |

### 3.2. Card-based Surcharges

| Condition | Surcharge |
|-----------|-----------|
| `country_code != "ID"` | BIN Country Surcharge |
| `principal == "AMEX"` | AMEX Surcharge |

---

## 4. Go Struct Definition

```go
type PaymentMethod struct {
    Type        string `json:"type"`         // "cc", "cash", "ewallet", "ECV", "trip_voucher", "cc_cp"
    Principal   string `json:"principal"`    // "VISA", "MASTERCARD", "AMEX"
    CountryCode string `json:"country_code"` // "ID", "US", "SG"
    CardType    string `json:"card_type"`    // "DEBIT", "CREDIT"
}
```

---

## 5. DPF Request Example

```json
{
    "service_type": "RIDE_BLUE_ARGO",
    "urgency_type": "immediate",
    "payment_method": {
        "type": "cc",
        "principal": "AMEX",
        "country_code": "US"
    }
}
```

---

## 6. Evaluation Logic

### 6.1. Corporate Detection

```go
var corporatePaymentTypes = map[string]bool{
    "ECV":          true,
    "trip_voucher": true,
    "cc_cp":        true,
}

func isCorporatePayment(paymentType string) bool {
    return corporatePaymentTypes[paymentType]
}
```

### 6.2. BIN Country Check

```go
func (s *DPFService) calculateBINCountrySurcharge(feeCtx *FeeContext) int {
    if feeCtx.PaymentMethod.Type != "cc" {
        return 0
    }
    if strings.ToUpper(feeCtx.PaymentMethod.CountryCode) == "ID" {
        return 0
    }
    return s.config.GetInt("bin_country_surcharge")
}
```

### 6.3. AMEX Check

```go
func (s *DPFService) calculateAMEXSurcharge(feeCtx *FeeContext) int {
    if feeCtx.PaymentMethod.Type != "cc" {
        return 0
    }
    if strings.ToUpper(feeCtx.PaymentMethod.Principal) != "AMEX" {
        return 0
    }
    return s.config.GetInt("amex_surcharge")
}
```

---

## 7. Double Surcharge Scenarios

| Scenario | is_foreign | country_code | principal | Surcharges Applied |
|----------|------------|--------------|-----------|-------------------|
| Foreign + Local Card + VISA | true | ID | VISA | Foreign only |
| Foreign + Foreign Card + VISA | true | US | VISA | Foreign + BIN |
| Foreign + Foreign Card + AMEX | true | US | AMEX | Foreign + BIN + AMEX |
| Local + Foreign Card + AMEX | false | US | AMEX | BIN + AMEX |
| Corporate + ECV | - | - | - | Corporate |

---

## 8. UPG Callback Enhancement (RFC-003)

Untuk deteksi foreign user via add payment card, UPG callback ke Webhook Forwarder harus include `country_code`:

```json
{
    "event": "card_registered",
    "data": {
        "bbid": "BB123456",
        "principal": "VISA",
        "country_code": "US",
        "card_type": "CREDIT"
    }
}
```

**Flow:**
1. User add CC via MyBB
2. Payment Provider callback ke UPG
3. UPG forward ke Webhook Forwarder (include `country_code`)
4. Webhook Forwarder check: `country_code != "ID"` → call User Service `PUT /v1/user/{bbid}/foreign-flag`

---

## 9. Related Documents

- [[RFC-001-dynamic-platform-fee-service]]
- [[RFC-003-user-service-foreign-flag]]
- [[rule-engine-implementation]]
- [[test-scenarios]]

---

**Version:** 2.0  
**Alignment Date:** 2026-03-02
