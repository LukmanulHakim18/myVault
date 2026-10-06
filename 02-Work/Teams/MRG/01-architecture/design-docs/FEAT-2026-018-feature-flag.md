# FEAT-2026-018 — Feature Flag via Flipt

> **Status**: Draft  
> **Tanggal**: 2026-06-30  
> **Architect**: Adi Kurniawan  
> **Engineer**: (assignee)  
> **Scope**: Global flags only (per-user targeting → Phase 2)

---

## 1. Overview

Tambah RPC `GetFeatureFlags` ke config-service agar MRG GW bisa serve resolved flag snapshot ke Mobile SDK tanpa menyentuh logic di legacy gateway.

```
Mobile SDK
    │ gRPC
    ▼
MRG GW  ──── gRPC call ────► config-service
                                   ├── Redis HIT → return snapshot
                                   └── Redis MISS → Flipt → cache → return
```

config-service adalah satu-satunya yang tahu tentang Flipt. MRG GW hanya forward.

---

## 2. Proto Contract

Tambahkan ke `contract/config-service.proto`:

```protobuf
// === Feature Flag ===

rpc GetFeatureFlags(GetFeatureFlagsRequest) returns (GetFeatureFlagsResponse);

message GetFeatureFlagsRequest {}

message FeatureFlag {
  string key     = 1;
  bool   enabled = 2;
}

message GetFeatureFlagsResponse {
  repeated FeatureFlag flags                  = 1;
  string               etag                   = 2;
  int32                poll_interval_seconds   = 3;
  int32                stale_threshold_seconds = 4;
  string               evaluated_at            = 5; // RFC3339, Asia/Jakarta
}
```

> **Note**: `variant` dan `payload` tidak dimasukkan dulu — scope global flag saja.  
> Akan ditambah di Phase 2 bersama per-user targeting.

---

## 3. Generate Skeleton dengan Enera

Setelah update `.proto`, jalankan enera untuk generate transport:

```bash
# dari root configservice/
enera generate
# atau jika pakai makefile:
make generate
```

Ini akan menghasilkan:
- `transport/get_feature_flags.go` — auto-generated, jangan edit manual
- Update `transport/base_transport.go` — `UseCaseContract` interface ditambah method `GetFeatureFlags`

Cek hasilnya, pastikan method baru muncul di `UseCaseContract`:

```go
GetFeatureFlags(ctx context.Context, request *contract.GetFeatureFlagsRequest) (response *contract.GetFeatureFlagsResponse, err error)
```

---

## 4. Implementasi Layer by Layer

### 4.1 Repository Interface — Flipt (baru)

Buat file `repository/repoiface/flipt.go`:

```go
package repoiface

import (
    "context"
    "git.bluebird.id/mybb-ms/configservice/contract"
)

type Flipt interface {
    ListFlags(ctx context.Context, namespace string) ([]*contract.FeatureFlag, error)
}
```

### 4.2 Repository Implementor — Flipt (baru)

**Tambah dependency terlebih dahulu:**
```bash
go get go.flipt.io/flipt/rpc/flipt@latest
```

Buat folder `repository/flipt/` dengan file `implementor.go`:

```go
package flipt

import (
    "context"
    "fmt"
    "time"

    "git.bluebird.id/mybb-ms/configservice/contract"
    fliptRPC "go.flipt.io/flipt/rpc/flipt"
    "google.golang.org/grpc"
    "google.golang.org/grpc/credentials/insecure"
    "google.golang.org/grpc/metadata"
)

type Client struct {
    grpc      fliptRPC.FliptClient
    namespace string
    token     string
}

func New(target, namespace, token string, timeout time.Duration) (*Client, error) {
    conn, err := grpc.NewClient(
        target,
        grpc.WithTransportCredentials(insecure.NewCredentials()),
        grpc.WithTimeout(timeout),
    )
    if err != nil {
        return nil, fmt.Errorf("flipt grpc dial: %w", err)
    }
    return &Client{
        grpc:      fliptRPC.NewFliptClient(conn),
        namespace: namespace,
        token:     token,
    }, nil
}

func (c *Client) ListFlags(ctx context.Context, namespace string) ([]*contract.FeatureFlag, error) {
    if c.token != "" {
        ctx = metadata.AppendToOutgoingContext(ctx, "authorization", "Bearer "+c.token)
    }

    resp, err := c.grpc.ListFlags(ctx, &fliptRPC.ListFlagRequest{
        NamespaceKey: namespace,
    })
    if err != nil {
        return nil, fmt.Errorf("flipt ListFlags: %w", err)
    }

    flags := make([]*contract.FeatureFlag, 0, len(resp.Flags))
    for _, f := range resp.Flags {
        flags = append(flags, &contract.FeatureFlag{
            Key:     f.Key,
            Enabled: f.Enabled,
        })
    }
    return flags, nil
}
```

### 4.3 Repository Interface — Redis (tambah method)

Tambah 2 method ke `repository/repoiface/redis.go`:

```go
SetFeatureFlagSnapshot(ctx context.Context, payload *contract.GetFeatureFlagsResponse, ttl time.Duration) error
GetFeatureFlagSnapshot(ctx context.Context) (*contract.GetFeatureFlagsResponse, error)
```

### 4.4 Repository Implementor — Redis (tambah method)

Tambah ke `repository/redis/implementor.go`:

```go
const featureFlagSnapshotKey = "ff:global:snapshot"

func (r *RedisClient) SetFeatureFlagSnapshot(ctx context.Context, payload *contract.GetFeatureFlagsResponse, ttl time.Duration) error {
    b, err := r.marshaler.Marshal(payload)
    if err != nil {
        return err
    }
    return r.conn.Client().Set(ctx, featureFlagSnapshotKey, b, ttl).Err()
}

func (r *RedisClient) GetFeatureFlagSnapshot(ctx context.Context) (*contract.GetFeatureFlagsResponse, error) {
    b, err := r.conn.Client().Get(ctx, featureFlagSnapshotKey).Bytes()
    if err != nil {
        return nil, err
    }
    var response contract.GetFeatureFlagsResponse
    if err := r.marshaler.Unmarshal(b, &response); err != nil {
        return nil, err
    }
    return &response, nil
}
```

> Menggunakan `ProtoMarshaler` yang sudah ada — konsisten dengan pattern HomeService/NavigationState.

### 4.5 Wire Flipt ke Repository

Update `config/repository/repository.go` — tambah Flipt client ke struct `Repository`:

```go
// tambah field
Flipt repoiface.Flipt

// di konstruktor, inisialisasi dari env:
fliptTarget    := config.GetConfig("flipt_target").GetString()   // host:port gRPC
fliptNamespace := config.GetConfig("flipt_namespace").GetString()
fliptToken     := config.GetConfig("flipt_token").GetString()
fliptTimeout   := config.GetConfig("flipt_timeout").GetDuration() // default: 2s

fliptClient, err := flipt.New(fliptTarget, fliptNamespace, fliptToken, fliptTimeout)
if err != nil {
    // fatal — service tidak bisa start tanpa koneksi Flipt
    logger.GetLogger().FatalWithContext(ctx, fmt.Sprintf("failed to init flipt client: %v", err))
}
repo.Flipt = fliptClient
```

### 4.6 Usecase — `GetFeatureFlags`

Buat `usecase/get_feature_flags.go`:

```go
package usecase

import (
    "context"
    "crypto/sha256"
    "fmt"
    "time"

    commonLogger   "git.bluebird.id/mybb-ms/aphrodite/logger"
    commonMetadata "git.bluebird.id/mybb-ms/aphrodite/metadata"
    "git.bluebird.id/mybb-ms/configservice/config"
    "git.bluebird.id/mybb-ms/configservice/config/logger"
    "git.bluebird.id/mybb-ms/configservice/contract"
    "github.com/go-redis/redis/v8"
)

func (u *UseCase) GetFeatureFlags(ctx context.Context, _ *contract.GetFeatureFlagsRequest) (*contract.GetFeatureFlagsResponse, error) {
    md := commonMetadata.GetMetaDataFromContext(ctx)
    logData := map[string]any{"method": "GetFeatureFlags", "metadata": md}

    // [1] Redis cache hit
    cached, err := u.Repo.Redis.GetFeatureFlagSnapshot(ctx)
    if err == nil && cached != nil {
        logger.GetLogger().InfoWithContext(ctx, "feature flags: cache hit", commonLogger.ConvertMapToFieldsWithMetadata(ctx, logData)...)
        return cached, nil
    }
    if err != nil && err != redis.Nil {
        logger.GetLogger().WarnWithContext(ctx, "feature flags: redis error, fallback to flipt", commonLogger.ConvertMapToFieldsWithMetadata(ctx, logData)...)
    }

    // [2] Redis miss → call Flipt
    namespace := config.GetConfig("flipt_namespace").GetString()
    flags, err := u.Repo.Flipt.ListFlags(ctx, namespace)
    if err != nil {
        logger.GetLogger().ErrorWithContext(ctx, "feature flags: flipt error, return empty", commonLogger.ConvertMapToFieldsWithMetadata(ctx, logData)...)
        // safe empty — jangan crash client
        return &contract.GetFeatureFlagsResponse{
            Flags:                []*contract.FeatureFlag{},
            PollIntervalSeconds:  int32(config.GetConfig("flipt_poll_interval_seconds").GetInt()),
            StaleThresholdSeconds: int32(config.GetConfig("flipt_stale_threshold_seconds").GetInt()),
        }, nil
    }

    now := time.Now().In(jakartaLoc())
    etag := buildEtag(flags)
    resp := &contract.GetFeatureFlagsResponse{
        Flags:                 flags,
        Etag:                  etag,
        PollIntervalSeconds:   int32(config.GetConfig("flipt_poll_interval_seconds").GetInt()),
        StaleThresholdSeconds: int32(config.GetConfig("flipt_stale_threshold_seconds").GetInt()),
        EvaluatedAt:           now.Format(time.RFC3339),
    }

    // [3] Set cache
    ttl := config.GetConfig("flipt_cache_ttl").GetDuration()
    if setErr := u.Repo.Redis.SetFeatureFlagSnapshot(ctx, resp, ttl); setErr != nil {
        logger.GetLogger().WarnWithContext(ctx, "feature flags: failed to set cache", commonLogger.ConvertMapToFieldsWithMetadata(ctx, logData)...)
    }

    return resp, nil
}

func buildEtag(flags []*contract.FeatureFlag) string {
    h := sha256.New()
    for _, f := range flags {
        fmt.Fprintf(h, "%s=%v;", f.Key, f.Enabled)
    }
    return fmt.Sprintf("%x", h.Sum(nil))[:16]
}

func jakartaLoc() *time.Location {
    loc, _ := time.LoadLocation("Asia/Jakarta")
    return loc
}
```

---

## 5. Environment Variables (tambah ke `.env.sample`)

```env
# Flipt (gRPC)
FLIPT_TARGET=dev-bbone-grpc-feature-flag-envoy.internal.bluebird.id:80
FLIPT_TOKEN=
FLIPT_NAMESPACE=mybluebird
FLIPT_TIMEOUT=2s
FLIPT_CACHE_TTL=60s
FLIPT_POLL_INTERVAL_SECONDS=300
FLIPT_STALE_THRESHOLD_SECONDS=300
```

> `FLIPT_TARGET` format: `host:port` — tanpa `http://` karena gRPC dial langsung.  
> Port default gRPC Flipt adalah **9000**. Konfirmasi ke DevOps jika port Envoy berbeda.

| Env | Dev | Staging | Prod |
|---|---|---|---|
| `FLIPT_TARGET` | `dev-bbone-grpc-feature-flag-envoy.internal.bluebird.id:80` | TBD | TBD |
| `FLIPT_TOKEN` | (minta ke DevOps/infra) | TBD | TBD |
| `FLIPT_NAMESPACE` | `mybluebird` | `mybluebird` | `mybluebird` |

---

## 6. Cara Setup Flipt

### Dev Environment

Flipt sudah deployed di dev. Akses via:
```
http://dev-bbone-grpc-feature-flag-envoy.internal.bluebird.id
```

> Instance ini di-proxy oleh **Envoy** — pastikan network policy pod config-service bisa reach host ini dari dalam cluster.

**Langkah pertama di dev:**

1. Minta API token ke DevOps/Infra → set ke env `FLIPT_TOKEN`

2. Validasi koneksi gRPC dari lokal (via VPN) pakai `grpcurl`:
```bash
grpcurl \
  -H "authorization: Bearer <token>" \
  -d '{"namespace_key": "mybluebird"}' \
  -plaintext \
  dev-bbone-grpc-feature-flag-envoy.internal.bluebird.id:80 \
  flipt.Flipt/ListFlags
```

Expected response:
```json
{
  "flags": [
    { "key": "new_payment_ui", "enabled": false, "namespaceKey": "mybluebird" }
  ]
}
```

3. Kalau namespace `mybluebird` belum ada, buat via gRPC:
```bash
grpcurl \
  -H "authorization: Bearer <token>" \
  -d '{"key": "mybluebird", "name": "MyBluebird"}' \
  -plaintext \
  dev-bbone-grpc-feature-flag-envoy.internal.bluebird.id:80 \
  flipt.Flipt/CreateNamespace
```

4. Buat flags awal via gRPC:

| Key | Enabled | Keterangan |
|---|---|---|
| `new_payment_ui` | false | Kill-switch payment UI baru |
| `ride_promo_banner` | true | Banner promo beranda |
| `gb_rent_ecv` | false | Feature ECV GB Rent |

```bash
# contoh create flag
grpcurl \
  -H "authorization: Bearer <token>" \
  -d '{"namespace_key": "mybluebird", "key": "new_payment_ui", "name": "New Payment UI", "enabled": false, "type": "BOOLEAN_FLAG_TYPE"}' \
  -plaintext \
  dev-bbone-grpc-feature-flag-envoy.internal.bluebird.id:80 \
  flipt.Flipt/CreateFlag
```

### Staging / Prod

URL TBD — koordinasi dengan Infra. Pattern naming kemungkinan:
- Staging: `stg-bbone-grpc-feature-flag-envoy.internal.bluebird.id`
- Prod: `bbone-grpc-feature-flag-envoy.internal.bluebird.id`

---

## 7. Tambah HTTP Route di config-service.yaml

Route REST di-define di `contract/config-service.yaml`, bukan di proto langsung.  
Tambah 1 rule baru:

```yaml
    - selector: configservice.ConfigurationService.GetFeatureFlags
      get: /feature-flags
```

Setelah enera generate, `contract/config-service.pb.gw.go` akan include route ini otomatis.

**Catatan response format** — config-service menggunakan `CustomMarshaler` yang wrap response:
```json
{
  "data": {
    "flags": [...],
    "etag": "abc123",
    "poll_interval_seconds": 300,
    "stale_threshold_seconds": 300,
    "evaluated_at": "2026-06-30T10:00:00+07:00"
  }
}
```

---

## 8. Integrasi MRG GW (thin proxy via REST)

MRG GW akses config-service via **REST**, bukan gRPC langsung. Config-service expose REST via grpc-gateway di `rest_port`.

MRG GW **tidak implement logic apapun**. Cukup:

1. **Tambah env** di MRG GW:
```env
CONFIG_SERVICE_URL=http://config-service:<rest_port>
```

2. **Controller baru** `controllers/api/v6/flag.go`:

```go
// GET /api/v6/flags/evaluate
func (c *FlagController) Evaluate() {
    configURL := os.Getenv("CONFIG_SERVICE_URL") + "/feature-flags"
    resp, err := http.Get(configURL)
    if err != nil || resp.StatusCode != http.StatusOK {
        // fallback: empty flags, jangan 5xx ke client
        c.Data["json"] = map[string]interface{}{"flags": []interface{}{}}
        c.ServeJSON()
        return
    }
    defer resp.Body.Close()
    // forward body as-is ke client (sudah dalam format {"data": {...}})
    var body map[string]interface{}
    json.NewDecoder(resp.Body).Decode(&body)
    c.Data["json"] = body
    c.ServeJSON()
}
```

---

## 9. Urutan Pengerjaan

```
[ ] 1. go get go.flipt.io/flipt/rpc/flipt@latest
[ ] 2. Update contract/config-service.proto — tambah RPC + messages
[ ] 3. Update contract/config-service.yaml — tambah HTTP rule GET /feature-flags
[ ] 4. Jalankan enera generate → verify:
        - transport/get_feature_flags.go terbentuk
        - contract/config-service.pb.gw.go include route /feature-flags
[ ] 5. Buat repository/repoiface/flipt.go
[ ] 6. Buat repository/flipt/implementor.go (gRPC SDK)
[ ] 7. Tambah method ke repoiface/redis.go + repository/redis/implementor.go
[ ] 8. Wire Flipt di config/repository/repository.go
[ ] 9. Implement usecase/get_feature_flags.go
[ ] 10. Tambah env vars ke .env.sample (FLIPT_TARGET, bukan FLIPT_URL)
[ ] 11. Minta FLIPT_TOKEN ke DevOps, konfirmasi port gRPC Envoy
[ ] 12. Validasi grpcurl ke dev Flipt + buat namespace mybluebird + flags awal
[ ] 13. Integration test: REST GET /feature-flags → verify Redis cache + Flipt gRPC response
[ ] 14. MRG GW: tambah env CONFIG_SERVICE_URL + thin proxy controller
```

---

## 10. Open Items

| ID | Item | Owner |
|---|---|---|
| OI-1 | Flipt DB: SQLite OK untuk start, migrasi ke PostgreSQL di Phase 2? | Infra |
| OI-2 | Auth token Flipt: simpan di Vault/K8s Secret | DevOps |
| OI-3 | Siapa yang bisa toggle flags di Flipt UI? Governance | PM/Ops |
| OI-4 | Phase 2: per-user targeting → tambah `user_id`/`device_id` ke request | Architect |

---

*Dokumen ini adalah implementation brief untuk engineer. Untuk context integrasi client-side, lihat:*  
*`features/FEAT-2026-018-mybb-feature-flag-client/integration-doc.md` (Bluelink)*
