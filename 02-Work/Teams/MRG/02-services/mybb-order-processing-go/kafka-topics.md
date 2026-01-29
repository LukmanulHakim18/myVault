---
tags:
  - mrg
  - service
  - kafka
  - topics
  - order-processing
parent: '[[README]]'
created: '2026-01-29'
updated: '2026-01-29'
total_topics: 91
---
# Order Processing - Kafka Topics Reference

**Parent**: [[README|Order Processing Service]]

---

## 📊 Summary

| Category | Count |
|----------|-------|
| Core Order Operations | 8 |
| Payment & Billing | 8 |
| Trip History & Receipt | 8 |
| Query Operations | 6 |
| Rating & Feedback | 5 |
| Open API B2B | 5 |
| Wallet Operations | 5 |
| OTP & Verification | 3 |
| Golden Bird | 6 |
| Fare & Pricing | 5 |
| Miscellaneous | 32 |
| **Total** | **91** |

---

## 🎯 Core Order Operations (8 Topics)

| Topic | Consumer | Group | Description |
|-------|----------|-------|-------------|
| `TOPIC_REQUEST_BOOKING` | RequestBookingConsumer | ✅ Yes | Create new booking |
| `TOPIC_UPDATE_CURRENT_BOOKING` | UpdateCurrentBookingConsumer | ❌ Event | Update dari BBD (state, driver, vehicle) |
| `TOPIC_CANCEL_BOOKING` | CancelBookingConsumer | ✅ Yes | Cancel by user |
| `TOPIC_CANCEL_BOOKING_BY_SYSTEM` | CancelBookingBySystemConsumer | ❌ Event | System cancellation |
| `TOPIC_END_TRIP` | EndTripConsumer | ✅ Yes | End trip dari BBD |
| `TOPIC_UPDATE_ORDER` | UpdateOrderConsumer | ✅ Yes | Generic order update |
| `TOPIC_ORDER_CHANGES` | OrderChangesConsumer | ❌ Simple | Stream order changes |
| `TOPIC_ORDER_CALLBACK` | OrderCallbackConsumer | ✅ Yes | BBD callback |

### Consumer Group Strategy

| Type | Behavior | Use Case |
|------|----------|----------|
| **Group Consumer** | Guaranteed delivery, replay on restart | Critical operations |
| **Event Consumer** | No replay, lose data on restart | Real-time updates |
| **Simple Consumer** | Stream-based, no group | Continuous streaming |

---

## 💳 Payment & Billing (8 Topics)

| Topic | Consumer | Group | Description |
|-------|----------|-------|-------------|
| `TOPIC_RECHARGE_BOOKING` | RechargeBookingConsumer | ✅ Yes | Recharge order |
| `TOPIC_RECHARGE_BOOKING_WITH_PAYMENT` | RechargeBookingWithPaymentConsumer | ✅ Yes | Recharge dengan payment |
| `TOPIC_UPDATE_CHARGE_STATUS` | UpdateChargeStatusConsumer | ❌ Event | Payment status callback |
| `TOPIC_UPDATE_REFUND_STATUS` | UpdateRefundStatusConsumer | ✅ Yes | Refund status callback |
| `TOPIC_RETRY_CANCELLED_BOOKING_REFUND` | RetryRefundConsumer | ✅ Yes | Retry failed refund |
| `TOPIC_CANCELLATION_FEE` | CancellationFeeConsumer | ✅ Yes | Calculate cancel fee |
| `TOPIC_TIPS_DRIVER` | TipsDriverConsumer | ✅ Yes | Driver tipping |
| `TOPIC_ADJUSTMENT_CHARGE_STATUS` | AdjustmentChargeStatusConsumer | ❌ Event | Fare adjustment |

---

## 📜 Trip History & Receipt (8 Topics)

| Topic | Consumer | Group | Description |
|-------|----------|-------|-------------|
| `TOPIC_GET_TRIP_HISTORY` | GetTripHistoryConsumer | ✅ Yes | Get trip history (legacy) |
| `TOPIC_GET_TRIP_HISTORY_V4` | GetTripHistoryV4Consumer | ✅ Yes | Trip history v4 |
| `TOPIC_GET_TRIP_HISTORY_V5` | GetTripHistoryV5Consumer | ✅ Yes | Trip history v5 (current) |
| `TOPIC_CREATE_TRIP_RECEIPT` | CreateTripReceiptConsumer | ✅ Yes | Generate trip receipt |
| `TOPIC_GET_TRIP_RECEIPT` | GetTripReceiptConsumer | ✅ Yes | Get existing receipt |
| `TOPIC_SHARE_TRIP` | ShareTripConsumer | ✅ Yes | Share trip to email |
| `TOPIC_SHARE_RECEIPT` | ShareReceiptConsumer | ✅ Yes | Share receipt |
| `TOPIC_REGENERATE_TRIP_MAP` | RegenerateTripMapConsumer | ✅ Yes | Regenerate trip image |

---

## 🔍 Query Operations (6 Topics)

| Topic | Consumer | Group | Description |
|-------|----------|-------|-------------|
| `TOPIC_GET_ONGOING_ORDERS` | GetOngoingOrdersConsumer | ✅ Yes | Get ongoing orders (legacy) |
| `TOPIC_GET_ONGOING_ORDERS_V3` | GetOngoingOrdersV3Consumer | ✅ Yes | Ongoing orders v3 |
| `TOPIC_GET_ACTIVE_BOOKINGS` | GetActiveBookingsConsumer | ✅ Yes | Get active bookings |
| `TOPIC_GET_CURRENT_BOOKING` | GetCurrentBookingConsumer | ✅ Yes | Get current booking |
| `TOPIC_QUERY_OUTSTANDING_ORDERS` | QueryOutstandingOrdersConsumer | ✅ Yes | Query outstanding |
| `TOPIC_QUERY_ORDERS_NOT_END` | QueryOrdersNotEndConsumer | ✅ Yes | Orders not ended |

---

## ⭐ Rating & Feedback (5 Topics)

| Topic | Consumer | Group | Description |
|-------|----------|-------|-------------|
| `TOPIC_RATE_DRIVER` | RateDriverConsumer | ✅ Yes | Rate driver |
| `TOPIC_RATE_ORDER` | RateOrderConsumer | ✅ Yes | Rate order |
| `TOPIC_SAVE_RATING` | SaveRatingConsumer | ✅ Yes | Save rating |
| `TOPIC_GET_RATING_FLAG` | GetRatingFlagConsumer | ✅ Yes | Get rating flag |
| `TOPIC_HIDE_RATING` | HideRatingConsumer | ✅ Yes | Hide rating |

---

## 🌐 Open API B2B (5 Topics)

| Topic | Consumer | Group | Description |
|-------|----------|-------|-------------|
| `TOPIC_OPEN_API_ORDER_STATE_CHANGED` | OpenAPIOrderStateChangedConsumer | ✅ Yes | Order state changed |
| `TOPIC_OPEN_API_CANCEL_BOOKING` | OpenAPICancelBookingConsumer | ✅ Yes | Cancel booking |
| `TOPIC_OPEN_API_GET_MERCHANT_ORDERS` | OpenAPIGetMerchantOrdersConsumer | ✅ Yes | Get merchant orders |
| `TOPIC_OPEN_API_ORDER_FINISHED` | OpenAPIOrderFinishedConsumer | ✅ Yes | Order finished |
| `TOPIC_OPEN_API_TRACK_ORDER` | OpenAPITrackOrderConsumer | ✅ Yes | Track order |

---

## 👛 Wallet Operations (5 Topics)

| Topic | Consumer | Group | Description |
|-------|----------|-------|-------------|
| `TOPIC_CHECK_WALLET_BALANCE` | CheckWalletBalanceConsumer | ✅ Yes | Check balance |
| `TOPIC_GET_WALLET_BALANCE` | GetWalletBalanceConsumer | ✅ Yes | Get balance |
| `TOPIC_RESERVE_WALLET` | ReserveWalletConsumer | ✅ Yes | Reserve balance |
| `TOPIC_RELEASE_WALLET` | ReleaseWalletConsumer | ✅ Yes | Release reserved |
| `TOPIC_E2C_TRANSACTION` | E2CTransactionConsumer | ✅ Yes | E-payment to Cash |

---

## 🔐 OTP & Verification (3 Topics)

| Topic | Consumer | Group | Description |
|-------|----------|-------|-------------|
| `TOPIC_VERIFY_OTP` | VerifyOTPConsumer | ✅ Yes | Verify OTP |
| `TOPIC_RESEND_OTP` | ResendOTPConsumer | ✅ Yes | Resend OTP |
| `TOPIC_STORE_FAILOVER_OTP_INFO` | StoreFailoverOTPInfoConsumer | ❌ Simple | Store failover OTP |

---

## 🚗 Golden Bird (6 Topics)

| Topic | Consumer | Group | Description |
|-------|----------|-------|-------------|
| `TOPIC_GB_CREATE_ORDER` | GBCreateOrderConsumer | ✅ Yes | Create GB order |
| `TOPIC_GB_UPDATE_ORDER` | GBUpdateOrderConsumer | ✅ Yes | Update GB order |
| `TOPIC_GB_CANCEL_ORDER` | GBCancelOrderConsumer | ✅ Yes | Cancel GB order |
| `TOPIC_GB_GET_ORDER_DETAIL` | GBGetOrderDetailConsumer | ✅ Yes | Get GB order detail |
| `TOPIC_GB_LIMO_CREATE_ORDER` | GBLimoCreateOrderConsumer | ✅ Yes | Create GB Limo order |
| `TOPIC_GB_RENT_MULTIDAYS` | GBRentMultidaysConsumer | ✅ Yes | GB Rent Multidays |

---

## 💰 Fare & Pricing (5 Topics)

| Topic | Consumer | Group | Description |
|-------|----------|-------|-------------|
| `TOPIC_GET_FARE_ESTIMATION` | GetFareEstimationConsumer | ✅ Yes | Get fare estimation |
| `TOPIC_UPDATE_ORDER_FEES` | UpdateOrderFeesConsumer | ✅ Yes | Update order fees |
| `TOPIC_UPDATE_ORDER_FEES_STATUS` | UpdateOrderFeesStatusConsumer | ❌ Event | Fees status callback |
| `TOPIC_CALCULATE_SURGE_PRICE` | CalculateSurgePriceConsumer | ✅ Yes | Calculate surge |
| `TOPIC_GET_FIXED_FARE` | GetFixedFareConsumer | ✅ Yes | Get fixed fare |

---

## 📦 Miscellaneous (32 Topics)

### Vehicle & Driver
| Topic | Consumer | Description |
|-------|----------|-------------|
| `TOPIC_GET_VEHICLE_INFO` | GetVehicleInfoConsumer | Get vehicle info |
| `TOPIC_UPDATE_VEHICLE_INFO` | UpdateVehicleInfoConsumer | Update vehicle info |
| `TOPIC_GET_DRIVER_LOCATION` | GetDriverLocationConsumer | Get driver location |
| `TOPIC_CALL_DRIVER` | CallDriverConsumer | Call driver |

### Notification
| Topic | Consumer | Description |
|-------|----------|-------------|
| `TOPIC_SEND_NOTIFICATION` | SendNotificationConsumer | Send push notification |
| `TOPIC_SEND_EMAIL` | SendEmailConsumer | Send email |

### Subscription
| Topic | Consumer | Description |
|-------|----------|-------------|
| `TOPIC_CREATE_SUBSCRIPTION` | CreateSubscriptionConsumer | Create subscription |
| `TOPIC_UPDATE_SUBSCRIPTION` | UpdateSubscriptionConsumer | Update subscription |
| `TOPIC_CANCEL_SUBSCRIPTION` | CancelSubscriptionConsumer | Cancel subscription |

### Promo & Marketing
| Topic | Consumer | Description |
|-------|----------|-------------|
| `TOPIC_APPLY_PROMO` | ApplyPromoConsumer | Apply promo code |
| `TOPIC_VALIDATE_PROMO` | ValidatePromoConsumer | Validate promo |
| `TOPIC_RESERVE_BUDGET_MKT` | ReserveBudgetMKTConsumer | Reserve marketing budget |

### Post Request Booking
| Topic | Consumer | Description |
|-------|----------|-------------|
| `TOPIC_POST_REQUEST_BOOKING` | PostRequestBookingConsumer | PreAuth callback |

### Live Tracking
| Topic | Consumer | Description |
|-------|----------|-------------|
| `TOPIC_STORE_BUS_LIVE_TRACKING_STATUS` | StoreBusLiveTrackingStatusConsumer | Bus tracking |

### Miscellaneous
| Topic | Consumer | Description |
|-------|----------|-------------|
| `TOPIC_REMOVE_BOOKING` | RemoveBookingConsumer | Remove booking |
| `TOPIC_UPDATE_TRIP_PURPOSE` | UpdateTripPurposeConsumer | Update trip purpose |
| `TOPIC_UPDATE_CUSTOMER_INFO` | UpdateCustomerInfoConsumer | Update customer info |
| `TOPIC_GET_LAST_TRIP_PURPOSE` | GetLastTripPurposeConsumer | Get last trip purpose |
| `TOPIC_CHECK_ORDER_UPSELL` | CheckOrderUpsellConsumer | Check upsell eligibility |
| `TOPIC_NOTIFY_BOOKING_RESULT` | - | Notify booking result (produce only) |
| `TOPIC_NOTIFY_END_TRIP` | - | Notify end trip (produce only) |
| `TOPIC_NOTIFY_ORDER_CHANGES` | - | Notify order changes (produce only) |

---

## 🔧 Topic Registration

```go
// router/router.go
func RegisterTopics(consumer *consumers.BaseConsumer) {
    // Core Operations
    consumer.Register(TOPIC_REQUEST_BOOKING, &consumers.RequestBookingConsumer{})
    consumer.Register(TOPIC_UPDATE_CURRENT_BOOKING, &consumers.UpdateCurrentBookingConsumer{}, WithEventConsumer())
    consumer.Register(TOPIC_CANCEL_BOOKING, &consumers.CancelBookingConsumer{})
    // ... 88 more topics
}
```

---

## 📝 Message Contract Examples

### Request Booking
```json
{
    "gateway_data": {
        "uuid": "request-id",
        "device_uuid": "device-id"
    },
    "meta_data": {
        "user_id": "BB00123456",
        "app_version": "6.32.0"
    },
    "pickup_latitude": -6.2088,
    "pickup_longitude": 106.8456,
    "pickup_address": "Jl. Sudirman No. 1",
    "dropoff_latitude": -6.1753,
    "dropoff_longitude": 106.8271,
    "dropoff_address": "Jl. Gatot Subroto",
    "service_type_id": 0,
    "payment_method": "cash",
    "is_on_street_ride": false
}
```

### Update Current Booking
```json
{
    "external_order_id": 12345,
    "itop_id": "IT001",
    "state": "DRIVER_FOUND",
    "driver_id": "D001",
    "driver_name": "John Doe",
    "car_number": "B 1234 CD",
    "car_latitude": -6.2088,
    "car_longitude": 106.8456
}
```

### End Trip
```json
{
    "external_order_id": 12345,
    "itop_id": "IT001",
    "trip_fare": 50000,
    "distance": 10.5,
    "duration": 30,
    "waypoints": [
        {"lat": -6.2088, "lng": 106.8456, "time": "2026-01-29T10:00:00Z"},
        {"lat": -6.1753, "lng": 106.8271, "time": "2026-01-29T10:30:00Z"}
    ]
}
```

---

## 🔗 Related Documentation

- [[README|Order Processing Service]]
- [[dependencies|Dependencies]]
- [[order-flows|Order Flows]]
- [[api-reference|API Reference]]

---

#mrg #service #kafka #topics #order-processing

---

*Last Updated*: 2026-01-29
