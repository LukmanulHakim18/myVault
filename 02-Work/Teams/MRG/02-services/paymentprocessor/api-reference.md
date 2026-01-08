---
tags:
  - mrg
  - api-reference
  - payment
  - grpc
  - rest
type: api-documentation
title: Payment Processor API Reference
created: '2025-01-07'
updated: '2025-01-07'
---
# Payment Processor - API Reference

Dokumentasi lengkap untuk API endpoints Payment Processor service.

---

## 📡 Service Information

**Service Name**: `PaymentProcessor`  
**Package**: `paymentprocessor`  
**Proto File**: `contract/payment_processor.proto`  
**gRPC Port**: `6007`  
**REST Port**: `8007`

---

## 🔑 Core Transaction APIs

### CompleteTx

Complete transaction setelah trip selesai.

**Method**: `CompleteTx`  
**Interceptors**: `ValidateInput`

**Request Fields**:
```protobuf
message CompleteTxRequest {
  string payment_method = 1;      // Payment method type
  int32 order_id = 2;             // Order ID
  string user_id = 3;             // User/Customer ID
  double trip_fare = 4;           // Trip fare amount
  double final_fare = 5;          // Final fare after promo
  double extra = 6;               // Extra charges
  double tips = 7;                // Driver tips
  string promo_code = 8;          // Applied promo code
  double discount_amount = 9;     // Discount from promo
}
```

**Response**: Standard transaction response with status

**Use Case**: Dipanggil setelah trip selesai untuk finalisasi pembayaran

---

### CancelTx

Cancel transaction jika trip dibatalkan.

**Method**: `CancelTx`  
**Interceptors**: `ValidateInput`

**Request Fields**:
```protobuf
message CancelTxRequest {
  int32 order_id = 1;            // Order ID to cancel
  string user_id = 2;            // User/Customer ID
  string reason = 3;             // Cancellation reason
  double cancellation_fee = 4;   // Fee to charge
}
```

**Response**: Cancel status and refund information

---

### ReserveBalance

Reserve/check balance sebelum start transaction.

**Method**: `ReserveBalance`  
**Interceptors**: `ValidateInput`

**Request Fields**:
```protobuf
message ReserveBalanceRequest {
  string payment_type = 1;       // Payment type (CC, ECV, etc)
  string user_id = 2;            // User/Customer ID
  double amount = 3;             // Amount to reserve
  string payment_identifier = 4; // Payment method identifier
  int32 order_id = 5;            // Order ID
}
```

**Response**: Balance status, available balance

**Use Case**: Pre-check sebelum mulai trip untuk ensure balance cukup

---

### CheckSufficientBalanceForOrder

Quick check apakah balance cukup untuk order.

**Method**: `CheckSufficientBalanceForOrder`

**Request Fields**:
```protobuf
message CheckSufficientBalanceForOrderRequest {
  int64 order_id = 1;            // Order ID
  double actual_argo = 2;        // Actual argo amount
}
```

**Response**: Boolean sufficient balance

---

### TxStatus

Get status transaksi.

**Method**: `TxStatus`  
**Interceptors**: `ValidateInput`

**Request Fields**:
```protobuf
message TxStatusRequest {
  int32 order_id = 1;            // Order ID
  string user_id = 2;            // User/Customer ID
}
```

**Response**: Transaction status details

---

## 💳 Credit Card APIs

### RegisterCC

Register kartu kredit baru.

**Method**: `RegisterCC`  
**Interceptors**: `ValidateMetadata`

**Request Fields**:
```protobuf
message RegisterCCRequest {
  string user_id = 1;            // User ID
  string card_number = 2;        // Card number (masked)
  string card_holder = 3;        // Card holder name
  string expiry_month = 4;       // Expiry month
  string expiry_year = 5;        // Expiry year
  string cvv = 6;                // CVV (not stored)
}
```

**Response**: Card token, card info

---

### RemoveCC

Remove/unlink kartu kredit.

**Method**: `RemoveCC`  
**Interceptors**: `ValidateMetadata`

**Request Fields**:
```protobuf
message RemoveCCRequest {
  string user_id = 1;            // User ID
  string card_token = 2;         // Card token to remove
}
```

---

### PreauthCCReserve

Pre-authorize kartu kredit (hold amount).

**Method**: `PreauthCCReserve`  
**Interceptors**: `ValidateInput`

**Request Fields**:
```protobuf
message PreauthCCReserveRequest {
  string user_id = 1;            // User ID
  string card_token = 2;         // Card token
  double amount = 3;             // Amount to hold
  int32 order_id = 4;            // Order ID
  string currency = 5;           // Currency code
}
```

**Response**: Pre-auth ID, status

---

### PreauthCCStatus

Check status pre-authorization.

**Method**: `PreauthCCStatus`  
**Interceptors**: `ValidateInput`

**Request Fields**:
```protobuf
message PreauthCCStatusRequest {
  string preauth_id = 1;         // Pre-auth ID
  int32 order_id = 2;            // Order ID
}
```

**Response**: Pre-auth status, held amount

---

### PreauthCCCallback

Callback handler untuk pre-auth dari gateway.

**Method**: `PreauthCCCallback`  
**Interceptors**: `ValidateInput`

---

## 🏢 Corporate Payment APIs

### VerifyECV

Verify corporate voucher.

**Method**: `VerifyECV`  
**Interceptors**: `ValidateMetadata`

**Request Fields**:
```protobuf
message VerifyECVRequest {
  string user_id = 1;            // User ID
  string company_code = 2;       // Company code
  string employee_id = 3;        // Employee ID
  string cost_center = 4;        // Cost center
}
```

**Response**: ECV account info, balance

---

### GetECVPolicy

Get ECV policy untuk company.

**Method**: `GetECVPolicy`

**Request Fields**:
```protobuf
message GetECVPolicyRequest {
  string company_code = 1;       // Company code
  string service_type = 2;       // Service type
}
```

**Response**: Policy rules, limits, restrictions

---

### ValidateECVPolicy

Validate apakah trip sesuai policy.

**Method**: `ValidateECVPolicy`

**Request Fields**:
```protobuf
message ValidateECVPolicyRequest {
  string company_code = 1;       // Company code
  string pickup_time = 2;        // Pickup time
  double pickup_lat = 3;         // Pickup latitude
  double pickup_lng = 4;         // Pickup longitude
  double dest_lat = 5;           // Destination latitude
  double dest_lng = 6;           // Destination longitude
  string service_type = 7;       // Service type
}
```

**Response**: Valid or not, policy violations

---

### ReserveBalanceECVGoldenbird

Reserve ECV balance untuk Goldenbird transaction.

**Method**: `ReserveBalanceECVGoldenbird`  
**Interceptors**: `ValidateInput`

---

### CompleteTxECVGoldenbird

Complete ECV transaction.

**Method**: `CompleteTxECVGoldenbird`  
**Interceptors**: `ValidateInput`

---

### VoidECVGoldenbird

Void/rollback ECV transaction.

**Method**: `VoidECVGoldenbird`  
**Interceptors**: `ValidateInput`

---

### CancelOrderECVGoldenbird

Cancel order dengan ECV payment.

**Method**: `CancelOrderECVGoldenbird`  
**Interceptors**: `ValidateInput`

---

### VoucherAccountDetail

Get detail voucher account.

**Method**: `VoucherAccountDetail`  
**Interceptors**: `ValidateInput`

---

## 🎫 Trip Voucher APIs

### AddTripVoucher

Add trip voucher ke account user.

**Method**: `AddTripVoucher`  
**Interceptors**: `ValidateMetadata, ValidateInput`

**Request Fields**:
```protobuf
message AddTripVoucherRequest {
  string user_id = 1;            // User ID
  string voucher_code = 2;       // Voucher code
}
```

**Response**: Voucher info, balance

---

### VerifyTripVoucher

Verify trip voucher validity.

**Method**: `VerifyTripVoucher`

**Request Fields**:
```protobuf
message VerifyTripVoucherRequest {
  string user_id = 1;            // User ID
  string voucher_id = 2;         // Voucher ID
  double amount = 3;             // Amount to use
}
```

**Response**: Voucher valid or not, available balance

---

### StartTripVoucherTx

Start trip voucher transaction.

**Method**: `StartTripVoucherTx`

---

### CompleteTripVoucherTx

Complete trip voucher transaction.

**Method**: `CompleteTripVoucherTx`

---

## 💰 E-Wallet APIs

### LoginWallet

Login/link e-wallet account.

**Method**: `LoginWallet`  
**Interceptors**: `ValidateMetadata, ValidateInput`

**Request Fields**:
```protobuf
message LoginWalletRequest {
  string user_id = 1;            // User ID
  string provider = 2;           // Wallet provider (ovo, linkaja)
  string phone_number = 3;       // Phone number
  string otp = 4;                // OTP code
}
```

**Response**: Wallet token, account info

---

### RemoveWallet

Remove/unlink e-wallet.

**Method**: `RemoveWallet`  
**Interceptors**: `ValidateMetadata, ValidateInput`

---

### GetEwalletURL

Get e-wallet login URL.

**Method**: `GetEwalletURL`

**Request Fields**:
```protobuf
message GetEwalletURLRequest {
  string provider = 1;           // Wallet provider
  string callback_url = 2;       // Callback URL after login
}
```

**Response**: Login URL

---

### GetEwalletBalance

Get e-wallet balance.

**Method**: `GetEwalletBalance`

**Request Fields**:
```protobuf
message GetEwalletBalanceRequest {
  string user_id = 1;            // User ID
  string provider = 2;           // Wallet provider
}
```

**Response**: Balance amount, currency

---

### WalletBalance

Get wallet balance (v4 API).

**Method**: `WalletBalance`  
**Interceptors**: `ValidateInput`

---

### StartWalletTx

Start wallet transaction.

**Method**: `StartWalletTx`

---

### CompleteWalletTx

Complete wallet transaction.

**Method**: `CompleteWalletTx`

---

## 🎁 Promo APIs

### PromoRedeem

Redeem promo code.

**Method**: `PromoRedeem`  
**Interceptors**: `ValidateInput`

**Request Fields**:
```protobuf
message PromoRedeemRequest {
  string user_id = 1;            // User ID
  string promo_code = 2;         // Promo code
  string epayment_token = 3;     // Payment token
  double final_fare = 4;         // Final fare amount
}
```

**Response**: Promo info, discount amount

---

### PromoRedeemCalculation

Calculate promo discount amount.

**Method**: `PromoRedeemCalculation`  
**Interceptors**: `ValidateInput`

**Request Fields**:
```protobuf
message PromoRedeemCalculationRequest {
  string identifier = 1;         // User identifier
  string promotion_code = 2;     // Promo code
  double discount_amount = 3;    // Discount amount
  int32 order_id = 4;            // Order ID
}
```

**Response**: Calculated discount

---

### PromoReservedAmount

Reserve promo budget untuk prevent double use.

**Method**: `PromoReservedAmount`  
**Interceptors**: `ValidateInput`

**Request Fields**:
```protobuf
message PromoReservedAmountRequest {
  int32 order_id = 1;            // Order ID
  string identifier = 2;         // User identifier
  string estimate_amount = 3;    // Estimated amount
  string promotion_code = 4;     // Promo code
  string redeem_date = 5;        // Redeem date
}
```

---

### PromoRemoveReservedBudget

Remove reserved promo budget (on cancel).

**Method**: `PromoRemoveReservedBudget`  
**Interceptors**: `ValidateInput`

**Request Fields**:
```protobuf
message PromoRemoveReservedBudgetRequest {
  int32 order_id = 1;            // Order ID
}
```

---

### ValidatePromo

Validate promo code.

**Method**: `ValidatePromo`

**Request Fields**:
```protobuf
message ValidatePromoRequest {
  string customer_id = 1;        // Customer ID
  string promo_code = 2;         // Promo code
  string service_type = 3;       // Service type
  string payment_type = 4;       // Payment type
  double minimum_fare = 5;       // Minimum fare
}
```

**Response**: Valid or not, promo details

---

### ValidateMarketingPromo

Validate marketing promo.

**Method**: `ValidateMarketingPromo`

---

## 💳 Payment Method APIs

### GetPaymentMethod

Get list payment method available untuk user.

**Method**: `GetPaymentMethod`

**Request Fields**:
```protobuf
message GetPaymentMethodRequest {
  string user_id = 1;            // User ID
  string service_type = 2;       // Service type
  double estimated_fare = 3;     // Estimated fare
}
```

**Response**: List of payment methods with availability

---

### UpdatePaymentMethod

Update payment method untuk order.

**Method**: `UpdatePaymentMethod`

---

### GetPaymentToken

Get payment token untuk specific payment method.

**Method**: `GetPaymentToken`  
**Interceptors**: `ValidateInput`

---

### GetSupportedPaymentMethod

Get supported payment methods untuk service.

**Method**: `GetSupportedPaymentMethod`

---

## 🔄 Refund & Adjustment APIs

### Refund

Process refund transaction.

**Method**: `Refund`

**Request Fields**:
```protobuf
message RefundRequest {
  int32 order_id = 1;            // Order ID
  string user_id = 2;            // User ID
  double refund_amount = 3;      // Amount to refund
  string reason = 4;             // Refund reason
}
```

---

### TransactionRefund

Transaction refund v2.

**Method**: `TransactionRefund`

---

### PaymentRefundWebhook

Webhook handler untuk refund notification.

**Method**: `PaymentRefundWebhook`

---

### AdjustmentChargeTx

Adjustment charge transaction (untuk extra charge).

**Method**: `AdjustmentChargeTx`  
**Interceptors**: `ValidateInput`

---

### ChargeCancelFee

Charge cancellation fee.

**Method**: `ChargeCancelFee`

---

### CallbackAdjustChargeStatus

Callback untuk adjustment charge status.

**Method**: `CallbackAdjustChargeStatus`

---

### CallbackUpdateChargeStatus

Callback untuk update charge status.

**Method**: `CallbackUpdateChargeStatus`

---

## 🔄 Direct Payment APIs

### DirectPay

Direct payment untuk simple payment flow.

**Method**: `DirectPay`

**Request Fields**:
```protobuf
message DirectPayRequest {
  string user_id = 1;            // User ID
  string payment_type = 2;       // Payment type
  double amount = 3;             // Amount to pay
  int32 order_id = 4;            // Order ID
  string payment_token = 5;      // Payment token
}
```

---

### DirectPaymentV1_1

Direct payment v1.1 (enhanced version).

**Method**: `DirectPaymentV1_1`

---

### SinglePaymentMethod

Single payment method transaction.

**Method**: `SinglePaymentMethod`  
**Interceptors**: `ValidateInput`

---

## 🔄 Hold-On Transaction APIs

### HOTransaction

Hold-on transaction untuk payment tertunda.

**Method**: `HOTransaction`

---

### HOECVTransaction

HO transaction khusus untuk ECV.

**Method**: `HOECVTransaction`

---

### ProcessHOTransaction

Process HO transaction (PubSub subscriber).

**Method**: `ProcessHOTransaction`

---

### ProcessHOECVTransaction

Process HO ECV transaction (PubSub subscriber).

**Method**: `ProcessHOECVTransaction`

---

### PushHOGoldenbirdCC

Push HO Goldenbird CC transaction.

**Method**: `PushHOGoldenbirdCC`  
**Interceptors**: `ValidateInput`

---

## 🎯 Goldenbird Specific APIs

### PaymentCCGoldenbird

Payment dengan CC untuk Goldenbird booking.

**Method**: `PaymentCCGoldenbird`  
**Interceptors**: `ValidateInput`

---

### CancelTxGoldenbird

Cancel transaction Goldenbird.

**Method**: `CancelTxGoldenbird`  
**Interceptors**: `ValidateInput`

---

### UpdatePaymentGoldenbird

Update payment Goldenbird order.

**Method**: `UpdatePaymentGoldenbird`

---

### VoidCCGoldenbird

Void CC transaction Goldenbird.

**Method**: `VoidCCGoldenbird`  
**Interceptors**: `ValidateInput`

---

## 🔧 Utility APIs

### HealthCheck

Service health check.

**Method**: `HealthCheck`

**Response**: Service status, dependencies status

---

### ValidateEpaymentProfile

Validate epayment profile.

**Method**: `ValidateEpaymentProfile`  
**Interceptors**: `ValidateInput`

---

### NotifyReferral

Notify referral system.

**Method**: `NotifyReferral`

---

### TipsDriver

Process driver tips payment.

**Method**: `TipsDriver`

---

### RechargeTx

Recharge transaction (for wallet topup).

**Method**: `RechargeTx`  
**Interceptors**: `ValidateInput`

---

### TransactionStatus

Get transaction status.

**Method**: `TransactionStatus`

---

### TxCancelSync

Sync cancel transaction.

**Method**: `TxCancelSync`

---

### UpdateTxResult

Update transaction result.

**Method**: `UpdateTxResult`  
**Interceptors**: `ValidateInput`

---

### AdjustmentTxResult

Adjustment transaction result.

**Method**: `AdjustmentTxResult`  
**Interceptors**: `ValidateInput`

---

### TxCompletionCallback

Transaction completion callback.

**Method**: `TxCompletionCallback`  
**Interceptors**: `ValidateInput`

---

### CompleteTxOld

Complete transaction (old version).

**Method**: `CompleteTxOld`

---

## 🔗 Related Documentation

- [[README|Payment Processor Overview]]
- [[dependencies|Dependencies]]
- [[02-Work/Teams/MRG/00-overview/README|MRG Team Overview]]

---

## 🏷️ Tags

#mrg #api-reference #payment #grpc #rest

---

*Last Updated*: 2025-01-07
