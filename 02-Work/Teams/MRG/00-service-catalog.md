
```yaml
# ============================================================
# MRG SERVICE CATALOG
# ============================================================
# Purpose   : Single source of truth untuk dependency & domain mapping
# Maintained: Update wajib saat RFC baru dibuat/diapprove
# Owner     : Architecture Engineer (MRG)
# Updated   : 2026-02-27
# ============================================================

metadata:
  team: MRG
  total_services: 22
  update_policy: "Update via RFC. Tambahkan 'Service Catalog Impact' di setiap RFC."

# ============================================================
# DOMAIN MAP - Gunakan ini untuk tentukan "feature ini masuk ke mana"
# ============================================================
domains:
  authentication:
    description: "Login, token, OTP, registrasi, password management"
    primary_service: authservice
    supporting: [fds, sessionmanager, userservice]

  user_management:
    description: "Profil user, alamat favorit, payment method user, preferensi"
    primary_service: userservice
    supporting: [authservice]

  booking_order:
    description: "Cart, create order, order lifecycle (multi-service)"
    primary_service: orderorchestrator
    supporting: [sessionmanager, taxipartnergateway, paymentprocessor, notificationcenter]

  order_query:
    description: "Read/query order, history, detail order"
    primary_service: orderquery
    supporting: []

  payment:
    description: "Payment processing, CC, e-wallet, ECV, promo redeem, refund"
    primary_service: paymentprocessor
    supporting: [mybb-ho-gateway]

  dispatch_driver:
    description: "Dispatch taxi ke driver via BBD"
    primary_service: taxipartnergateway
    supporting: []

  geolocation:
    description: "Reverse geocode, autocomplete, frequent location"
    primary_service: geoservice
    supporting: []

  tracking:
    description: "Real-time car tracking, nearby cars, share trip"
    primary_service: trackerservice
    supporting: []

  notification:
    description: "Email, SMS, Push, WhatsApp notifications"
    primary_service: notificationcenter
    supporting: []

  fraud_detection:
    description: "Soft ban, hard ban, phone fraud detection"
    primary_service: fds
    supporting: []

  session:
    description: "Booking session state (pre-order, rent, cititrans, reschedule)"
    primary_service: sessionmanager
    supporting: []

  head_office_integration:
    description: "Sync transaksi e-wallet ke sistem HO legacy"
    primary_service: mybb-ho-gateway
    supporting: []

  webhook_routing:
    description: "Centralized receiver untuk semua external webhook/callback ke internal broker"
    primary_service: webhookforwarder
    supporting: []

  legacy_gateway:
    description: "Legacy API gateway (monolith, sedang migrasi)"
    primary_service: mybb-api-gateway-go
    supporting: []

  legacy_order_processing:
    description: "Legacy order processing via Kafka (91 topics, sedang migrasi)"
    primary_service: mybb-order-processing-go
    supporting: []

# ============================================================
# SERVICE REGISTRY
# ============================================================
services:

  authservice:
    repo: git.bluebird.id/mybb-ms/authservice
    status: production
    domain: authentication
    ports:
      grpc: 6017
      rest: 8017
    tech: [go, grpc, redis, pubsub]
    exposes:
      - CreateToken / RefreshToken / ValidateToken
      - Login / Logout / RegisterUser
      - SendOtp / ValidateOTP
      - ChangePassword / ForgotPassword / ResetPassword
    depends_on:
      grpc: [userservice, fds]
      pubsub: [notificationcenter]
      infra: [redis]
    consumed_by: [mybb-api-gateway-go, userservice, orderorchestrator]

  userservice:
    repo: git.bluebird.id/mybb-ms/userservice
    status: production
    domain: user_management
    ports:
      grpc: 6015
      rest: 8015
    tech: [go, grpc, postgresql, redis, pubsub, gcs]
    exposes:
      - GetUser / CreateUser / UpdateProfile / DeleteAccount
      - GetFavoriteAddressesV2 / CreateFavoriteAddressV2
      - GetPaymentMethodUser / UpsertPaymentMethodUser
      - GetNavigationUserState / SetNavigationUserState
      - GetHomeServices / SaveLastLocation
    depends_on:
      grpc: [authservice, paymentprocessor, promogateway, orderquery, taxiorderprocessor]
      http: [mapservice]
      infra: [postgresql, redis, gcs, pubsub]
    consumed_by: [authservice, geoservice, orderquery, mybb-api-gateway-go]

  orderorchestrator:
    repo: git.bluebird.id/mybb-ms/orderorchestrator
    status: production
    domain: booking_order
    ports:
      grpc: 6000
      rest: 8000
    tech: [go, grpc, postgresql, redis, rabbitmq, pubsub]
    exposes:
      - CreateCart / AddCartItem / GetCart / DeleteCartItem
      - CreateOrder / CreateOrderWeb
      - GetBookingDetail / GetPayment / GetOrderReceipt
      - GetRideDetail / GetRentDetail / GetDeliveryDetail
      - CallbackPaymentInformation / CallbackOrderItem
      - PromoValidate
    depends_on:
      grpc: [sessionmanager, paymentprocessor, taxipartnergateway, promogateway, notificationcenter, configservice, geoservice]
      infra: [postgresql, redis, rabbitmq, pubsub]
    consumed_by: [mybb-api-gateway-go]

  orderquery:
    repo: git.bluebird.id/mybb-ms/orderquery
    status: production
    domain: order_query
    ports:
      grpc: 6005
      rest: 8005
    tech: [go, grpc, postgresql]
    exposes:
      - GetOrderList / GetOrderDetail / GetOrderSummary
      - GetGroupOrder / GetGroupOrderDetail
    depends_on:
      grpc: [userservice, configservice, sessionmanager, contentprovider, fareservice, promogateway, locationregistry, gborderprocessor]
      http: [autocomplete, copservice]
      infra: [postgresql_main, postgresql_legacy]
    consumed_by: [geoservice, trackerservice, paymentprocessor, userservice]

  paymentprocessor:
    repo: git.bluebird.id/mybb-ms/paymentprocessor
    status: production
    domain: payment
    ports:
      grpc: 6007
      rest: 8007
    tech: [go, grpc, postgresql, redis, pubsub]
    exposes:
      - CompleteTx / CancelTx / ReserveBalance / Refund
      - RegisterCC / RemoveCC / PreauthCCReserve
      - VerifyECV / CompleteTxECVGoldenbird
      - PromoRedeem / PromoReservedAmount
      - GetPaymentMethod / GetPaymentToken
      - TipsDriver / DirectPay / HOTransaction
    depends_on:
      grpc: [upggateway, upgmrggateway, upgpayment, corpportal, promogateway, mybb-ho-gateway, orderquery, taxiorderprocessor, copservice, userservice]
      infra: [postgresql, redis, pubsub]
    consumed_by: [orderorchestrator, userservice, trackerservice, mybb-order-processing-go]

  taxipartnergateway:
    repo: git.bluebird.id/mybb-ms/taxipartnergateway
    status: production
    domain: dispatch_driver
    ports:
      grpc: 6008
      rest: 8008
    tech: [go, grpc, jwt, oauth2]
    exposes:
      - OrderTaxi / CancelOrder / RetryFindingTaxi
      - CreateQuote / GetEstimateFare / EditPartnerFare
      - GetVacantVehicles / GetVehicleLocation / GetEta
      - SetOrderRating / GetOrderRatingReviews
      - GetAreaOperationalAndAirport / NotifyOrderEvent
    depends_on:
      grpc: [bbd_auth, bbd_grpc, bbd_area]
      infra: []
    consumed_by: [orderorchestrator, trackerservice]

  geoservice:
    repo: git.bluebird.id/mybb-ms/geoservice
    status: production
    domain: geolocation
    ports:
      grpc: 6000
      rest: 8000
    tech: [go, grpc, redis, postgresql, broker]
    exposes:
      - ReverseGeocodeV2 / ReverseGeocodeV3
      - AutoComplete / HistoryPlace
    depends_on:
      grpc: [userservice, areamanagement, contentprovider, orderquery]
      http: [mapservice_external]
      infra: [redis, postgresql_legacy, broker]
    consumed_by: [orderorchestrator]

  trackerservice:
    repo: git.bluebird.id/mybb-ms/trackerservice
    status: production
    domain: tracking
    ports:
      grpc: 6013
      rest: 8013
    tech: [go, grpc, redis, firebase, pubsub]
    exposes:
      - CarTracking / GetNearbyCars
      - ShareTrip / OrderChanges
    depends_on:
      grpc: [taxipartnergateway, paymentprocessor, gbgateway, orderquery, routinemanager]
      http: [gorooster]
      infra: [redis, firebase, pubsub]
    consumed_by: []

  notificationcenter:
    repo: git.bluebird.id/mybb-ms/notificationcenter
    status: production
    domain: notification
    ports:
      grpc: 50051
      rest: 8080
    tech: [go, grpc, postgresql, redis, kafka, pubsub, rabbitmq]
    exposes:
      - SendEmail / SendSMS / SendPush / SendWhatsApp
      - SendOTP (multi-channel)
      - BookingEventListener / OrderEventListener
    depends_on:
      infra: [postgresql, redis, kafka, pubsub, rabbitmq]
    consumed_by: [authservice, orderorchestrator, mybb-order-processing-go]

  sessionmanager:
    repo: git.bluebird.id/mybb-ms/sessionmanager
    status: production
    domain: session
    ports:
      grpc: 6027
      rest: 8027
    tech: [go, grpc, redis]
    exposes:
      - SetSession / GetSession / DeleteSession
      - SetSessionEZPay / GetSessionEZPay
      - SetSessionCititrans / GetSessionCititrans
      - SetSessionReschedule / GetSessionReschedule
      - SetSessionRentRevamp / GetSessionRentRevamp / DeleteSessionRentRevamp
    depends_on:
      infra: [redis]
    consumed_by: [orderorchestrator, orderquery, serviceinfo]

  fds:
    repo: git.bluebird.id/mybb-ms/fds
    status: production
    domain: fraud_detection
    ports:
      grpc: 6001
      rest: 8001
    tech: [go, grpc, postgresql, redis]
    exposes:
      - GetSoftBanned / SoftBannedCounter / RevokeSoftBanned
      - SetHardBanned / CheckHardBanned / RevokeHardBanned
      - ScanFraudPhoneNumber / WhitelistPhoneNumbers
    depends_on:
      infra: [postgresql, redis, postgresql_legacy]
    consumed_by: [authservice, mybb-order-processing-go]

  mybb-ho-gateway:
    repo: git.bluebird.id/mybb-ms/mybb-ho-gateway
    status: production
    domain: head_office_integration
    ports:
      grpc: 50052
      rest: 8082
    tech: [go, grpc, postgresql, rabbitmq]
    exposes:
      - HoEWalletTransaction
      - PublishMessage / SendMessage
    depends_on:
      grpc: [configservice]
      http: [ho_legacy_service]
      infra: [postgresql, rabbitmq]
    consumed_by: [paymentprocessor]

  mybb-api-gateway-go:
    repo: git.bluebird.id/mybb-legacy/mybb-app/src/mybb-api-gateway-go
    status: legacy
    domain: legacy_gateway
    ports:
      rest: 3000
      grpc: 3443
    tech: [go, beego, grpc, postgresql, redis, kafka, firebase]
    note: "Monolith legacy. Entry point mobile apps. Migration ongoing."
    migration_to: [authservice, orderorchestrator, userservice, paymentprocessor]

  mybb-order-processing-go:
    repo: git.bluebird.id/mybb-legacy/mybb-app/src/mybb-order-processing-go
    status: legacy
    domain: legacy_order_processing
    ports:
      rest: dynamic
    tech: [go, kafka, postgresql, mongodb, redis]
    note: "91 Kafka topics. Kandidat dekomposisi. Jangan tambah fitur baru di sini."
    migration_to: [orderorchestrator, paymentprocessor, notificationcenter]

  serviceinfo:
    status: production
    note: "Handles fleet list, pricing session. Belum ada README lengkap."
    consumed_by: [sessionmanager]

  top:
    alias: taxiorderprocessor
    status: production
    note: "Taxi Order Processor. Digunakan oleh paymentprocessor & userservice."

undefined

  reservation-service:
    status: unknown
    note: "Belum ada README di Obsidian. Perlu enrichment."

  vehicle-service:
    status: unknown
    note: "Belum ada README di Obsidian. Perlu enrichment."

# ============================================================
# DEPENDENCY MATRIX (quick lookup)
# Format: service -> [services yang dipanggil]
# ============================================================
dependency_matrix:
  authservice:          [userservice, fds, notificationcenter, redis]
  userservice:          [authservice, paymentprocessor, promogateway, orderquery, taxiorderprocessor, mapservice, postgresql, redis, gcs]
  orderorchestrator:    [sessionmanager, paymentprocessor, taxipartnergateway, promogateway, notificationcenter, configservice, geoservice, postgresql, redis]
  orderquery:           [userservice, configservice, sessionmanager, contentprovider, fareservice, promogateway, gborderprocessor, autocomplete, copservice, postgresql]
  paymentprocessor:     [upggateway, corpportal, promogateway, mybb-ho-gateway, orderquery, taxiorderprocessor, copservice, userservice, postgresql, redis]
  taxipartnergateway:   [bbd_auth, bbd_grpc, bbd_area]
  geoservice:           [userservice, areamanagement, contentprovider, orderquery, mapservice_external, redis, postgresql_legacy]
  trackerservice:       [taxipartnergateway, paymentprocessor, gbgateway, orderquery, gorooster, redis, firebase]
  notificationcenter:   [postgresql, redis, kafka, pubsub]
  sessionmanager:       [redis]
  fds:                  [postgresql, redis]
  webhookforwarder:       [oldpaymentproc, cititrans_order_processor, rabbitmq, pubsub]
  mybb-ho-gateway:      [configservice, ho_legacy_service, postgresql, rabbitmq]

# ============================================================
# FEATURE PLACEMENT GUIDE
# Gunakan ini saat ingin menambahkan feature baru
# ============================================================
feature_placement_guide:
  "Login / auth baru":             authservice
  "Data profil / user":            userservice
  "Alamat favorit":                userservice
  "Payment method baru":           paymentprocessor  # + userservice jika ada UI
  "Promo / diskon":                promogateway       # external, atau paymentprocessor jika logic internal
  "Booking / cart":                orderorchestrator
  "Order detail / history":        orderquery
  "Dispatch ke driver":            taxipartnergateway
  "Tracking realtime":             trackerservice
  "Notifikasi (email/sms/push)":   notificationcenter
  "Session booking":               sessionmanager
  "Geocode / autocomplete":        geoservice
  "Fraud prevention / ban":        fds
  "Integrasi HO / akuntansi":      mybb-ho-gateway
  "Feature flag / config":         configservice      # external service
  "Platform fee / handling fee":   orderorchestrator  # atau orderquery jika hanya display
  "Callback dari payment gateway":  webhookforwarder
  "Callback dari BBD order":         webhookforwarder
  "Penambahan tipe kendaraan":     vehicle-service    # perlu enrichment dulu
  "Reservasi web":                 reservation-service
```
