# 📍 Maps Related Issues & Solutions

Berikut analisis isu Maps yang ditemukan beserta penjelasan akar masalah dan solusi implementasinya.

---

## 1. Lokasi POI tidak benar

- **Contoh:** "Gambir Train Station" muncul di Manggarai.
- **Analisa / Root Cause:** Data POI Google Maps tidak akurat sehingga koordinat pickup salah.
- **Impact:** Driver salah jemput, fixed price tidak sesuai.
- **Solusi:**
  - Perbaikan via GMCP (DONE).
  - Tambahkan **internal POI correction layer** (override DB internal) agar lebih cepat tanpa menunggu update Google.

---

## 2. Alamat POI tidak benar

- **Contoh:** Alamat hasil reverse geocoding: "Jalan Tanpa Nama" atau "Sumatera Selatan" padahal lokasi di Jakarta.
- **Analisa / Root Cause:** Google Maps memberikan alamat reverse geocode yang tidak sesuai.
- **Impact:** Customer menolak order karena alamat tidak sesuai.
- **Solusi:**
  - Cross-check dengan data **OpenStreetMap / HERE API**.
  - Simpan alamat hasil validasi manual (mapping `place_id` → alamat benar) untuk area yang bermasalah.

---

## 3. Saat macet parah, ATA >> ETA

- **Contoh:** ETA 1 jam, real perjalanan 1,5 jam.
- **Analisa / Root Cause:** ETA tidak mempertimbangkan traffic ekstrem (jalan stuck total).
- **Impact:** Real argo jauh > Fixed price.
- **Solusi:**
  - Tambah **waiting fee** jika kecepatan rata-rata <15 km/jam (INPROGRESS).
  - Integrasi data traffic dari **HERE / TomTom** untuk perhitungan lebih realistis.
  - Terapkan **dynamic pricing adjustment** bila gap ETA vs ATA terlalu besar.

---

## 4. Beda rute saat hitung harga vs navigasi

- **Contoh:** Perhitungan harga via Kuningan, navigasi lewat Sudirman.
- **Analisa / Root Cause:** Engine pricing dan routing menggunakan preferensi berbeda.
- **Impact:** Real argo ≠ Fixed price.
- **Solusi:**
  - Gunakan overlay routing yang sama di pricing & navigasi (DONE).
  - Standardisasi ke satu **routing preference (fastest + traffic-aware)**.

---

## 5. Rute tidak mengenali private road

- **Contoh:** Navigasi taxi diarahkan memutar karena private road tidak dikenali.
- **Analisa / Root Cause:** Google Maps tidak mengenali private access road.
- **Impact:** Rute memutar, argo tidak sesuai.
- **Solusi:**
  - Submit private road ke GMCP (TODO).
  - Tambahkan **custom private road layer internal** dengan data OSM/GraphHopper.
  - Override hasil Google jika private road terdeteksi di DB internal.

---

## 6. Taksi bergerak tapi tracking tidak

- **Contoh:** Taxi bergerak, tapi tracking di app diam.
- **Analisa / Root Cause:** Bug pada SDK ODRD v5.
- **Impact:** ETA pickup lama, order bisa batal.
- **Solusi:**
  - Upgrade SDK ke v6 (DONE).
  - Tambahkan **heartbeat GPS driver app** → trigger re-sync jika lokasi tidak update.

---

## 7. Snap to Road salah → ETA lama

- **Contoh:** Taxi di jalan biasa tapi di-snap ke jalan layang.
- **Analisa / Root Cause:** Algoritma Snap-to-Road salah memilih jalur.
- **Impact:** ETA pickup lama → order berpotensi batal.
- **Solusi:**
  - Nonaktifkan Snap-to-Road jika deviasi lokasi > 15m (TBD).
  - Validasi dengan multi-source positioning (GPS + sensor device).
  - Prioritaskan jalan setingkat, bukan elevated road.

---

## 8. Navigasi Taksi berputar (u-turn logic error)

- **Contoh:** Taxi diarahkan u-turn resmi padahal jalan 2 arah bisa langsung balik.
- **Analisa / Root Cause:** Routing engine ODRD hanya pakai u-turn resmi.
- **Impact:** ETA pickup lama → order batal.
- **Solusi:**
  - Ajukan u-turn ke GMCP (TODO).
  - Override rule: jika jalan 2 arah tanpa separator → izinkan u-turn manual.

---

## 9. Loading Tracking Navigasi lambat

- **Contoh:** Route muncul garis abu-abu statis, bukan indikator traffic.
- **Analisa / Root Cause:** Route ditampilkan statis, tidak real-time.
- **Impact:** User tidak lihat posisi taxi terbaru.
- **Solusi:**
  - Tambah event analytics tracking (INPROGRESS).
  - Gunakan **real-time streaming** (WebSocket/GRPC).
  - Render polyline dengan traffic indicator.

---

## 10. Inkonsistensi ETA

- **Contoh:** ETA berubah drastis dalam 2 menit.
- **Analisa / Root Cause:** Fake GPS atau stop tracking driver.
- **Impact:** User bingung, trust turun.
- **Solusi:**
  - Fake GPS detection (DONE).
  - Stop tracking otomatis saat end shift (DONE).
  - Tambahkan **anomaly detection** bila ETA berubah ekstrem.

---

## 11. Issue routing options

- **Contoh:** Rute MyBB beda dengan Google Maps (hindari jalan HR Rasuna Said).
- **Analisa / Root Cause:** Routing engine tidak traffic-aware.
- **Impact:** User tidak diarahkan ke rute tercepat.
- **Solusi:**
  - Ganti routing option ke **TRAFFIC_AWARE_OPTIMAL** (DONE).
  - Uji coba A/B untuk validasi ETA vs perjalanan real.

---

## 12. Halaman map blank

- **Contoh:** Area navigasi kosong meski destinasi & driver info muncul.
- **Analisa / Root Cause:** Bug rendering layer SDK Google Maps.
- **Impact:** User tidak bisa melihat tracking.
- **Solusi:**
  - Lapor ke Google (DONE).
  - Tambahkan **fallback renderer** (Mapbox/OSM).
  - Implementasi retry + error logging.

---

# 📌 Ringkasan Solusi

- **Isu Data POI & Jalan (1,2,5,8):** GMCP + internal override layer.
- **Isu ETA & Routing (3,4,7,10,11):** Routing engine konsisten & traffic-aware + anomaly detection.
- **Isu Tracking & Map Rendering (6,9,12):** SDK upgrade, fallback renderer, real-time streaming.
 