## 1. Latar Belakang

Sistem harga logistik dimodelkan sebagai **graph**, di mana:

- **Kota** direpresentasikan sebagai node
- **Rute** direpresentasikan sebagai edge berarah
- **Harga** merupakan bobot (weight) dari setiap rute

Pendekatan ini memungkinkan perhitungan harga secara otomatis melalui beberapa rute (multi-hop / chaining).

---

## 2. Struktur Database

### 2.1 Tabel `kota`

```sql
kota (
  id INT PRIMARY KEY,
  detail TEXT
)
```

### 2.2 Tabel `rute`

```sql
rute (
  id INT PRIMARY KEY,
  origin INT REFERENCES kota(id),
  dest INT REFERENCES kota(id),
  price DECIMAL
)
```

---

## 3. Konsep Graph

Setiap record pada tabel `rute` membentuk relasi berarah:

```
origin ──(price)──> dest
```

Contoh relasi:

```
A → B (10)
B → C (15)
C → D (20)
A → C (30)
```

Sistem mendukung:

- Rute langsung
- Rute berantai
- Perhitungan total harga otomatis

---

## 4. Query Chaining Rute Menggunakan Recursive CTE

### 4.1 Tujuan

- Menemukan seluruh kemungkinan rute dari kota asal ke kota tujuan
- Menghitung total harga setiap rute
- Mencegah infinite loop

### 4.2 Query Recursive

```sql
WITH RECURSIVE route_chain AS (
    SELECT
        r.origin,
        r.dest,
        r.price AS total_price,
        ARRAY[r.origin, r.dest] AS path,
        1 AS depth
    FROM rute r
    WHERE r.origin = :origin_id

    UNION ALL

    SELECT
        rc.origin,
        r.dest,
        rc.total_price + r.price AS total_price,
        rc.path || r.dest,
        rc.depth + 1
    FROM route_chain rc
    JOIN rute r ON r.origin = rc.dest
    WHERE rc.depth < 5
      AND NOT r.dest = ANY(rc.path)
)
SELECT *
FROM route_chain
WHERE dest = :dest_id;
```

---

## 5. Penjelasan Output

| Field       | Deskripsi                 |
| ----------- | ------------------------- |
| origin      | Kota asal                 |
| dest        | Kota tujuan akhir         |
| total_price | Total harga seluruh rute  |
| path        | Urutan kota yang dilewati |
| depth       | Jumlah hop                |

---

## 6. Mendapatkan Rute Termurah

```sql
SELECT *
FROM (
    -- query recursive sebelumnya
) t
ORDER BY total_price
LIMIT 1;
```

---

## 7. Optimasi Performa

```sql
CREATE INDEX idx_rute_origin ON rute(origin);
CREATE INDEX idx_rute_dest ON rute(dest);
```

---

## 8. Kesimpulan

- Harga logistik dapat dimodelkan sebagai graph
- Recursive CTE efektif untuk chaining rute
- Untuk sistem skala besar, kalkulasi sebaiknya dilakukan di service layer
