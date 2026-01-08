# API Reference

## Overview
Notification Center service menyediakan berbagai endpoints untuk mengirim notifikasi melalui multiple channels. API tersedia via **gRPC** (primary) dan **REST** (via gRPC Gateway).

## Base URLs

### gRPC
```
grpc://notificationcenter:50051
```

### REST (via Gateway)
```
http://notificationcenter:8080
```

## API Categories

### Core Notification APIs
- [[#Push Notification|Push Notifications]]
- [[#SMS|SMS Messaging]]
- [[#Email|Email Service]]
- [[#WhatsApp|WhatsApp Messaging]]

### Authentication & Security
- [[#OTP|OTP Management]]
- [[#Device Management|Device Subscription]]
- [[#Password Reset|Password Reset]]

### Order & Booking
- [[#Order Receipts|Order Receipts]]
- [[#Booking Notifications|Booking Events]]

### Email V2 (Strategy-Based)
- [[#Email V2 API|Email V2 Endpoints]]

### Legacy Support
- [[#Legacy APIs|Legacy Endpoints]]

---

## Push Notification

### Subscribe
Register device token untuk push notifications.

**gRPC**: `NotificationService.Subscribe`
**REST**: `POST /api/v1/subscribe`

**Request**:
```json
{
  "device_token": "fcm_token_here",
  "user_id": "user_123",
  "meta_data": {
    "device_type": "android",
    "app_version": "2.5.0",
    "device_id": "device_uuid"
  }
}
```

**Response**:
```json
{
  "code": 200,
  "message": "Success"
}
```

### Unsubscribe
Unregister device token.

**gRPC**: `NotificationService.Unsubscribe`
**REST**: `POST /api/v1/unsubscribe`

**Request**: Same as Subscribe

**Response**:
```json
{
  "code": 200,
  "message": "Success"
}
```

### Push
Send push notification to user.

**gRPC**: `NotificationService.Push`
**REST**: `POST /api/v1/push`

**Request**:
```json
{
  "message": "Your driver has arrived",
  "user_id": "user_123",
  "meta_data": {
    "device_type": "android",
    "app_version": "2.5.0"
  }
}
```

**Response**:
```json
{
  "code": 200,
  "message": "Notification sent"
}
```

---

## SMS

### SendSms
Send SMS message to phone number.

**gRPC**: `NotificationService.SendSms`
**REST**: `POST /api/v1/sms/send`

**Request**:
```json
{
  "number": "+628123456789",
  "content": "Your OTP is: 123456",
  "requested_by": "user_service",
  "meta_data": {
    "app_version": "2.5.0",
    "device_type": "android"
  }
}
```

**Response**:
```json
{
  "code": 200,
  "message": "SMS sent successfully"
}
```

**Providers**:
- DART SMS Gateway (primary)
- Configurable via `UpdateSmsConfig`

### UpdateSmsConfig
Configure SMS prefix whitelist.

**gRPC**: `NotificationService.UpdateSmsConfig`
**REST**: `POST /api/v1/sms/config`

**Request**:
```json
{
  "prefix": ["+62", "+65", "+60"]
}
```

### UpdateSmsStatus
Callback endpoint for SMS delivery status.

**gRPC**: `NotificationService.UpdateSmsStatus`
**REST**: `POST /api/v1/sms/status`

**Request**:
```json
{
  "message_id": "msg_123",
  "status": "DELIVERED",
  "provider_type": "DART"
}
```

**Status Values**:
- `SENT`
- `DELIVERED`
- `FAILED`
- `EXPIRED`

---

## Email

### SendEmail
Send transactional email (legacy format).

**gRPC**: `NotificationService.SendEmail`
**REST**: `POST /api/v1/email/send`

**Request**:
```json
{
  "from": "noreply@bluebird.id",
  "recipients": ["customer@example.com"],
  "cc": ["support@bluebird.id"],
  "subject": "Your Order Confirmation",
  "content": "<html>...</html>"
}
```

**Response**:
```json
{
  "code": 200,
  "message": "Email sent"
}
```

### RawEmail
Send raw HTML email without templates.

**gRPC**: `NotificationService.RawEmail`
**REST**: `POST /api/v1/email/raw`

**Request**:
```json
{
  "from": "noreply@bluebird.id",
  "recipients": ["user@example.com"],
  "cc": [],
  "subject": "Subject Line",
  "content": "<html><body>Email content</body></html>"
}
```

### ResetPassword
Send password reset email.

**gRPC**: `NotificationService.ResetPassword`
**REST**: `POST /api/v1/email/reset-password`

**Request**:
```json
{
  "full_name": "John Doe",
  "token": "reset_token_abc123",
  "email": "john@example.com",
  "hash": "user_hash",
  "link": "https://mybb.app/reset-password?token=...",
  "alternative_link": "https://web.bluebird.id/reset",
  "meta_data": {
    "app_version": "2.5.0",
    "accept_language": "en"
  }
}
```

### RegisterCreditCard
Send credit card registration confirmation.

**gRPC**: `NotificationService.RegisterCreditCard`
**REST**: `POST /api/v1/email/credit-card`

**Request**:
```json
{
  "full_name": "John Doe",
  "image_host": "https://cdn.bluebird.id",
  "email": "john@example.com"
}
```

### SendEmailChangeEmail
Send email change verification.

**gRPC**: `NotificationService.SendEmailChangeEmail`
**REST**: `POST /api/v1/email/change-email`

**Request**:
```json
{
  "email": "newemail@example.com",
  "full_name": "John Doe",
  "otp": "123456",
  "request_at": "2025-01-07T10:00:00Z",
  "domain": "mybb",
  "meta_data": {...}
}
```

### OrderReceiptNotifEmail
Send order receipt email.

**gRPC**: `NotificationService.OrderReceiptNotifEmail`
**REST**: `POST /api/v1/email/order-receipt`

**Request**:
```json
{
  "recipients": "customer@example.com",
  "data": {
    "order_id": 12345,
    "pickup_city": "Jakarta",
    "transaction_date": "2025-01-07",
    "customer_name": "John Doe",
    "total": "Rp 150,000",
    "distance": "10.5 km",
    "duration": "25 min",
    "pickup_address_detail": "Jl. Sudirman No. 123",
    "dropoff_address_detail": "Jl. Thamrin No. 456",
    "driver_name": "Pak Driver",
    "plat_number": "B 1234 XYZ",
    "service_type": "BlueBird",
    "fare": "Rp 130,000",
    "discount": "Rp 10,000",
    "handling_fee": "Rp 5,000"
  },
  "meta_data": {...}
}
```

### SendEmailNewPayment
Notify about new payment method.

**gRPC**: `NotificationService.SendEmailNewPayment`
**REST**: `POST /api/v1/email/new-payment`

**Request**:
```json
{
  "data": [
    {
      "internal_id": "payment_123",
      "payment_method_type": "CREDIT_CARD",
      "payment_method_display": "Visa **** 1234",
      "identifier": "card_token_xyz",
      "remaining_budget": 1000000.00,
      "remaining_trip": 10,
      "available": true,
      "Languange": "id",
      "CustomerEmail": "customer@example.com",
      "CustomerName": "John Doe",
      "CardNumber": "**** 1234",
      "BudgetAmount": "Rp 1,000,000",
      "TotalTrip": "10",
      "CompanyName": "PT Contoh"
    }
  ]
}
```

### SendEmailForCustomerCare
Send email on behalf of customer care.

**gRPC**: `NotificationService.SendEmailForCustomerCare`
**REST**: `POST /api/v1/email/customer-care`

**Request**:
```json
{
  "from": "support@bluebird.id",
  "recipients": ["customer@example.com"],
  "subject": "Response from Customer Care",
  "content": "<html>...</html>"
}
```

---

## Email V2 API

Email V2 menggunakan **Strategy Pattern** untuk routing berdasarkan channel, service, dan type.

**Lihat**: [[02-email-strategy|Email Strategy Documentation]]

### SendEmailV2
Send email using strategy-based routing.

**gRPC**: `NotificationService.SendEmailV2`
**REST**: `POST /api/v2/email/send`

**Request Structure**:
```protobuf
message SendEmailV2Request {
  string channel = 1;              // WebReservation | MyBB
  EmailV2ServiceCategory service_category = 2;  // RIDE | RENT
  EmailV2Type email_type = 3;      // ORDER_CREATED | PAYMENT_RECEIVED | etc
  
  oneof Request {
    EmailV2OrderCreatedRequest order_created = 10;
    EmailV2PaymentReceivedRequest payment_received = 11;
    EmailV2DriverAssignedRequest driver_assigned = 12;
    EmailV2DriverAssignedChangedRequest driver_assigned_changed = 13;
    EmailV2EReceiptRequest e_receipt = 14;
    EmailV2RescheduleRequest reschedule = 15;
    EmailV2CancelledBySystemRequest cancelled_by_system = 16;
    EmailV2CancelledByUserRequest cancelled_by_user = 17;
  }
}
```

### Email Types

#### ORDER_CREATED
```json
{
  "channel": "WebReservation",
  "service_category": "RIDE",
  "email_type": "ORDER_CREATED",
  "order_created": {
    "email": "customer@example.com",
    "order_id": "12345",
    "customer_name": "John Doe",
    "pickup_address": "Jl. Sudirman",
    "dropoff_address": "Jl. Thamrin",
    "pickup_datetime": "2025-01-07T10:00:00Z",
    "service_type": "BlueBird",
    "estimated_fare": "Rp 150,000"
  }
}
```

#### PAYMENT_RECEIVED
```json
{
  "channel": "MyBB",
  "service_category": "RENT",
  "email_type": "PAYMENT_RECEIVED",
  "payment_received": {
    "email": "customer@example.com",
    "order_id": "12345",
    "payment_amount": "Rp 500,000",
    "payment_method": "Credit Card",
    "transaction_id": "TRX123456"
  }
}
```

#### DRIVER_ASSIGNED
```json
{
  "channel": "WebReservation",
  "service_category": "RIDE",
  "email_type": "DRIVER_ASSIGNED",
  "driver_assigned": {
    "email": "customer@example.com",
    "order_id": "12345",
    "driver_name": "Pak Driver",
    "driver_phone": "+628123456789",
    "car_type": "BlueBird",
    "plate_number": "B 1234 XYZ",
    "estimated_arrival": "5 minutes"
  }
}
```

#### E_RECEIPT
```json
{
  "channel": "MyBB",
  "service_category": "RIDE",
  "email_type": "E_RECEIPT",
  "e_receipt": {
    "email": "customer@example.com",
    "order_id": "12345",
    "transaction_date": "2025-01-07T10:00:00Z",
    "total_fare": "Rp 150,000",
    "discount": "Rp 10,000",
    "final_amount": "Rp 140,000",
    "payment_method": "GoPay"
  }
}
```

**Response**:
```json
{
  "code": 200,
  "message": "Email sent successfully"
}
```

**Error Responses**:
```json
// Flow not covered by strategy
{
  "code": 501,
  "message": "DriverAssignedChanged: this flow is not covered by the current strategy"
}

// Invalid request
{
  "code": 400,
  "message": "Invalid email type"
}

// Internal error
{
  "code": 500,
  "message": "Failed to send email: connection timeout"
}
```

### SendEmailBBOne
Send email via BBOne email service with attachments.

**gRPC**: `NotificationService.SendEmailBBOne`
**REST**: `POST /api/v1/email/bbone`

**Request**:
```json
{
  "from": {
    "email": "noreply@bluebird.id",
    "name": "Bluebird Group"
  },
  "to": [
    {
      "email": "customer@example.com",
      "name": "John Doe"
    }
  ],
  "subject": "Your Order Confirmation",
  "html_content": "<html>...</html>",
  "attachments": [
    {
      "filename": "invoice.pdf",
      "content": "base64_encoded_content",
      "type": "application/pdf"
    }
  ]
}
```

---

## OTP

### SendOpt
Send OTP via Email, SMS, or WhatsApp.

**gRPC**: `NotificationService.SendOpt`
**REST**: `POST /api/v1/otp/send`

**Request**:
```json
{
  "internal_id": "user_123",
  "full_name": "John Doe",
  "destination": "+628123456789",  // or email@example.com
  "otp": "123456",
  "type": "SMS",  // or "EMAIL" or "WHATSAPP"
  "languange": "id",
  "request_at": "2025-01-07T10:00:00Z",
  "domain": "mybb",
  "request_by": "user_service",
  "purpose": "REGISTER"  // or "UPDATE_PHONE_NUMBER"
}
```

**OTP Types**:
```protobuf
enum OtpType {
  WHATSAPP = 0;
  EMAIL = 1;
  SMS = 2;
}

enum PurposeOtp {
  REGISTER = 0;
  UPDATE_PHONE_NUMBER = 1;
}
```

### GetAvailableOTPType
Get available OTP methods for a country.

**gRPC**: `NotificationService.GetAvailableOTPType`
**REST**: `GET /api/v1/otp/available?country_code=ID`

**Request**:
```json
{
  "country_code": "ID"
}
```

**Response**:
```json
{
  "otp_list": [
    {
      "type_id": 0,
      "name": "WhatsApp",
      "order_priority": 1,
      "message": "We'll send OTP to your WhatsApp"
    },
    {
      "type_id": 2,
      "name": "SMS",
      "order_priority": 2,
      "message": "We'll send OTP via SMS"
    },
    {
      "type_id": 1,
      "name": "Email",
      "order_priority": 3,
      "message": "We'll send OTP to your email"
    }
  ],
  "country_code": "ID"
}
```

### CallBackOtpWa
Callback for WhatsApp OTP delivery status.

**gRPC**: `NotificationService.CallBackOtpWa`
**REST**: `POST /api/v1/otp/callback/whatsapp`

**Request**:
```json
{
  "message_id": "wa_msg_123",
  "status": "DELIVERED",
  "phone_number": "+628123456789"
}
```

---

## WhatsApp

### OTPWALegacy
Send OTP via WhatsApp (legacy method).

**gRPC**: `NotificationService.OTPWALegacy`
**REST**: `POST /api/v1/whatsapp/otp`

**Request**:
```json
{
  "phone_number": "+628123456789",
  "otp": "123456",
  "usecase": "registration",
  "language": "id",
  "current_version": "2.5.0"
}
```

### UpdateWaLogLegacy
Update WhatsApp message log status.

**gRPC**: `NotificationService.UpdateWaLogLegacy`
**REST**: `POST /api/v1/whatsapp/log`

**Request**:
```json
{
  "message_id": "wa_msg_123",
  "status": "DELIVERED"
}
```

---

## Order & Booking Events

### GetBookingEvent
Event listener for booking lifecycle events.

**gRPC**: `NotificationService.GetBookingEvent`
**PubSub**: Topic `booking.events`

**Request**:
```json
{
  "order_id": "12345",
  "order_channel": "APP",  // APP | WEB | MITRA | CORP
  "order_event_status": "CREATED",  // CREATED | PAYMENT_RECEIVED | etc
  "customer_email": "customer@example.com",
  "customer_name": "John Doe",
  "customer_phone": "+628123456789",
  "pickup_address": "Jl. Sudirman",
  "dropoff_address": "Jl. Thamrin",
  "service_type": "BlueBird"
}
```

**Event Statuses**:
- `CREATED`
- `PAYMENT_RECEIVED`
- `DRIVER_ASSIGNED`
- `DRIVER_CHANGED`
- `PICKED_UP`
- `COMPLETED`
- `CANCELLED`
- `RESCHEDULED`

**Response**:
```json
{
  "code": 200,
  "message": "Notification sent"
}
```

### GetOrderItemEvent
Event listener for order item events.

**gRPC**: `NotificationService.GetOrderItemEvent`
**PubSub**: Topic `order.item.events`

### GetPaymentEvent
Event listener for payment events.

**gRPC**: `NotificationService.GetPaymentEvent`
**PubSub**: Topic `payment.events`

---

## Driver App Notifications

### SendNotifToDriverApp
Send notification to driver app.

**gRPC**: `NotificationService.SendNotifToDriverApp`
**REST**: `POST /api/v1/driver/notify`

**Request**:
```json
{
  "tip": 10000.00,
  "order_id": 12345,
  "external_order_id": "EXT12345",
  "external_driver_id": "DRV789"
}
```

---

## Analytics & Monitoring

### NotifySMSCount
Get SMS usage count for period.

**gRPC**: `NotificationService.NotifySMSCount`
**REST**: `GET /api/v1/analytics/sms/count`

**Request**:
```json
{
  "requested_at": "2025-01-07T10:00:00Z",
  "user_id": "user_123",
  "app_version": "2.5.0",
  "lang": "id",
  "os": "android",
  "manufacturer": "Samsung",
  "start_time": "2025-01-01T00:00:00Z",
  "end_time": "2025-01-07T23:59:59Z",
  "provider": "DART"
}
```

### NotifyEmailCount
Get email usage count for period.

**gRPC**: `NotificationService.NotifyEmailCount`
**REST**: `GET /api/v1/analytics/email/count`

---

## Special Use Cases

### NotifyAnnouncement
Send system-wide announcement.

**gRPC**: `NotificationService.NotifyAnnouncement`
**REST**: `POST /api/v1/notify/announcement`

### RemoveAnnouncement
Remove active announcement.

**gRPC**: `NotificationService.RemoveAnnouncement`
**REST**: `DELETE /api/v1/notify/announcement/{id}`

### NotifyFailedToChargeBookings
Notify about failed payment charges.

**gRPC**: `NotificationService.NotifyFailedToChargeBookings`
**REST**: `POST /api/v1/notify/failed-charges`

### DeleteBBOneUserRecipientId
Delete BBOne user recipient ID for GDPR compliance.

**gRPC**: `NotificationService.DeleteBBOneUserRecipientId`
**REST**: `DELETE /api/v1/bbone/recipient/{user_id}`

**Use Case**: MBMD-12474 - GDPR data deletion

---

## Health Check

### HealthCheck
Check service health status.

**gRPC**: `NotificationService.HealthCheck`
**REST**: `GET /health`

**Response**:
```json
{
  "code": 200,
  "message": "Service is healthy"
}
```

---

## Error Codes

| Code | Status | Description |
|------|--------|-------------|
| 200 | OK | Request successful |
| 400 | Bad Request | Invalid request parameters |
| 401 | Unauthorized | Missing or invalid authentication |
| 404 | Not Found | Resource not found |
| 429 | Too Many Requests | Rate limit exceeded |
| 500 | Internal Server Error | Server error |
| 501 | Not Implemented | Feature not supported by current strategy |
| 503 | Service Unavailable | Dependent service unavailable |

## Rate Limiting

### Per-User Limits
- **OTP**: 3 requests per 5 minutes
- **Push Notifications**: 100 per minute
- **Email**: 50 per hour
- **SMS**: 10 per hour

### Duplicate Request Prevention
- Menggunakan Redis lock dengan 10 second TTL
- Request hash berdasarkan protobuf message content
- Key format: `DUPLICATE_LOCK:notification-center:{function_name}:{request_hash}`

---

## Related Documentation
- [[01-overview|Service Overview]]
- [[02-email-strategy|Email Strategy Pattern]]
- [[03-rule-engine|Rule Engine Architecture]]
- [Swagger Documentation](http://notificationcenter:8080/swagger)

---
**Last Updated**: 2025-01-07
**API Version**: v1 (legacy), v2 (email strategy)
**Protocol**: gRPC + REST Gateway