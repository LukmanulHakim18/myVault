---
tags:
  - api
  - grpc
  - userservice
  - mrg
  - reference
type: api-documentation
title: User Service - API Reference
parent: userservice
---
# User Service - API Reference

**Service**: [[README|User Service]]  
**Type**: API Documentation

---

## 📡 Server Configuration

| Protocol | Port | Default |
|----------|------|---------|
| gRPC | `GRPC_PORT` | 6015 |
| REST (gRPC-Gateway) | `REST_PORT` | 8015 |

---

## 🔧 gRPC Methods

### Health & Utility

#### HealthCheck
```protobuf
rpc HealthCheck(Ping) returns (Ping);
```

---

### Profile Management

#### GetProfile
```protobuf
rpc GetProfile(ProfileRequest) returns (ProfileResponse);
```
- **Input**:
  ```protobuf
  message ProfileRequest {
    string user_id = 1;
  }
  ```
- **Output**: `ProfileResponse` (see Common Types)

#### UpdateProfile
```protobuf
rpc UpdateProfile(UpdateProfileRequest) returns (UpdateProfileResponse);
```
- **Interceptors**: ValidateMetadata
- **Input**:
  ```protobuf
  message UpdateProfileRequest {
    string name = 1;
    string phone_number = 2;
    string country_code = 3;
    string email = 4;
    File file = 5;
    string gender = 6;
    string dob = 7;
    int32 domicile_id = 8;
    int32 occupation_id = 9;
    int32 hobby_id = 10;
  }
  ```
- **Output**:
  ```protobuf
  message UpdateProfileResponse {
    string token = 1;
    string email = 2;
    string name = 3;
    string phone = 4;
    string profile_image = 5;
    bool is_activated = 6;
    int32 verification_id = 7;
    int32 verification_resendable_in_sec = 8;
  }
  ```

#### Verify
```protobuf
rpc Verify(VerifyUpdateProfileRequest) returns (ProfileResponse);
```
- **Input**:
  ```protobuf
  message VerifyUpdateProfileRequest {
    int32 verification_id = 1;
    string code = 2;
  }
  ```

#### ResendVerificationUpdateProfile
```protobuf
rpc ResendVerificationUpdateProfile(ResendVerificationRequest) returns (ResendVerificationResponse);
```
- **Input**: `{verification_id: int64}`
- **Output**: `{code, message, data: {resendable_in_sec}}`

#### UpdateLanguage
```protobuf
rpc UpdateLanguage(DefaultRequest) returns (DefaultResponse);
```
- **Input**: `{request: string}` (language code)

#### RequestVerificationEmail
```protobuf
rpc RequestVerificationEmail(EmailRequest) returns (RequestVerificationEmailResponse);
```
- **Input**: `{email: string}`
- **Output**:
  ```protobuf
  message RequestVerificationEmailResponse {
    string token = 1;
    string email = 2;
    string name = 3;
    string phone = 4;
    int32 verification_id = 5;
    int32 verification_resendable_in_sec = 6;
    string deleted_at = 7;
    string loyalty_id = 8;
  }
  ```

---

### User Management

#### GetUser
```protobuf
rpc GetUser(FindUserRequest) returns (UserResponse);
```
- **Input**:
  ```protobuf
  enum FindBy {
    internal_id = 0;
    phone_number = 1;
    email = 2;
  }
  
  message FindUserRequest {
    FindBy find_by = 1;
    string value = 2;
  }
  ```
- **Output**: `UserResponse` (see Common Types)

#### GetUserInternalIdByPhoneNumber
```protobuf
rpc GetUserInternalIdByPhoneNumber(GetUserInternalIdByPhoneNumberRequest) returns (GetUserInternalIdByPhoneNumberResponse);
```
- **Interceptors**: ValidateInput
- **Input**: `{phone_number: string}`
- **Output**: `{internal_id: string}`

#### CreateUser
```protobuf
rpc CreateUser(CreateUserRequest) returns (ProfileResponse);
```
- **Interceptors**: DeviceVersion, Token
- **Input**:
  ```protobuf
  message CreateUserRequest {
    string name = 1;
    string phone_number = 2;
    string email = 3;
    string password = 4;
    string referral_code = 5;
    double latitude = 6;
    double longitude = 7;
  }
  ```

#### Login
```protobuf
rpc Login(LoginRequest) returns (ProfileResponse);
```
- **Input**: `{phone_number, password}`

#### ChangePassword
```protobuf
rpc ChangePassword(ChangePasswordReq) returns (ChangePasswordResponse);
```
- **Input**: `{old_password, new_password}`
- **Output**: `{code, message, success}`

#### ResetPassword
```protobuf
rpc ResetPassword(ResetPasswordReq) returns (DefaultResponse);
```
- **Input**: `{bb_id, new_password}`

#### DeleteAccount
```protobuf
rpc DeleteAccount(DeleteAccountRequest) returns (DeleteAccountResponse);
```
- **Interceptors**: ValidateInput, ValidateMetadata
- **Input**: `{password, reason_deleted}`
- **Output**: `{success: bool}`

#### IsUserAllowedToDeleteAccount
```protobuf
rpc IsUserAllowedToDeleteAccount(EmptyRequest) returns (IsUserAllowedToDeleteAccountResponse);
```
- **Interceptors**: ValidateMetadata
- **Output**: `{is_user_allowed_to_delete_account: bool}`

---

### Favorite Address (V1)

#### ListFavoriteAddress
```protobuf
rpc ListFavoriteAddress(PageReq) returns (FavoriteAddressResp);
```
- **Input**: `{page, per_page}`
- **Output**:
  ```protobuf
  message FavoriteAddressResp {
    MetaFavoriteAddress meta = 1;  // {is_full: bool}
    repeated FavoriteAddress data = 2;
  }
  ```

#### CreateFavoriteAddress
```protobuf
rpc CreateFavoriteAddress(FavoriteAddressReq) returns (FavoriteAddress);
```
- **Input**:
  ```protobuf
  message FavoriteAddressReq {
    int32 favorite_address_id = 1;
    string tag = 2;
    string name = 3;
    int32 position = 4;
    string icon = 5;
    bool bookmark = 6;
    double latitude = 7;
    double longitude = 8;
    string display_address = 9;
    string color = 10;
    string note_to_driver = 11;
    string place_id = 12;
  }
  ```

#### UpdateFavoriteAddress
```protobuf
rpc UpdateFavoriteAddress(FavoriteAddressReq) returns (FavoriteAddress);
```

#### DeleteFavoriteAddress
```protobuf
rpc DeleteFavoriteAddress(FavoriteAddressId) returns (DefaultResponse);
```
- **Interceptors**: CustomHttpCodeResponseNoContent
- **Input**: `{favorite_address_id: int32}`

---

### Favorite Address (V2)

#### GetFavoriteAddressesV2
```protobuf
rpc GetFavoriteAddressesV2(GetFavoriteAddressesV2Req) returns (GetFavoriteAddressesV2Resp);
```
- **Input**: `{page, per_page}`
- **Output**:
  ```protobuf
  message GetFavoriteAddressesV2Resp {
    GetFavoriteAddressesV2Resp_Meta meta = 1;  // {is_full: bool}
    repeated FavoriteAddressV2 data = 2;
  }
  
  message FavoriteAddressV2 {
    int32 favorite_address_id = 1;
    string bb_id = 2;
    string location_name = 3;
    string address_short_name = 4;
    string address_long_name = 5;
    double latitude = 6;
    double longitude = 7;
    string driver_notes = 8;
    int32 position = 9;
    string place_id = 10;
    string icon = 11;
  }
  ```

#### CreateFavoriteAddressV2
```protobuf
rpc CreateFavoriteAddressV2(CreateFavoriteAddressV2Req) returns (CreateFavoriteAddressV2Resp);
```
- **Input**:
  ```protobuf
  message CreateFavoriteAddressV2Req {
    string bb_id = 1;
    string location_name = 2;
    string address_short_name = 3;
    string address_long_name = 4;
    double latitude = 5;
    double longitude = 6;
    string driver_notes = 7;
    string place_id = 8;
  }
  ```

#### UpdateFavoriteAddressV2
```protobuf
rpc UpdateFavoriteAddressV2(UpdateFavoriteAddressV2Req) returns (UpdateFavoriteAddressV2Resp);
```

#### DeleteFavoriteAddressV2
```protobuf
rpc DeleteFavoriteAddressV2(DeleteFavoriteAddressV2Req) returns (DeleteFavoriteAddressV2Resp);
```
- **Interceptors**: CustomHttpCodeResponseNoContent
- **Input**: `{bb_id, favorite_address_id}`
- **Output**: `{success: bool}`

#### UpdateFavoriteAddressesPosition
```protobuf
rpc UpdateFavoriteAddressesPosition(UpdateFavoriteAddressesPositionReq) returns (UpdateFavoriteAddressesPositionResp);
```
- **Input**:
  ```protobuf
  message UpdateFavoriteAddressesPositionReq {
    string bb_id = 1;
    repeated FavoriteAddressPosition favorite_address_position = 2;
  }
  
  message FavoriteAddressPosition {
    int32 favorite_address_id = 1;
    int32 position = 2;
  }
  ```

#### GetSearchLocationHistory
```protobuf
rpc GetSearchLocationHistory(GetSearchLocationHistoryReq) returns (GetSearchLocationHistoryResp);
```
- **Input**:
  ```protobuf
  message GetSearchLocationHistoryReq {
    string bb_id = 1;
    enum Type { pickup = 0; dropoff = 1; }
    Type type = 2;
  }
  ```
- **Output**:
  ```protobuf
  message GetSearchLocationHistoryResp {
    string bb_id = 1;
    repeated SearchLocationHistory search_location_history = 2;
  }
  
  message SearchLocationHistory {
    string address_short_name = 1;
    string address_long_name = 2;
    double latitude = 3;
    double longitude = 4;
    int32 favorite_address_id = 5;
    bool is_favorite_address = 6;
    string note_to_driver = 7;
  }
  ```

---

### Favorite Trip

#### AddFavoriteTrip
```protobuf
rpc AddFavoriteTrip(FavoriteTripRequest) returns (DefaultResponse);
```
- **Interceptors**: ValidateInput, ValidateMetadata
- **Input**: `{order_id: string}`

#### GetFavoriteTrip
```protobuf
rpc GetFavoriteTrip(FavoriteTripRequest) returns (FavoriteTripResponse);
```
- **Interceptors**: ValidateMetadata
- **Output**:
  ```protobuf
  message FavoriteTripResponse {
    repeated FavoriteTrip favorite_trips = 1;
  }
  
  message FavoriteTrip {
    int64 id = 1;
    LocationDetails pickup = 2;
    LocationDetails dropoff = 3;
    string payment_method_name = 4;
    string product_type = 5;
    string category = 6;
    string fleet_icon = 7;
    string order_id = 8;
  }
  ```

#### DeleteFavoriteTrip
```protobuf
rpc DeleteFavoriteTrip(FavoriteTripIdRequest) returns (DefaultResponse);
```
- **Interceptors**: ValidateInput, ValidateMetadata
- **Input**: `{id: int64}`

---

### Payment Method

#### GetPaymentMethodUser
```protobuf
rpc GetPaymentMethodUser(PaymentMethodUserRequest) returns (PaymentMethodUserResponse);
```
- **Interceptors**: PaymentMethod
- **Output**: `PaymentMethodUserResponse` (see Common Types)

#### GetOrderPaymentMethodUser
```protobuf
rpc GetOrderPaymentMethodUser(OrderPaymentMethodUserRequest) returns (PaymentMethodUserResponse);
```
- **Interceptors**: PaymentMethod
- **Input**: `{product_type, service_type}`

#### GetDefaultPaymentMethodUser
```protobuf
rpc GetDefaultPaymentMethodUser(OrderPaymentMethodUserRequest) returns (PaymentMethodUserResponse);
```
- **Interceptors**: PaymentMethod

#### GetDefaultPaymentMethodSubscription
```protobuf
rpc GetDefaultPaymentMethodSubscription(PaymentMethodUserRequest) returns (PaymentMethodUserResponse);
```
- **Interceptors**: PaymentMethod

#### UpsertPaymentMethodUser
```protobuf
rpc UpsertPaymentMethodUser(UpsertPaymentMethodUserRequest) returns (DefaultResponse);
```
- **Input**:
  ```protobuf
  message UpsertPaymentMethodUserRequest {
    message Data {
      string internal_id = 1;
      string payment_method_type = 2;
      string payment_method_display = 3;
      string identifier = 4;
      string type_cc = 5;
      string company_id = 6;
      string Languange = 7;
      string CustomerEmail = 8;
      string CustomerName = 9;
      string CustomerPhoneNumber = 10;
      string CardNumber = 11;
      string BudgetAmount = 12;
      string TotalTrip = 13;
      string CompanyName = 14;
    }
    repeated Data data = 1;
  }
  ```

#### DeletePaymentMethodUser
```protobuf
rpc DeletePaymentMethodUser(DeletePaymentMethodUserRequest) returns (DefaultResponse);
```
- **Input**: `{internal_id, payment_method_type, identifier}`

#### UpdateDefaultPayment
```protobuf
rpc UpdateDefaultPayment(UpdateDefaultPaymentReq) returns (DefaultResponse);
```
- **Input**: `{payment_method, is_default, identifier}`

---

### Navigation State

#### GetNavigationUserState
```protobuf
rpc GetNavigationUserState(NavigationUserState) returns (NavigationUserState);
```
- **Interceptors**: ValidateInput, ValidateMetadata
- **Input/Output**: `{key: string, visited: bool}`

#### GetNavigationUserStateList
```protobuf
rpc GetNavigationUserStateList(EmptyRequest) returns (NavigationUserStateList);
```
- **Interceptors**: ValidateInput, ValidateMetadata
- **Output**: `{navigation_user_state_list: repeated string}`

#### SetNavigationUserState
```protobuf
rpc SetNavigationUserState(NavigationUserState) returns (NavigationUserState);
```
- **Interceptors**: ValidateInput, ValidateMetadata

---

### Home Services & Location

#### GetHomeServices
```protobuf
rpc GetHomeServices(GetHomeServicesRequest) returns (GetHomeServicesResponse);
```
- **Interceptors**: ValidateInput, ValidateMetadata
- **Input**: `{payment_method: string}`
- **Output**:
  ```protobuf
  message GetHomeServicesResponse {
    repeated GetHomeServicesResponse_HomeService data = 1;
  }
  
  message GetHomeServicesResponse_HomeService {
    int32 id = 1;
    GetHomeServicesResponse_HomeService_Name name = 2;  // {en, id}
    string icon = 3;
    bool is_new = 4;
    int32 sort = 5;
    GetHomeServicesResponse_HomeService_Multiplatform multiplatform = 6;
  }
  ```

#### SaveLastLocation
```protobuf
rpc SaveLastLocation(LastLocationRequest) returns (DefaultResponse);
```
- **Interceptors**: ValidateInput, ValidateMetadata
- **Input**: `{latitude, longitude, place_id, title, address}`

#### GetLastLocation
```protobuf
rpc GetLastLocation(EmptyRequest) returns (LastLocationResponse);
```
- **Interceptors**: ValidateMetadata
- **Output**: `{user_id, latitude, longitude, place_id, title, address}`

---

### Referral System

#### GetReferralList
```protobuf
rpc GetReferralList(EmptyRequest) returns (GetReferralListResponse);
```
- **Interceptors**: ValidateMetadata
- **Output**:
  ```protobuf
  message GetReferralListResponse {
    int32 code = 1;
    string status = 2;
    GetReferralListResponseData data = 3;  // {referral_code, referees, total_point}
  }
  ```

#### GetReferralExplanation
```protobuf
rpc GetReferralExplanation(GetReferralExplanationRequest) returns (GetReferralExplanationResponse);
```
- **Interceptors**: ValidateInput, ValidateMetadata
- **Input**: `{usecase: string}`

---

### Additional Info Lists

#### GetListOfHobby
```protobuf
rpc GetListOfHobby(DefaultRequest) returns (GetListOfHobbyResponse);
```
- **Interceptors**: ValidateMetadata
- **Output**: `{data: repeated Hobby}` where Hobby = `{id, hobby_en, hobby_id}`

#### GetListOfDomicile
```protobuf
rpc GetListOfDomicile(DefaultRequest) returns (GetListOfDomicileResponse);
```
- **Interceptors**: ValidateMetadata
- **Output**: `{data: repeated Domicile}` where Domicile = `{id, domicile_en, domicile_id}`

#### GetListOfOccupation
```protobuf
rpc GetListOfOccupation(DefaultRequest) returns (GetListOfOccupationResponse);
```
- **Interceptors**: ValidateMetadata
- **Output**: `{data: repeated Occupation}` where Occupation = `{id, occupation_en, occupation_id}`

---

### Feature Flags & Whitelist

#### WhitelistCheck
```protobuf
rpc WhitelistCheck(WhitelistCheckRequest) returns (WhitelistCheckResponse);
```
- **Input**:
  ```protobuf
  enum FeatureWhitelist {
    FEATURE_UNKNOWN = 0;
    FEATURE_RESCHEDULE = 1;
    FEATURE_BROKER_MIGRATION = 2;
  }
  
  message WhitelistCheckRequest {
    FeatureWhitelist feature = 1;
    string bbid = 2;
    string app_version = 3;
  }
  ```
- **Output**: `{is_whitelisted: bool}`

#### CheckEnabledFeature
```protobuf
rpc CheckEnabledFeature(CheckEnabledFeatureRequest) returns (CheckEnabledFeatureResponse);
```
- **Input**:
  ```protobuf
  enum FeatureKey {
    FEATUREKEY_UNKNOWN = 0;
    FEATUREKEY_RESCHEDULE = 1;
    FEATUREKEY_BROKER_MIGRATION = 2;
    FEATUREKEY_BYPASS_OLDKAFKA = 3;
    FEATUREKEY_REDIRECT_HUAWEI = 4;
  }
  
  message CheckEnabledFeatureRequest {
    FeatureKey key = 1;
    FeatureSubKey sub_key = 2;
    string bbid = 3;
    string app_version = 4;
  }
  ```
- **Output**: `{is_enabled: bool}`

---

### Notification Preferences

#### UpdateLastStatusPromoNotification
```protobuf
rpc UpdateLastStatusPromoNotification(UpdateLastStatusPromoNotificationRequest) returns (UpdateLastStatusPromoNotificationResponse);
```
- **Input**: `{notification_type, identifier, is_active}`

#### GetLastStatusPromoNotification
```protobuf
rpc GetLastStatusPromoNotification(GetLastStatusPromoNotificationRequest) returns (GetLastStatusPromoNotificationResponse);
```
- **Input**: `{bb_id, email, lang}`
- **Output**:
  ```protobuf
  message GetLastStatusPromoNotificationResponse {
    repeated PromoNotificationCategory data = 1;
  }
  
  message PromoNotificationCategory {
    int32 category_id = 1;
    string category_title = 2;
    string category_description = 3;
    repeated PromoNotificationCategory_Notification notification = 4;
  }
  ```

---

### Other Methods

#### GetCountryList
```protobuf
rpc GetCountryList(GetCountryListRequest) returns (GetCountryListResponse);
```
- **Input**: `{phone_number_prefix, region_code}`
- **Output**: `{code, message, data: repeated CountryList}`

#### StreetHailingIsUserEligibleToUseBarcodeFeature
```protobuf
rpc StreetHailingIsUserEligibleToUseBarcodeFeature(EmptyRequest) returns (StreetHailingIsUserEligibleToUseBarcodeFeatureResponse);
```
- **Interceptors**: ValidateMetadata
- **Output**: `{is_eligible: bool}`

#### StoreSchoolBusParentActiveStatus
```protobuf
rpc StoreSchoolBusParentActiveStatus(StoreSchoolBusParentActiveStatusRequest) returns (StoreSchoolBusParentActiveStatusResponse);
```
- **Interceptors**: ValidateInput
- **Input**: `{phone_number, is_active}`
- **Output**: `{success: bool}`

#### GetOutletInfo
```protobuf
rpc GetOutletInfo(GetOutletInfoRequest) returns (GetOutletInfoResponse);
```
- **Input**: `{latitude, longitude, service_type_id, language, source}`
- **Output**: `{url, area_id, area_name, is_booth_widget_enabled, tooltip_text, booth_button_text}`

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
  string firebase_token = 10;
  string subscribe_error = 11;
  repeated string privileges = 12;
  int64 points = 13;
  int64 created_at = 7;
  int64 updated_at = 8;
  bool is_legacy = 6;
  bool is_activated = 9;
  bool is_verified_email = 14;
  bool is_white_list_user = 15;
  string gender = 16;
  string dob = 17;
  Domicile domicile = 18;
  Occupation occupation = 19;
  Hobby hobby = 20;
}
```

### UserResponse
```protobuf
message UserResponse {
  string internal_id = 1;
  string name = 2;
  string email = 3;
  string phone_number = 4;
  string profile_image = 5;
  string referral_code = 6;
  int64 created_at = 7;
  int64 verified_phone_at = 8;
  int64 verified_email_at = 9;
  string user_id = 10;
  string phone = 11;
  double create_at = 12;
  double update_at = 13;
  bool is_activated = 14;
  bool is_legacy = 15;
  double points = 16;
  DefaultPayment personal_default_payment = 17;
  DefaultPayment corporate_default_payment = 18;
  DefaultPayment payment_display = 19;
}
```

### PaymentMethodUserResponse
```protobuf
message PaymentMethodUserResponse {
  int32 code = 1;
  string message = 2;
  repeated PaymentMethodList data = 3;
}

message PaymentMethodList {
  string display = 1;
  string identifier = 2;
  double balance = 3;
  int32 remaining_trip = 4;
  string type = 5;
  bool is_active = 6;
  repeated CarTypeAvailabilityPayment car_type_availability = 7;
  bool is_available_today = 8;
  bool is_available_tomorrow = 9;
  repeated PaymentMethodList group_items = 10;
  DescPayment description = 11;
  string type_cc = 12;
  string category = 13;
  string company_id = 14;
  bool is_unlimited = 15;
  bool is_deletable = 16;
  bool is_expired = 17;
  bool is_outstanding = 18;
  bool is_default = 19;
  bool is_default_corporate = 20;
  bool is_default_personal = 21;
}
```

### FavoriteAddress
```protobuf
message FavoriteAddress {
  int32 id = 1;
  string tag = 2;
  string name = 3;
  int32 position = 4;
  string icon = 5;
  bool bookmark = 6;
  double latitude = 7;
  double longitude = 8;
  string display_address = 9;
  string note_to_driver = 10;
  string color = 11;
  int32 created_at = 12;
  int32 updated_at = 13;
  string place_id = 14;
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

#api #grpc #userservice #mrg #reference

---

*Last Updated*: 2025-01-05
