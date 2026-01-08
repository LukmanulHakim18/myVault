---
tags:
  - api
  - grpc
  - fds
  - fraud-detection
  - mrg
  - reference
type: api-documentation
title: FDS Service - API Reference
parent: fds
---
# FDS Service - API Reference

**Service**: [[README|FDS Service]]  
**Type**: API Documentation

---

## 📡 Server Configuration

| Protocol | Port | Default |
|----------|------|---------|
| gRPC | `GRPC_PORT` | 6001 / 6101 |
| REST (gRPC-Gateway) | `REST_PORT` | 8001 / 8101 |

---

## 🔧 gRPC Methods

### Health Check

#### HealthCheck
```protobuf
rpc HealthCheck(EmptyRequest) returns (DefaultResponse);
```
- **Input**: `{}`
- **Output**: `{code, message}`

---

### Soft Ban Operations

#### GetSoftBanned
Get all soft bans for an identifier.
```protobuf
rpc GetSoftBanned(SoftBannedRequest) returns (GetSoftBannedListResponse);
```
- **Interceptors**: ValidateInput
- **Input**:
  ```protobuf
  message SoftBannedRequest {
    string identifier = 1;  // BBID or phone number
  }
  ```
- **Output**:
  ```protobuf
  message GetSoftBannedListResponse {
    string identifier = 1;
    bool is_banned = 2;
    repeated SoftBannedDetail banned_detail = 3;
    string message = 4;
  }
  
  message SoftBannedDetail {
    string banned_type = 1;
    string banned_reason = 2;
    string ban_start_time = 3;
    string ban_end_time = 4;
  }
  ```

#### GetSoftBannedByType
Get specific soft ban by type for an identifier.
```protobuf
rpc GetSoftBannedByType(GetSoftBannedByTypeRequest) returns (GetSoftBannedResponse);
```
- **Interceptors**: ValidateInput
- **Input**:
  ```protobuf
  message GetSoftBannedByTypeRequest {
    string banned_type = 1;  // CANCEL_ORDER, OTP_ATTEMPT, WRONG_PASSWORD, CANCEL_ORDER_AFTER_DISPATCH
    string identifier = 2;
  }
  ```
- **Output**:
  ```protobuf
  message GetSoftBannedResponse {
    string identifier = 1;
    bool is_banned = 2;
    SoftBannedDetail banned_detail = 3;
    string message = 4;
  }
  ```

#### SoftBannedCounter
Increment attempt counter for soft ban. Will trigger ban if threshold exceeded.
```protobuf
rpc SoftBannedCounter(SoftBannedByTypeRequest) returns (DefaultResponse);
```
- **Interceptors**: ValidateInput
- **Input**:
  ```protobuf
  message SoftBannedByTypeRequest {
    string identifier = 1;
    string banned_type = 2;
    SoftBannedProperties properties = 3;
  }
  
  message SoftBannedProperties {
    string city = 1;
  }
  ```
- **Output**: `{code, message}`

**Usage Example:**
```json
POST /soft-banned
{
  "identifier": "BB123123",
  "banned_type": "CANCEL_ORDER",
  "properties": {
    "city": "Jakarta"
  }
}
```

#### RevokeSoftBanned
Remove soft ban for an identifier.
```protobuf
rpc RevokeSoftBanned(RevokeSoftBannedRequest) returns (DefaultResponse);
```
- **Interceptors**: ValidateInput
- **Input**:
  ```protobuf
  message RevokeSoftBannedRequest {
    string identifier = 1;
    string banned_type = 2;
  }
  ```
- **Output**: `{code, message}`

**Usage Example:**
```json
POST /revoke-soft-banned
{
  "identifier": "BB123123",
  "banned_type": "CANCEL_ORDER"
}
```

---

### Hard Ban Operations

#### SetHardBanned
Create a hard ban for an identifier.
```protobuf
rpc SetHardBanned(HardBanned) returns (DefaultResponse);
```
- **Interceptors**: ValidateInput
- **Input**:
  ```protobuf
  message HardBanned {
    int64 id = 1;
    string identifier = 2;          // BBID, phone number, or device ID
    string login_token = 3;         // For force logout
    string identifier_type = 4;     // BBID, PHONE_NUMBER, DEVICE_ID
    string banned_reason_id = 5;    // Reason in Indonesian
    string banned_reason_en = 6;    // Reason in English
  }
  ```
- **Output**: `{code, message}`

#### CheckHardBanned
Check if an identifier is hard banned.
```protobuf
rpc CheckHardBanned(CheckHardBannedRequest) returns (CheckHardBannedResponse);
```
- **Interceptors**: ValidateInput
- **Input**:
  ```protobuf
  message CheckHardBannedRequest {
    string identifier = 1;
  }
  ```
- **Output**:
  ```protobuf
  message CheckHardBannedResponse {
    string identifier = 2;
    HardBanned banned_detail = 1;
    bool is_banned = 3;
    string message = 4;
  }
  ```

#### GetHardBanned
Get hard ban details by ID.
```protobuf
rpc GetHardBanned(RequestId) returns (HardBanned);
```
- **Interceptors**: ValidateInput
- **Input**:
  ```protobuf
  message RequestId {
    int64 Id = 1;
  }
  ```
- **Output**: `HardBanned`

#### RevokeHardBanned
Remove hard ban by ID.
```protobuf
rpc RevokeHardBanned(RequestId) returns (DefaultResponse);
```
- **Interceptors**: ValidateInput
- **Input**: `{Id: int64}`
- **Output**: `{code, message}`

**Usage Example:**
```json
POST /revoke-hard-banned
{
  "Id": 9
}
```

---

### Phone Number Operations

#### ScanFraudPhoneNumber
Check if a phone number is banned.
```protobuf
rpc ScanFraudPhoneNumber(ScanRequest) returns (ScanResponse);
```
- **Input**:
  ```protobuf
  message ScanRequest {
    string phone_number = 1;
  }
  ```
- **Output**:
  ```protobuf
  message ScanResponse {
    bool phone_number_is_banned = 1;
  }
  ```

#### WhitelistPhoneNumbers
Add phone numbers to whitelist (exclude from banning).
```protobuf
rpc WhitelistPhoneNumbers(WhitelistRequest) returns (WhitelistPhoneNumberResponse);
```
- **Interceptors**: ValidateInput
- **Input**:
  ```protobuf
  message WhitelistRequest {
    repeated string phone_numbers = 1;
  }
  ```
- **Output**:
  ```protobuf
  message WhitelistPhoneNumberResponse {
    bool success_add_phone_number = 1;
  }
  ```

---

## 📝 Common Types

### DefaultResponse
```protobuf
message DefaultResponse {
  string code = 1;
  string message = 2;
}
```

### HardBanned
```protobuf
message HardBanned {
  int64 id = 1;
  string identifier = 2;
  string login_token = 3;
  string identifier_type = 4;  // BBID, PHONE_NUMBER, DEVICE_ID
  string banned_reason_id = 5;
  string banned_reason_en = 6;
}
```

### SoftBannedDetail
```protobuf
message SoftBannedDetail {
  string banned_type = 1;     // CANCEL_ORDER, OTP_ATTEMPT, WRONG_PASSWORD, CANCEL_ORDER_AFTER_DISPATCH
  string banned_reason = 2;
  string ban_start_time = 3;
  string ban_end_time = 4;
}
```

---

## 📊 Soft Ban Types

| Type | Description | Threshold | Duration |
|------|-------------|-----------|----------|
| `CANCEL_ORDER` | User cancels order too frequently | 5 in 2h | 24h ban |
| `CANCEL_ORDER_AFTER_DISPATCH` | Cancel after driver dispatched | 5 in 2h | 24h ban |
| `OTP_ATTEMPT` | Too many OTP requests | 3 in 24h | 24h ban |
| `WRONG_PASSWORD` | Too many wrong password attempts | 5 in 2m | 5m ban |

---

## 📊 Identifier Types

| Type | Description | Example |
|------|-------------|---------|
| `BBID` | Bluebird User ID | `BB00123456` |
| `PHONE_NUMBER` | User's phone number | `+6281234567890` |
| `DEVICE_ID` | Mobile device identifier | `abc123-def456` |

---

## 🏷️ Tags

#api #grpc #fds #fraud-detection #mrg #reference

---

*Last Updated*: 2025-01-05
