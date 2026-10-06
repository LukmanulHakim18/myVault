---
title: "N2C Service - Business Flows"
type: flow-documentation
tags: [n2cservice, flow, payment, preauth, mrg]
parent: n2cservice
created: 2026-04-08
updated: 2026-04-08
---

# N2C Service - Business Flows

**Service**: [[README|N2C Service]]

---

## 1️⃣ CC Pre-auth Increment Flow

**Skenario**: Argo aktual melebihi jumlah pre-auth kartu kredit yang ada

```mermaid
sequenceDiagram
    participant Caller
    participant N2C
    participant OrderQuery
    participant TaxiOP as TaxiOrderProcessor
    participant PayProc as PaymentProcessor
    participant MsgBroker as MessageBroker

    Caller->>N2C: EligibilityByJobId(order_id, actual_argo, trip_status=CC)
    N2C->>TaxiOP: GetOrderByJobId / Redis cache
    TaxiOP-->>N2C: OrderDetail (payment_method=CC)

    N2C->>OrderQuery: GetOrderInfoByOrderId
    OrderQuery-->>N2C: is_eligible_preauth_increment

    alt Not eligible
        N2C-->>Caller: {is_sufficient: true, force_to_cash: false}
    end

    N2C->>TaxiOP: GetOrderPreAuth
    TaxiOP-->>N2C: OrderPreAuth (last_amount, is_preauth)

    alt Non-preauth card
        N2C-->>Caller: {is_sufficient: true, force_to_cash: false}
    end

    alt actual_argo <= last_amount
        N2C-->>Caller: {is_sufficient: true, force_to_cash: false}
    end

    N2C->>PayProc: PreAuthCancellation / CancelTx (cancel lama)
    alt Cancel gagal
        N2C->>TaxiOP: UpdateLockedFare(amount=0, force_to_cash=true)
        N2C->>MsgBroker: PushMessageForcedToCash
        N2C-->>Caller: {is_sufficient: false, force_to_cash: true}
    end

    N2C->>PayProc: RePreAuth(new_amount = last + cycle_charge)
    alt RePreAuth gagal / FAILED
        N2C->>TaxiOP: UpdateLockedFare(force_to_cash=true)
        N2C->>MsgBroker: PushMessageForcedToCash
        N2C-->>Caller: {is_sufficient: false, force_to_cash: true}
    end

    N2C->>TaxiOP: UpdateLockedFare(new_amount, is_success=true)
    N2C->>MsgBroker: PushMessageRePreAuth
    N2C-->>Caller: {is_sufficient: true, force_to_cash: false}
```

---

## 2️⃣ E-wallet Eligibility Flow

**Skenario**: Cek saldo e-wallet selama trip berlangsung

```mermaid
sequenceDiagram
    participant Caller
    participant N2C
    participant TaxiOP as TaxiOrderProcessor
    participant PayProc as PaymentProcessor
    participant Redis
    participant MsgBroker as MessageBroker

    Caller->>N2C: EligibilityByJobId(order_id, actual_argo, payment_method=ewallet)
    N2C->>TaxiOP: GetOrderByJobId / Redis cache
    TaxiOP-->>N2C: OrderDetail

    N2C->>Redis: GetBalanceNotif(order_id)
    Redis-->>N2C: BalanceNotif (last_balance, is_sufficient)

    N2C->>PayProc: GetEwalletBalance(user_id, provider)
    PayProc-->>N2C: balance

    Note over N2C: balance_from_service = balance + locked_fare

    alt balance_from_service naik (top-up terdeteksi)
        Note over N2C: isBalanceHasBeenTopUp = true
    end

    alt is_end_trip AND saldo tidak cukup
        N2C->>PayProc: CancelSyncTransaction
        N2C->>TaxiOP: UpdateLockedFare(force_to_cash=true)
        N2C->>MsgBroker: PushMessageForcedToCash
        N2C->>Redis: DeleteBalanceNotif
        N2C-->>Caller: {is_sufficient: false, force_to_cash: true}
    else mid-trip AND saldo tidak cukup
        N2C->>MsgBroker: PushMessageInsufficientForTopup
        N2C->>Redis: StoreBalanceNotif (count++)
        N2C-->>Caller: {is_sufficient: false, force_to_cash: false}
    else saldo cukup
        N2C-->>Caller: {is_sufficient: true, force_to_cash: false}
    end
```

---

## 📊 Flow Comparison Matrix

| Flow | Payment Type | Dependencies | End-trip Behavior |
|------|-------------|--------------|-------------------|
| **Fixed Fare** | Any | Redis (order cache) | Selalu eligible |
| **CC Pre-auth** | Credit Card | TaxiOP, OrderQuery, PayProc, Config, MsgBroker | Increment atau force-to-cash |
| **E-wallet** | GoPay, OVO, dll | TaxiOP, PayProc, Redis, MsgBroker | Force-to-cash jika kurang |
| **Cash** | Cash | Redis (order cache) | Selalu eligible |

---

## 🔒 Key Points

- **Order detail di-cache** di Redis untuk mengurangi load ke TaxiOrderProcessor
- **Balance notif di-cache** untuk mendeteksi apakah user sudah top-up sejak pengecekan terakhir
- **Legacy vs new preauth**: `CancelTx` untuk preauth lama (tanpa `LastPreAuthAt`), `PreAuthCancellation` untuk preauth baru
- **End-trip** selalu memicu evaluasi final — tidak ada retry, langsung force-to-cash jika gagal

---

#n2cservice #flow #payment #preauth #mrg

*Last Updated*: 2026-04-08
*Generated from*: Repository analysis — D:\code\go\mybb-ms\n2cservice
