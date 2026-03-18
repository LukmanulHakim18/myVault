---
title: Dynamic Platform Fee - Rule Engine Implementation
tags:
  - design-doc
  - mrg
  - dynamic-platform-fee
  - implementation
  - rule-engine
  - golang
created: '2026-02-18'
status: implemented
team: MRG
updated: '2026-03-05'
---
# Dynamic Platform Fee — Rule Engine Implementation Guide

**Updated:** 2026-03-05  
**Status:** Implemented  
**Owner:** MRG Architecture Team

---

## 1. Overview

Guide ini menjelaskan implementasi rule evaluation engine untuk Dynamic Platform Fee (DPF) sesuai dengan kode aktual di repository `dpf_super_claude_tdd`.

**Key Topics:**
- Data sources (header vs external call)
- Rule evaluation via JSONB condition
- Calculator formula
- Caching strategy

---

## 2. Data Sources

### 2.1. Dari Header `User-Info` (via aphrodite/metadata)

DPF **tidak query** Loyalty Service. Semua data user context diambil dari header:

```go
md := commonMetadata.GetMetaDataFromContext(ctx)
bbid := md.GetBBID()
userInfo := md.GetUserInfo()
isForeign := userInfo.IsForeign
loyaltyTier := userInfo.LoyaltyTier  // sudah uppercase
```

**Header Format:**
```
User-Info: {"internal_id":"BB123456","is_foreign":true,"loyalty_tier":"ELITE"}
```

### 2.2. Dari Order Orchestrator

Hanya `total_completed_order` yang di-fetch dari external service:

```go
orderCount, err := uc.Repo.OrderOrchestrator.GetOrderCounter(ctx, bbid)
if err != nil {
    // Fail-safe: treat as new user
    orderCount = 0
}
```

### 2.3. Dari Request Body

```go
paymentType := req.PaymentMethod.Type         // cc, ECV, trip_voucher, cc_cp
cardPrincipal := req.PaymentMethod.Principal  // VISA, MASTERCARD, AMEX
cardCountry := req.PaymentMethod.CountryCode  // ID, US, SG
urgencyType := req.UrgencyType                // immediate, advance
```

---

## 3. Corporate Detection

Corporate user ditentukan **per-transaksi** berdasarkan `payment_method.type`:

```go
var corporatePayments = map[string]bool{
    "ECV":          true,
    "trip_voucher": true,
    "cc_cp":        true,
}

isCorporate := corporatePayments[paymentType]
```

---

## 4. User Category untuk Cache Key

`userCategory` mencerminkan modifier yang **benar-benar berlaku**:

```go
userCategory := "retail"
if isForeign && orderCount < foreignThreshold {
    userCategory = "foreign"
}
if isCorporate {
    userCategory = "corporate"
}
```

> **Note:** User foreign yang sudah loyal (order >= threshold) tidak di-cache sebagai "foreign" karena surcharge-nya tidak berlaku.

---

## 5. Rule Evaluation

### 5.1. Condition Value Format (JSONB)

```json
{ "operator": "==|!=|>=|<=|>|<|in", "value": <any> }
```

### 5.2. Seed Rules

| parameter | condition_key | condition_value | adjustment |
|---|---|---|---|
| is_foreign | is_foreign_true | `{"operator":"==","value":true}` | +1000 |
| payment_type | payment_type_corporate | `{"operator":"in","value":["ECV","trip_voucher","cc_cp"]}` | +500 |
| card_country | card_country_foreign | `{"operator":"!=","value":"ID"}` | +1000 |
| card_principal | card_principal_amex | `{"operator":"==","value":"AMEX"}` | +500 |
| urgency_type | urgency_immediate | `{"operator":"==","value":"immediate"}` | +250 |
| urgency_type | urgency_advance | `{"operator":"==","value":"advance"}` | +300 |

### 5.3. Evaluator Flow

```go
actuals := map[string]interface{}{
    "is_foreign":     isForeign,
    "payment_type":   paymentType,
    "card_country":   cardCountry,
    "card_principal": cardPrincipal,
    "urgency_type":   req.UrgencyType,
}

for _, rule := range rules {
    cond, _ := evaluator.ParseCondition(rule.ConditionValue)
    actualVal := actuals[rule.Parameter]
    
    if evaluator.Evaluate(cond, actualVal) {
        switch rule.Parameter {
        case "is_foreign":
            calcInput.ForeignAdd += rule.AdjustmentAmount
        case "payment_type":
            calcInput.CorporateAdd += rule.AdjustmentAmount
        case "card_country":
            calcInput.BinCountryAdd += rule.AdjustmentAmount
        case "card_principal":
            calcInput.CardPrincipalAdd += rule.AdjustmentAmount
        case "urgency_type":
            calcInput.UrgencyAdd += rule.AdjustmentAmount
        }
    }
}
```

---

## 6. Loyalty Discount

Loyalty discount diambil dari `dpf_global_config`, bukan dari rules:

```go
if loyaltyTier != "" {
    discountKey := "loyalty_discount_" + strings.ToLower(loyaltyTier)
    if discountCfg, ok := cfg[discountKey]; ok {
        calcInput.LoyaltyDiscount = discountCfg.IntVal()
    }
}
```

**Config Values:**

| config_key | value |
|---|---|
| loyalty_discount_explorer | 500 |
| loyalty_discount_traveler | 1000 |
| loyalty_discount_voyager | 1500 |
| loyalty_discount_elite | 2000 |

---

## 7. Calculator Formula

```go
type Input struct {
    OrderCount       int
    NewUserThreshold int
    BasePlatformFee  int
    CapPlatformFee   int
    IsForeign        bool
    ForeignThreshold int
    IsCorporate      bool
    ForeignAdd       int
    CorporateAdd     int
    BinCountryAdd    int
    CardPrincipalAdd int
    UrgencyAdd       int
    LoyaltyDiscount  int
}

func Calculate(input Input) Result {
    // 1. New user guard
    if input.OrderCount <= input.NewUserThreshold {
        return Result{FinalPlatformFee: 0, IsNewUser: true}
    }

    // 2. Calculate raw
    raw := input.BasePlatformFee
    
    // Foreign surcharge (hanya jika order >= threshold)
    if input.IsForeign && input.OrderCount >= input.ForeignThreshold {
        raw += input.ForeignAdd
    }
    
    raw += input.CorporateAdd
    raw += input.BinCountryAdd
    raw += input.CardPrincipalAdd
    raw += input.UrgencyAdd
    raw -= input.LoyaltyDiscount

    // 3. Round up to nearest 1000
    rounded := int(math.Ceil(float64(raw)/1000.0)) * 1000

    // 4. Apply cap
    final := rounded
    capApplied := false
    if final > input.CapPlatformFee {
        final = input.CapPlatformFee
        capApplied = true
    }

    // 5. Apply floor
    if final < input.BasePlatformFee {
        final = input.BasePlatformFee
    }

    return Result{
        BasePlatformFee:  input.BasePlatformFee,
        CapPlatformFee:   input.CapPlatformFee,
        TotalAdjustment:  rounded - input.BasePlatformFee,
        TotalPlatformFee: rounded,
        FinalPlatformFee: final,
        CapApplied:       capApplied,
    }
}
```

---

## 8. Calculation Example

```
Input:
- Base (BLUE/RIDE/ARGO)    : 4,000
- is_foreign=true, order=10, threshold=6

Modifiers Applied:
- Foreign surcharge        : +1,000
- Corporate (ECV)          : +  500
- BIN country (US)         : +1,000
- AMEX                     : +  500
- Urgency immediate        : +  250
- Loyalty ELITE            : -2,000

Calculation:
- Raw = 4000 + 1000 + 500 + 1000 + 500 + 250 - 2000 = 5,250
- Round up (×1000) = 6,000
- Cap check (50,000) = ✅
- Floor check (4,000) = ✅
- Final = 6,000
```

---

## 9. Caching

### 9.1. Cache Key Format

```
dpf:fee:{product|ALL}:{service|ALL}:{pricing|ALL}:{user_category}:{loyalty_tier}:{payment_type}:{urgency_type}
```

### 9.2. Cache Key Builder

```go
ck := cachekey.Build(cachekey.Params{
    Product:      product,        // atau "ALL" jika kosong
    Service:      service,        // atau "ALL" jika kosong
    Pricing:      pricing,        // atau "ALL" jika kosong
    UserCategory: userCategory,   // foreign/retail/corporate
    LoyaltyTier:  loyaltyTier,    // EXPLORER/TRAVELER/VOYAGER/ELITE
    PaymentType:  paymentType,    // cc/debit/ECV/trip_voucher/cc_cp
    UrgencyType:  urgencyType,    // immediate/advance
})
```

### 9.3. TTL

```go
ttl := time.Duration(cfg["cache_ttl_seconds"].IntVal()) * time.Second  // default 3600s
```

---

## 10. Package Structure

```
pkg/
├── calculator/
│   ├── calculator.go      ← Pure formula engine (no I/O)
│   └── calculator_test.go
├── evaluator/
│   ├── evaluator.go       ← JSONB condition evaluator
│   └── evaluator_test.go
└── cachekey/
    ├── cachekey.go        ← Cache key builder
    └── cachekey_test.go
```

Ketiga package ini adalah **pure functions** — tidak ada I/O, mudah di-test tanpa mock.

---

## 11. Related Documents

- [[RFC-001-dynamic-platform-fee-service]]
- [[RFC-002-order-orchestrator-counter]]
- [[test-scenarios]]

---

**Version:** 3.0  
**Last Review:** 2026-03-05
