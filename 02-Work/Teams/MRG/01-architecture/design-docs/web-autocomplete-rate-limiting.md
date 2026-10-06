---
title: "Web AutoComplete Rate Limiting — Google ke OSNames"
category: performance
date: '2026-04-13'
source: brainstorm
status: published
deployment_status: development
card: null
summary: >
  Rate limiting per device_id untuk web client (channel WebReservation) pada
  usecase AutoComplete — switch provider dari Google (berbayar) ke OSNames
  (open source, gratis) secara otomatis saat kuota per device habis dalam
  satu time window. Counter disimpan di Redis dengan fixed window TTL.
tags:
  - performance
  - cache
  - architecture
  - api
team: MRG
service: geoservice
---

## Konteks

Geoservice menyediakan endpoint `AutoComplete` yang secara internal memanggil
**Area Management Service** dengan provider `google`. Google Maps Platform API
bersifat berbayar dan memiliki kuota penggunaan.

**Masalah:**
Web client (channel `WebReservation`) mengkonsumsi Google API quota melalui
AutoComplete — berpotensi menyebabkan over-quota yang berdampak biaya atau
downtime layanan pencarian lokasi untuk semua client.

**Kondisi saat ini:**
- Channel `WebReservation` → provider selalu `google`
- Channel `MyBB` (app) → provider selalu `google`
- `ReverseGeocodeV3` untuk WebReservation sudah menggunakan `nominatim` (gratis) — **tidak perlu diubah**
- Area Management sudah mendukung provider alternatif: `osnames` (autocomplete) dan `nominatim` (geocode)

**Mengapa perlu rate limiting per device?**
Pembatasan di level device (via `md.DeviceId` dari metadata Aphrodite)
memungkinkan kontrol granular — user yang berlebihan tidak mempengaruhi
device/session lain. Threshold ditentukan oleh STO.

---

## Isi Utama

### Pendekatan: Fixed Window Counter di Redis

```
AutoComplete request (WebReservation)
         │
         ├─ md.DeviceId kosong? ──→ fail open (Google)
         │
         └─ Redis INCR "geoservice:rate_limit:autocomplete:{device_id}"
                  │
                  ├─ Redis error? ──→ fail open (Google)
                  │
                  ├─ count ≤ threshold ──→ provider = "google"
                  │
                  └─ count > threshold ──→ provider = "osnames"
                                │
                     AreaManagement.AutoComplete(provider="osnames")
```

**Fixed Window — cara kerja:**
- Key di Redis dibuat saat pertama kali request dalam window
- TTL di-set hanya sekali (saat `count == 1`) → window tidak geser
- Saat TTL habis, key otomatis terhapus → counter reset untuk window berikutnya

### Perbandingan Opsi Counter

| | Fixed Window | Sliding Window |
|---|---|---|
| Implementasi | `INCR` + `EXPIRE` once | Sorted Set + `ZRANGEBYSCORE` |
| Akurasi | ±1 window di edge case | Tepat per detik |
| Kompleksitas | Rendah | Tinggi |
| Memory Redis | 1 key per device | N entries per device |
| Cocok untuk use case ini | ✅ | Overkill |

Fixed window dipilih karena akurasi edge-case tidak kritis untuk rate limiting biaya API.

### Komponen yang Terlibat

| Komponen | Peran | Perubahan |
|---|---|---|
| `usecase/auto_complete.go` | Logic provider selection | Tambah `resolveWebProvider()` |
| `repository/repoiface/redis.go` | Interface IRedis | +1 method `IncrWebAutoCompleteCounter` |
| `repository/redis/implementor.go` | Redis implementation | +1 method implementasi |
| `repository/repoiface/mocks.go` | Mock untuk test | +1 mock method |
| `config/default.go` | Konfigurasi | +2 config key |
| Area Management Service | Provider routing | Tidak berubah (sudah support `osnames`) |

### Config Baru

| Key | Default | Deskripsi |
|---|---|---|
| `web_autocomplete_rate_limit_threshold` | `100` | Maks request Google per window per device |
| `web_autocomplete_rate_limit_window` | `1h` | Durasi window (reset otomatis) |

> Nilai aktual ditentukan STO. Default di atas hanya placeholder.

### Redis Key Design

```
Key    : geoservice:rate_limit:autocomplete:{device_id}
Type   : STRING (integer counter)
TTL    : web_autocomplete_rate_limit_window (e.g. 1h)
Reset  : Otomatis saat TTL habis (fixed window)
```

### Behavior Tabel Lengkap

| Kondisi | Provider |
|---|---|
| Channel = MyBB (app) | `google` — tidak kena rate limit |
| Channel = WebReservation, DeviceId kosong | `google` (fail open) |
| Channel = WebReservation, Redis error | `google` (fail open) |
| Channel = WebReservation, count ≤ threshold | `google` |
| Channel = WebReservation, count > threshold | `osnames` |
| area_only = true + count ≤ threshold | `area,google` |
| area_only = true + count > threshold | `area,osnames` |

### Implementasi Inti

**`repository/repoiface/redis.go` — tambah method:**
```go
IncrWebAutoCompleteCounter(ctx context.Context, deviceID string, window time.Duration) (int64, error)
```

**`repository/redis/implementor.go` — implementasi:**
```go
func (r *RedisClient) IncrWebAutoCompleteCounter(
    ctx context.Context, deviceID string, window time.Duration,
) (int64, error) {
    key := fmt.Sprintf("geoservice:rate_limit:autocomplete:%s", deviceID)
    count, err := r.cli.Incr(ctx, key).Result()
    if err != nil {
        return 0, err
    }
    if count == 1 {
        r.cli.Expire(ctx, key, window) // TTL hanya di-set sekali
    }
    return count, nil
}
```

**`usecase/auto_complete.go` — helper method:**
```go
func (u *UseCase) resolveWebProvider(ctx context.Context, deviceID string) constants.AreaManagementProvider {
    if deviceID == "" {
        return constants.ProviderGoogle
    }
    threshold := config.GetConfig("web_autocomplete_rate_limit_threshold").GetInt()
    window    := config.GetConfig("web_autocomplete_rate_limit_window").GetDuration()

    count, err := u.Repo.Redis.IncrWebAutoCompleteCounter(ctx, deviceID, window)
    if err != nil {
        return constants.ProviderGoogle // fail open
    }
    if count > int64(threshold) {
        return constants.ProviderOsnames
    }
    return constants.ProviderGoogle
}
```

---

## Keputusan / Hasil

**Keputusan:** Implementasi fixed window rate limiting per `device_id` untuk
channel `WebReservation` pada usecase `AutoComplete`, dengan fallback otomatis
ke provider `osnames` (gratis) saat kuota terlampaui.

**Alasan:**
- Area Management sudah support `osnames` — tidak perlu integrasi baru
- `md.DeviceId` sudah tersedia dari metadata Aphrodite — tidak perlu header tambahan
- Scope perubahan minimal: 3 file dimodifikasi, 0 file baru
- App client (MyBB) tidak terdampak — logic hanya aktif untuk WebReservation
- Fail open design menjaga ketersediaan layanan meski Redis bermasalah

**Trade-off yang diterima:**
- **Fixed window edge case**: Request di akhir window + awal window berikutnya bisa melebihi 2× threshold dalam ~1 detik. Diterima karena akurasi per-detik tidak kritis untuk kontrol biaya API.
- **DeviceId kosong = no tracking**: Web session tanpa device_id tidak ditracking. Diterima — kasus ini minor dan fail open ke Google.
- **`osnames` quality**: Kualitas hasil `osnames` mungkin berbeda dari Google. Trade-off yang disadari antara biaya dan kualitas saat quota habis.

---

## Referensi

- `usecase/auto_complete.go` — provider selection logic (baris 82–104)
- `repository/repoiface/redis.go` — interface IRedis
- `repository/redis/implementor.go` — Redis client implementation
- `repository/repoiface/mocks.go` — generated mocks untuk test
- `util/constants/area_management.go` — definisi ProviderGoogle, ProviderOsnames
- `config/default.go` — konfigurasi service
- `.docs/meta-spec/flows.md` — Auto Complete Flow (sequence diagram)
- `.docs/meta-spec/dependencies.md` — interface IRedis dan AreaManagement

---
*Generated by `/arc:feature` — 2026-04-13*
*Session: brainstorm → design → feature doc*
