---
tags:
  - api
  - grpc
  - authservice
  - mrg
  - reference
type: api-documentation
title: Auth Service - API Reference
parent: authservice
---
# Auth Service - API Reference

**Service**: [[README|Auth Service]]  
**Type**: API Documentation

---

## 📡 Server Configuration

| Protocol | Port | Default |
|----------|------|---------|
| gRPC | `GRPC_PORT` | 6017 |
| REST (gRPC-Gateway) | `REST_PORT` | 8017 |
| Swagger | `SWAGGER_PORT` | 9017 |

---

## 🔧 gRPC Methods

### Health & Utility

#### HealthCheck
```protobuf
rpc HealthCheck(EmptyRequest) returns (DefaultResponse);
```

#### GetTimeSync
```protobuf
rpc GetTimeSync(EmptyRequest) returns (GetTimeSyncResponse);
```
- **Output**: `{epoch_server_time: uint32}`

---

### Token Management

#### CreateToken
```protobuf
rpc CreateToken(CreateTokenRequest) returns (Token);
```
- **Interceptors**: InputValidation
- **Input**:
  ```protobuf
  message CreateTokenRequest {
    string user_id = 1;
    string email = 2;
    string name = 3;
  }
  ```
- **Output**:
  ```protobuf
  message Token {
    string access_token = 1;
    string refresh_token = 2;
  }
  ```

#### RefreshToken
```protobuf
rpc RefreshToken(Token) returns (Token);
```
- **Interceptors**: InputValidation, MetadataAfterLoginValidation
- **Input**: `{access_token, refresh_token}`
- **Output**: `{new_access_token, refresh_token}`

#### ValidateToken
```protobuf
rpc ValidateToken(Token) returns (DefaultResponse);
```
- **Interceptors**: InputValidation
- **Input**: `{access_token}`
- **Output**: `{code, message}`

---

### User Validation & OTP

#### ValidateUser
```protobuf
rpc ValidateUser(ValidateUserRequest) returns (ValidateUserResponse);
```
- **Interceptors**: InputValidation, MetadataValidation, FormatPhoneNumber
- **Input**:
  ```protobuf
  message ValidateUserRequest {
    string country_code = 1;
    string login_id = 2;
    string login_type = 3;
    string region_code = 4;
  }
  ```
- **Output**:
  ```protobuf
  message ValidateUserResponse {
    bool is_registered = 1;
    string verification_token = 2;
    string session_token = 3;
    uint32 resend_otp_time = 4;
  }
  ```

#### ValidateUserWithProviderList
```protobuf
rpc ValidateUserWithProviderList(ValidateUserRequest) returns (ValidateUserWithOtpListResponse);
```
- **Interceptors**: InputValidation, MetadataValidation, FormatPhoneNumber
- **Output**: Same as ValidateUser + `otp_list: [OTPTypeResponse]`

#### SendOtp
```protobuf
rpc SendOtp(SendOTPRequest) returns (ValidateUserResponse);
```
- **Interceptors**: OTT, InputValidation, MetadataValidation
- **Input**:
  ```protobuf
  enum OtpType {
    WHATSAPP = 0;
    SMS = 1;
    EMAIL = 2;
  }
  
  message SendOTPRequest {
    string verification_token = 1;
    OtpType type = 2;
    string login_id = 3;
  }
  ```

#### ValidateOTP
```protobuf
rpc ValidateOTP(ValidateOTPRequest) returns (ValidateOTPResponse);
```
- **Interceptors**: InputValidation, MetadataValidation
- **Input**:
  ```protobuf
  message ValidateOTPRequest {
    string verification_token = 1;
    string verification_code = 2;
    OtpType type = 3;
  }
  ```
- **Output**: `{register_token: string}`

---

### Authentication

#### RegisterUser
```protobuf
rpc RegisterUser(RegisterRequest) returns (ProfileResponse);
```
- **Interceptors**: InputValidation, MetadataValidation
- **Input**:
  ```protobuf
  message RegisterRequest {
    string register_token = 1;
    string name = 2;
    string email = 3;
    string password = 4;
    double latitude = 5;
    double longitude = 6;
    string device_token = 7;
    string invited_referral_code = 8;
    string referral_code = 9;
  }
  ```
- **Output**: `ProfileResponse` (full profile + tokens)

#### Login
```protobuf
rpc Login(LoginRequest) returns (ProfileResponse);
```
- **Interceptors**: InputValidation, MetadataValidation
- **Input**:
  ```protobuf
  message LoginRequest {
    string session_token = 1;
    string password = 2;
  }
  ```
- **Output**: `ProfileResponse`

#### Logout
```protobuf
rpc Logout(Token) returns (DefaultResponse);
```
- **Interceptors**: InputValidation, MetadataAfterLoginValidation
- **Input**: `{access_token}`

#### RevokeAllRefreshToken
```protobuf
rpc RevokeAllRefreshToken(EmptyRequest) returns (DefaultResponse);
```
- **Interceptors**: MetadataAfterLoginValidation
- **Description**: Logout from all devices

---

### Password Management

#### ChangePassword
```protobuf
rpc ChangePassword(ChangePasswordRequest) returns (DefaultResponse);
```
- **Interceptors**: InputValidation, MetadataAfterLoginValidation
- **Input**:
  ```protobuf
  message ChangePasswordRequest {
    string old_password = 1;
    string new_password = 2;
  }
  ```

#### ForgotPassword
```protobuf
rpc ForgotPassword(ForgotPasswordRequest) returns (DefaultResponse);
```
- **Interceptors**: InputValidation, MetadataAfterLoginValidation
- **Input**:
  ```protobuf
  message ForgotPasswordRequest {
    string session_token = 1;
    string email = 2;
  }
  ```

#### ResetPassword
```protobuf
rpc ResetPassword(ResetPasswordRequest) returns (DefaultResponse);
```
- **Interceptors**: InputValidation
- **Input**:
  ```protobuf
  message ResetPasswordRequest {
    string reset_password_token = 1;
    string new_password = 2;
  }
  ```

---

### Migration & Internal

#### MigrateToken
```protobuf
rpc MigrateToken(MigrateTokenRequest) returns (Token);
```
- **Interceptors**: InputValidation
- **Description**: Migrate legacy token to new format
- **Input**:
  ```protobuf
  message MigrateTokenRequest {
    string old_token = 1;
    string device_token = 2;
  }
  ```

#### EncryptPass / DecryptPass
```protobuf
rpc EncryptPass(EncPassword) returns (DefaultResponse);
rpc DecryptPass(EncPassword) returns (DefaultResponse);
```
- **Description**: Internal password encryption utilities

---

## 📝 Common Types

### ProfileResponse
```protobuf
message ProfileResponse {
  string user_id = 1;
  string email = 2;
  string name = 3;
  string phone = 4;
  string profile_image = 5;
  bool is_legacy = 6;
  int64 created_at = 7;
  int64 updated_at = 8;
  bool is_activated = 9;
  string firebase_token = 10;
  string subscribe_error = 11;
  repeated string privileges = 12;
  int64 points = 13;
  bool is_verified_email = 14;
  bool is_white_list_user = 15;
  string access_token = 16;
  string refresh_token = 17;
}
```

### OTPTypeResponse
```protobuf
message OTPTypeResponse {
  int32 type_id = 1;
  string name = 2;
  int32 order_priority = 3;
  string message = 4;
  bool need_callback = 5;
}
```

### DefaultResponse
```protobuf
message DefaultResponse {
  string code = 1;
  string message = 2;
}
```

---

## 🏷️ Tags

#api #grpc #authservice #mrg #reference

---

*Last Updated*: 2025-01-05
