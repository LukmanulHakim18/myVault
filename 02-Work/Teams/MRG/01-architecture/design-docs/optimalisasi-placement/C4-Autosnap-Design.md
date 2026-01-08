---
title: C4 AutoSnap Design
parent: Optimalisasi Placement MRG
type: architecture
tags:
  - architecture
  - c4-model
  - autosnap
  - postgis
---
# C4 Model: Optimalisasi Placement - AutoSnap Design

**Parent**: [[README|Optimalisasi Placement untuk Tim MRG]]

---

## 1. System Context (Level 1)

**Goal**: Improve the accuracy of pickup and drop-off points based on user behavior and landmark clustering.

### Actors:

- **Mobile User**: Requests autosnap recommendations (`lat`, `lng`, `user_id`, `type`).
- **AutoSnap System**: Processes request and returns recommended points.

---

## 2. Container Diagram (Level 2)

### Components:

- **Mobile App**: Sends autosnap request.
- **API Gateway**: Validates and forwards the request to usecase layer.
- **AutoSnap Usecase**: Core business logic for generating location recommendations.
- **Repositories**:
  - `FavoriteRepo`: Fetch favorite user locations.
  - `FrequencyRepo`: Fetch frequent locations from `user_location_frequency`.
- **MapService**: Handles geo spatial queries like radius and landmark detection.
- **PostGISDB**: Stores all spatial data and supports PostGIS queries.
- **Kafka Consumer (Setter Flow)**: Listens to `endtrip` events and processes frequency updates.
- **Redis**: Temporary counter store for location usage.

---

## 3. Component Detail (Level 3)

### Get Flow (Getter)

Lihat diagram: [[Diagrams/get-auto-snap]]

### Set Flow (Setter)

- [[Diagrams/setter-frequency]] - Set frequency visited point
- [[Diagrams/setter-popular]] - Set popular point from landmark

---

## 4. Prioritization Logic

- **Favorite Place**: Private location manually selected by the user.
- **Frequent Place**: Calculated from usage (if visited >= 3 times).
- **Order of Preference**:
  1. Favorite place
  2. Frequent place

---

## 5. Concerns & Solutions

| Concern                      | Solution                                      |
| ---------------------------- | --------------------------------------------- |
| Heavy read load on PostGISDB | Use read-replica DB to distribute SELECT load |
| Expensive ST_DWithin queries | Ensure spatial index on geometry fields       |
| Landmark validation overhead | Cache landmark IDs in memory layer            |
| Redis memory growth          | Use TTL or capped counters                    |

---

## 6. Summary

This architecture supports accurate pickup and drop-off recommendations through:

- Realtime event-driven updates via Kafka.
- Radius-based matching.
- User-specific and global landmark awareness.
- Scalable components (Redis + Read Replica) for efficiency.

Ready to be plugged into production-grade mobility systems.

---

## 7. Database

### user_location_frequency

#### DDL

| Column      | Type                   | Description                                                  |
| ----------- | ---------------------- | ------------------------------------------------------------ |
| user_id     | varchar                | ID pengguna                                                  |
| type        | TEXT                   | Jenis lokasi: `pickup` atau `dropoff`                        |
| geom        | GEOGRAPHY(Point, 4326) | Titik koordinat lokasi (lat/lng)                             |
| address     | TEXT                   | Alamat hasil reverse geocoding                               |
| time_of_day | TEXT                   | Waktu penggunaan: `morning`, `afternoon`, `evening`, `night` |
| count       | INTEGER                | Jumlah frekuensi penggunaan lokasi tersebut                  |
| valid_until | TIMESTAMPTZ            | Data ini akan valid hingga.                                  |
| updated_at  | TIMESTAMPTZ            | Waktu terakhir data diperbarui                               |

**Indexes**:

- `GIST(geom)` → untuk query radius (ST_DWithin)
- `btree(user_id)`
- `btree(type)`
- `btree(time_of_day)`

```sql
CREATE TABLE user_location_frequency (
  user_id     varchar NOT NULL,
  type        TEXT NOT NULL CHECK (type IN ('pickup', 'dropoff')),
  geom        GEOGRAPHY(Point, 4326) NOT NULL,
  address     TEXT,
  time_of_day TEXT CHECK (time_of_day IN ('morning', 'afternoon', 'evening', 'night')),
  count       INTEGER NOT NULL DEFAULT 1,
  updated_at  TIMESTAMPTZ DEFAULT now()
);

CREATE INDEX idx_user_freq_geom ON user_location_frequency USING GIST (geom);
CREATE INDEX idx_user_freq_user_id ON user_location_frequency (user_id);
CREATE INDEX idx_user_freq_type ON user_location_frequency (type);
CREATE INDEX idx_user_freq_timeofday ON user_location_frequency (time_of_day);
```

#### Contoh Query ST_DWithin

```sql
SELECT address, count
FROM user_location_frequency
WHERE user_id = 'abc123'
  AND type = 'pickup'
  AND ST_DWithin(geom, ST_MakePoint(106.816, -6.2)::geography, 10)
ORDER BY count DESC;
```

---

### Popular Place (subplace) From Landmark

#### DDL

| Column      | Type                   | Description                                                  |
| ----------- | ---------------------- | ------------------------------------------------------------ |
| type        | TEXT                   | Jenis lokasi: `pickup` atau `dropoff`                        |
| geom        | GEOGRAPHY(Point, 4326) | Titik koordinat lokasi (lat/lng)                             |
| address     | TEXT                   | Alamat hasil reverse geocoding                               |
| time_of_day | TEXT                   | Waktu penggunaan: `morning`, `afternoon`, `evening`, `night` |
| count       | INTEGER                | Jumlah global frekuensi lokasi                               |
| landmark_id | varchar                | ID landmark (opsional, seperti mall)                         |
| subplace_id | varchar                | ID subplace (opsional, seperti lobby mall)                   |
| updated_at  | TIMESTAMPTZ            | Waktu terakhir data diperbarui                               |

**Indexes**:

- `GIST(geom)` → untuk spatial query
- `btree(type)`
- `btree(time_of_day)`

```sql
CREATE TABLE global_location_frequency (
  type         TEXT NOT NULL CHECK (type IN ('pickup', 'dropoff')),
  geom         GEOGRAPHY(Point, 4326) NOT NULL,
  time_of_day  TEXT CHECK (time_of_day IN ('morning', 'afternoon', 'evening', 'night')),
  count        INTEGER NOT NULL DEFAULT 1,
  updated_at   TIMESTAMPTZ DEFAULT now(),
  landmark_id  varchar,
  subplace_id  varchar
);

CREATE INDEX idx_global_freq_geom ON global_location_frequency USING GIST (geom);
CREATE INDEX idx_global_freq_type ON global_location_frequency (type);
CREATE INDEX idx_global_freq_timeofday ON global_location_frequency (time_of_day);
```

#### Contoh Query ST_DWithin + Time of Day

```sql
SELECT
  address,
  count,
  time_of_day,
  ST_AsText(geom) AS location
FROM global_location_frequency
WHERE type = 'pickup'
  AND ST_DWithin(
        geom,
        ST_MakePoint(106.816, -6.2)::geography,
        30
      )
ORDER BY count DESC
LIMIT 5;
```

---

## 8. Redis Strategy

| Tujuan                       | Redis Feature    | Command     |
| ---------------------------- | ---------------- | ----------- |
| Menyimpan lokasi pickup user | Geospatial Index | `GEOADD`    |
| Mengecek lokasi dekat        | Geospatial Query | `GEORADIUS` |
| Menghitung jumlah kunjungan  | Atomic Counter   | `INCR`      |
| Menyimpan hitungan awal      | Key-Value Store  | `SET`       |
| Menghapus data lama          | Key Deletion     | `DEL`       |

---

## 🏷️ Tags

#architecture #c4-model #autosnap #postgis #redis #database

---

*Last Updated*: 2025-01-05
