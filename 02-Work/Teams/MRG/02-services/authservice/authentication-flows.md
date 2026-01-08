---
tags:
  - authservice
  - authentication
  - flow
  - security
  - mrg
type: flow-documentation
title: Auth Service - Authentication Flows
parent: authservice
---
# Auth Service - Authentication Flows

**Service**: [[README|Auth Service]]  
**Type**: Flow Documentation

---

## 🔄 Flow Overview

AuthService mendukung berbagai authentication flows untuk different user scenarios. Setiap flow dirancang dengan security berlapis dan user experience yang smooth.

---

## 1️⃣ New User Registration Flow

**Scenario**: User baru ingin mendaftar ke aplikasi MyBB

### Sequence Diagram

```mermaid
sequenceDiagram
    participant Client
    participant AuthService
    participant FDS
    participant UserService
    participant Redis
    participant Notification

    Client->>AuthService: 1. ValidateUser(phone_number)
    AuthService->>FDS: Check fraud
    AuthService->>UserService: User exists?
    UserService-->>AuthService: is_registered: false
    AuthService->>Redis: Store session
    AuthService-->>Client: {session_token, otp_list}

    Client->>AuthService: 2. SendOTP(session_token, WHATSAPP)
    AuthService->>Redis: Generate & cache OTP
    AuthService->>Notification: Send WhatsApp OTP
    AuthService-->>Client: {verification_token, resend_otp_time: 60}

    Note over Client: User receives OTP

    Client->>AuthService: 3. ValidateOTP(verification_token, "123456")
    AuthService->>Redis: Compare OTP
    AuthService->>Redis: Generate register_token
    AuthService-->>Client: {register_token}

    Client->>AuthService: 4. RegisterUser(register_token, name, email, password)
    AuthService->>Redis: Validate register_token
    AuthService->>UserService: Create user account
    AuthService->>AuthService: Generate tokens
    AuthService->>Redis: Store tokens
    AuthService-->>Client: {user_profile, access_token, refresh_token}
```

### Key Points
- **4-step process**: Validate → SendOTP → ValidateOTP → Register
- **Auto-login**: User automatically logged in after registration
- **OTP expiration**: Default 5 minutes
- **Multi-channel OTP**: User can choose WhatsApp, SMS, or Email

---

## 2️⃣ Existing User Login Flow

**Scenario**: User yang sudah terdaftar ingin login

### Sequence Diagram

```mermaid
sequenceDiagram
    participant Client
    participant AuthService
    participant UserService
    participant Redis

    Client->>AuthService: 1. ValidateUser(phone_number)
    AuthService->>UserService: User exists?
    UserService-->>AuthService: is_registered: true
    AuthService->>Redis: Store session_token
    AuthService-->>Client: {is_registered: true, session_token}

    Client->>AuthService: 2. Login(session_token, password)
    AuthService->>Redis: Get phone from session
    AuthService->>UserService: Authenticate(phone, password)
    
    alt Success
        AuthService->>Redis: Reset wrong password counter
        AuthService->>AuthService: Generate tokens
        AuthService->>Redis: Store tokens & delete session
        AuthService-->>Client: {profile, access_token, refresh_token}
    else Wrong Password (< 5 attempts)
        AuthService->>Redis: Increment counter
        AuthService-->>Client: Error: "Wrong password. X attempts remaining"
    else Max Attempts Reached
        AuthService->>Redis: Ban user 24h, delete session
        AuthService-->>Client: Error: "Account locked for 24 hours"
    end
```

### Wrong Password Flow
```
Attempt 1: Wrong → "4 attempts remaining"
Attempt 2: Wrong → "3 attempts remaining"
Attempt 3: Wrong → "2 attempts remaining"
Attempt 4: Wrong → "1 attempt remaining"
Attempt 5: Wrong → "Account locked for 24 hours"
```

---

## 3️⃣ Token Refresh Flow

**Scenario**: Access token sudah expired, perlu token baru tanpa login ulang

### Sequence Diagram

```mermaid
sequenceDiagram
    participant Client
    participant AuthService
    participant Redis

    Client->>AuthService: API Request (expired access_token)
    AuthService-->>Client: Error: "Token expired"

    Client->>AuthService: RefreshToken(access_token, refresh_token)
    AuthService->>Redis: Validate refresh_token exists
    AuthService->>AuthService: Parse old access_token (extract user_id)
    AuthService->>AuthService: Generate new access_token
    AuthService->>Redis: Store new token, extend refresh TTL
    AuthService-->>Client: {new_access_token, same_refresh_token}
```

### Key Points
- **Seamless experience**: User tidak perlu login ulang
- **Refresh token reuse**: Same refresh token, new access token
- **TTL extension**: Refresh token TTL diperpanjang

---

## 4️⃣ Logout Flow

**Scenario**: User ingin logout dari aplikasi

### Sequence Diagram

```mermaid
sequenceDiagram
    participant Client
    participant AuthService
    participant Redis

    Client->>AuthService: Logout(access_token)
    AuthService->>AuthService: Validate & parse access_token
    AuthService->>Redis: Blacklist access_token
    AuthService->>Redis: Delete refresh_token
    AuthService-->>Client: {code: "SUCCESS"}
```

---

## 5️⃣ Forgot Password Flow

**Scenario**: User lupa password dan ingin reset

### Sequence Diagram

```mermaid
sequenceDiagram
    participant Client
    participant AuthService
    participant UserService
    participant Redis
    participant Email

    Client->>AuthService: 1. ValidateUser(email)
    AuthService->>UserService: Check email exists
    AuthService->>Redis: Store session_token
    AuthService-->>Client: {session_token}

    Client->>AuthService: 2. ForgotPassword(session_token, email)
    AuthService->>Redis: Get user from session
    AuthService->>AuthService: Generate reset_password_token
    AuthService->>Redis: Store reset token (TTL=1h)
    AuthService->>Email: Send reset link
    AuthService-->>Client: {code: "SUCCESS"}

    Note over Client: User clicks email link

    Client->>AuthService: 3. ResetPassword(reset_token, new_password)
    AuthService->>Redis: Validate reset_token
    AuthService->>UserService: Update password
    AuthService->>Redis: Delete reset token (one-time use)
    AuthService->>Redis: Revoke ALL refresh tokens
    AuthService-->>Client: {code: "SUCCESS"}
```

### Key Points
- **3-step process**: Validate → ForgotPassword → ResetPassword
- **Time-limited**: Reset token expires in 1 hour
- **One-time use**: Token deleted after successful reset
- **Security**: All sessions invalidated after password reset

---

## 6️⃣ Change Password Flow

**Scenario**: Logged-in user ingin ganti password

### Sequence Diagram

```mermaid
sequenceDiagram
    participant Client
    participant AuthService
    participant UserService
    participant Redis

    Client->>AuthService: ChangePassword(old_password, new_password)
    Note right of Client: [access_token in metadata]
    
    AuthService->>AuthService: Validate access_token
    AuthService->>UserService: Change password (validates old)
    AuthService->>Redis: Revoke ALL refresh tokens (except current)
    AuthService-->>Client: {code: "SUCCESS"}
```

### Key Points
- **Authentication required**: Must have valid access_token
- **Partial logout**: All OTHER sessions invalidated, current session remains

---

## 7️⃣ Multi-Device Logout Flow

**Scenario**: User ingin logout dari semua devices

### Sequence Diagram

```mermaid
sequenceDiagram
    participant Client
    participant AuthService
    participant Redis

    Client->>AuthService: RevokeAllRefreshToken()
    Note right of Client: [access_token in metadata]
    
    AuthService->>AuthService: Validate access_token
    AuthService->>Redis: Delete ALL refresh tokens for user_id
    AuthService-->>Client: {code: "SUCCESS"}
```

### Use Cases
- Password changed (security measure)
- Suspicious activity detected
- Device lost/stolen

---

## 📊 Flow Comparison Matrix

| Flow | Steps | Duration | Auth Required | External Services |
|------|-------|----------|---------------|-------------------|
| **Registration** | 4 | ~2-5 min | ❌ | User, FDS, Notification |
| **Login** | 2 | ~5 sec | ❌ | User |
| **Token Refresh** | 1 | ~1 sec | ✅ (refresh) | - |
| **Logout** | 1 | ~1 sec | ✅ | - |
| **Forgot Password** | 3 | ~5-10 min | ❌ | User, Notification |
| **Change Password** | 1 | ~3 sec | ✅ | User |
| **Revoke All** | 1 | ~1 sec | ✅ | - |

---

## 🔒 Security Measures

### Rate Limiting
| Action | Limit | Cooldown |
|--------|-------|----------|
| OTP sends | Max 5 per session | 60 seconds |
| Login attempts | Max 5 wrong | 24 hours ban |

### Token Expiration
| Token Type | TTL |
|------------|-----|
| Access Token | 2 hours |
| Refresh Token | 16 hours |
| Session Token | 15 minutes |
| Reset Token | 1 hour |
| OTP | 5 minutes |

---

## 🏷️ Tags

#authservice #authentication #flow #security #mrg

---

*Last Updated*: 2025-01-05
