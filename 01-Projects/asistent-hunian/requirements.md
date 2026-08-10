# Requirements Specification — Asisten Hunian

> **Status**: Requirements Discovery (hasil `/sa:brainstorm`)
> **Tanggal**: 2026-08-10
> **Scope MVP**: 1 lokasi — Rusun ASN PUPR Pasar Jumat
> **Next step**: `/sa:design` untuk arsitektur, atau `/sa:workflow` untuk implementation planning

---

## 1. Ringkasan & Tujuan

Platform on-demand home services (awalnya cleaning, diperluas ke jasa rumah tangga lain seperti instalasi heater, perbaikan, dll) yang mempertemukan **User** (penghuni) dengan **Mitra** (penyedia jasa), dikelola oleh **Admin** (pemilik platform, single-role untuk MVP).

MVP diluncurkan **hyper-local**: hanya untuk satu kompleks hunian (Rusun ASN PUPR Pasar Jumat), bukan per kota. Tidak ada kebutuhan geo-matching/multi-area untuk versi awal — semua User dan Mitra berada di lokasi yang sama.

**Model bisnis inti**: User bayar di muka (QRIS dinamis via Midtrans) → dana ditahan platform (escrow) → setelah order selesai dan lolos window komplain/rating → dana payout Mitra di-unlock untuk withdrawal. Selisih antara harga yang dibayar User dan payout Mitra (dikurangi platform fee flat) menjadi margin Owner (pemilik platform), yang antara lain dipakai untuk operasional termasuk gaji Admin di masa depan.

---

## 2. Actors & Roles

| Role | Deskripsi |
|---|---|
| **User** | Penghuni yang order jasa. Registrasi via email, verifikasi email wajib sebelum order. |
| **Mitra** | Penyedia jasa (bukan hanya cleaning — juga install heater, repair, dll). Didaftarkan dengan kategori/skill tertentu. Menerima konfirmasi order via WhatsApp. |
| **Admin** | Single role untuk MVP (dipegang Owner sendiri). Assign Mitra ke order, review komplain, kelola service catalog & harga. Tidak ada role Owner/Super Admin terpisah di sistem untuk versi ini — akan dipertimbangkan lagi kalau sudah rekrut Admin operasional. |

---

## 3. Functional Requirements

### 3.1 User

- **FR-U01 Register**: User daftar dengan email + password (atau data minimal lain). Field nomor HP disediakan dengan **validasi format saja** (regex pola nomor Indonesia), **tanpa OTP** — tidak wajib aktif/terverifikasi.
- **FR-U02 Verifikasi Email**: User wajib klik link/kode verifikasi email sebelum bisa login dan order. Ini satu-satunya mekanisme verifikasi untuk MVP (tidak ada verifikasi KTP/identitas).
- **FR-U03 Login**.
- **FR-U04 Browsing service catalog**: List service yang tersedia, dikelompokkan per kategori (Cleaning, Install Heater, dll), dengan harga tetap per service (diinput manual oleh Admin).
- **FR-U05 Order**: User pilih service, **pilih tanggal & jam** (scheduling terjadi di titik order, sebelum bayar), lalu bayar via QRIS dinamis (Midtrans).
- **FR-U06 List order aktif**: Menampilkan order dengan status: `waiting_for_payment`, `scheduled`, `in_progress`.
- **FR-U07 List order selesai**: Order dengan status `complete` (termasuk yang sudah/belum direview/komplain).
- **FR-U08 Order detail**: Detail satu order — service, harga, jadwal, Mitra assigned, status, foto bukti before/after (kalau sudah ada).
- **FR-U09 Rating**: Setelah order complete, User bisa kasih rating 1–5 bintang (opsional). **Kalau rating ≤ 3, komentar wajib diisi.** Rating bukan syarat wajib untuk order dianggap selesai di sisi User.
- **FR-U10 Complain**: User bisa ajukan komplain dalam window 3 hari setelah order complete. Complain masuk status pending untuk direview manual oleh Admin.

### 3.2 Admin

- **FR-A01 Login**.
- **FR-A02 List Mitra**: Melihat daftar Mitra terdaftar beserta kategori/skill masing-masing.
- **FR-A03 List order aktif**: Melihat semua order yang perlu ditindaklanjuti (termasuk yang sudah dibayar dan butuh pairing).
- **FR-A04 Pairing order ke Mitra**: Setelah order berstatus `PAID`, Admin pilih Mitra yang sesuai (difilter berdasarkan kategori/skill yang match dengan service order) untuk di-assign.
- **FR-A05 Kelola service catalog**: Admin bisa tambah/edit service — set harga yang ditampilkan ke User, set payout untuk Mitra (**diinput manual**, tidak pakai rumus persentase otomatis), dan set kategori/skill service tsb.
- **FR-A06 Review Complain**: Admin melihat komplain masuk, investigasi manual, putuskan **refund penuh/sebagian atau tolak**. Selama status pending, dana payout Mitra terkait tetap ditahan (block withdrawal).

### 3.3 Mitra

- **FR-M01 Registrasi Mitra** *(implisit — perlu didefinisikan lebih lanjut, lihat Open Questions)*: Mitra didaftarkan (oleh Admin atau self-register lalu diverifikasi Admin) dengan kategori/skill yang dikuasai.
- **FR-M02 Start order**: Mitra mulai kerja setelah **upload foto bukti kondisi "before"** — wajib, jadi syarat status order berubah ke `in_progress`.
- **FR-M03 Complete order**: Mitra selesaikan order setelah **upload foto bukti hasil "after"** — wajib, jadi syarat status order berubah ke `complete`.
- **FR-M04 Withdrawal**: Mitra bisa withdraw payout setelah **salah satu terpenuhi lebih dulu**: (a) User memberi rating, atau (b) window komplain 3 hari lewat tanpa komplain diajukan. Kalau ada komplain pending, withdrawal untuk order tsb diblokir sampai Admin memutuskan.

### 3.4 Konfigurasi (Config)

- **FR-C01 Platform fee**: Flat **Rp4.000** per transaksi, dihitung otomatis oleh sistem (tidak diinput manual per service).
- **FR-C02 Window komplain & rating**: 3 hari sejak order `complete`, dapat dikonfigurasi.
- **FR-C03 Payment gateway**: QRIS dinamis via **Midtrans**.
- **FR-C04 Notifikasi**: Email ke User (registrasi, verifikasi, status order, dll). WhatsApp ke Mitra (konfirmasi order start & complete).
- **FR-C05 Service & skill category**: Master data kategori service (Cleaning, Install Heater, dll) yang juga dipakai sebagai skill tag Mitra untuk keperluan pairing (FR-A04).

---

## 4. Non-Functional Requirements

- **NFR-01 Tech stack** *(constraint dari user, bukan hasil eksplorasi)*: Go (backend), MySQL (primary datastore), Redis (caching/session/queue).
- **NFR-02 Scale**: MVP untuk 1 lokasi (1 kompleks hunian) — volume User & Mitra relatif kecil, tidak butuh desain untuk skala multi-kota di awal.
- **NFR-03 Data integrity dana**: Karena ada model escrow (dana ditahan sebelum payout ke Mitra), sistem harus punya audit trail yang jelas untuk setiap perubahan status dana (paid → held → unlocked → withdrawn / refunded).
- **NFR-04 Auditability foto bukti**: Foto before/after dari Mitra harus tersimpan permanen (tidak bisa dihapus/replace setelah submit) karena jadi evidence untuk resolusi komplain.
- **NFR-05 Security**: Password hashing, session/token management untuk 3 jenis actor (User, Admin, Mitra) — kemungkinan besar butuh auth terpisah per role.

---

## 5. Keputusan Bisnis (Decision Log dari Sesi Brainstorm)

| # | Topik | Keputusan |
|---|---|---|
| 1 | Nama role "Cleaner" | Diganti jadi **Mitra** — scope diperluas ke home services umum, bukan cleaning saja |
| 2 | Model escrow | Platform menahan dana penuh sampai unlock condition (rating ATAU window komplain lewat) |
| 3 | Verifikasi User | Email saja untuk MVP. Nomor HP: validasi format saja, tanpa OTP |
| 4 | Alur Complain | Admin review manual → keputusan refund/reject |
| 5 | Revenue split | Admin input manual harga User & payout Mitra per service (tidak pakai rumus persentase) |
| 6 | Role Owner | Bukan role sistem terpisah — Owner = pemilik platform yang pegang Admin untuk MVP |
| 7 | Platform fee | Flat Rp4.000, otomatis, bukan input manual |
| 8 | Skill matching Mitra | Ya — service & Mitra punya kategori/skill formal untuk membantu Admin pairing |
| 9 | Jadwal order | User pilih tanggal/jam saat order, sebelum bayar |
| 10 | Scope area | Hyper-local — 1 kompleks hunian (Rusun ASN PUPR Pasar Jumat), bukan per kota |
| 11 | "Order prove" | Foto wajib sebelum (start) dan sesudah (complete) kerja Mitra |
| 12 | Rating | 1–5 bintang, opsional; komentar wajib kalau rating ≤ 3 |

---

## 6. User Stories & Acceptance Criteria (Sample — Core Flows)

### US-01: User order service dan bayar
**As a** User, **I want to** memilih service, jadwal, dan membayar via QRIS, **so that** order saya terjadwal dan siap dikerjakan Mitra.

- [ ] User bisa browse service catalog dengan kategori & harga
- [ ] User pilih tanggal & jam sebelum checkout
- [ ] User dapat QR code dinamis (Midtrans) untuk bayar
- [ ] Setelah bayar sukses, status order berubah `waiting_for_payment` → `scheduled`
- [ ] Order muncul di "List order aktif" User

### US-02: Admin pairing Mitra ke order
**As an** Admin, **I want to** melihat order yang sudah dibayar dan assign Mitra yang skill-nya cocok, **so that** order bisa dikerjakan oleh orang yang tepat.

- [ ] Order dengan status `scheduled`/`PAID` muncul di list Admin
- [ ] Admin bisa filter/lihat Mitra berdasarkan kategori skill yang match dengan service order
- [ ] Setelah pairing, Mitra menerima notifikasi WA
- [ ] Order ter-assign ke Mitra tsb, status tetap `scheduled` sampai Mitra mulai kerja

### US-03: Mitra menyelesaikan order dengan bukti foto
**As a** Mitra, **I want to** submit foto sebelum & sesudah kerja, **so that** ada bukti objektif pekerjaan selesai dengan baik.

- [ ] Mitra tidak bisa ubah status ke `in_progress` tanpa upload foto "before"
- [ ] Mitra tidak bisa ubah status ke `complete` tanpa upload foto "after"
- [ ] Foto tersimpan permanen, terkait ke order tsb
- [ ] Setelah `complete`, window komplain 3 hari mulai berjalan
- [ ] Mitra menerima notifikasi WA konfirmasi setiap perubahan status

### US-04: Resolusi Complain & withdrawal Mitra
**As an** Admin, **I want to** review komplain yang masuk dan putuskan refund/tolak, **so that** dana Mitra hanya cair untuk order yang valid.

- [ ] Complain dari User (dalam window 3 hari) masuk status `pending_review`
- [ ] Selama `pending_review`, payout Mitra untuk order tsb **tidak bisa** di-withdraw
- [ ] Admin putuskan refund (penuh/sebagian) atau tolak
- [ ] Kalau tidak ada komplain sampai window 3 hari habis (atau User sudah kasih rating lebih dulu), payout Mitra otomatis unlocked untuk withdrawal

---

## 7. Open Questions / TODO untuk Fase Berikutnya

Item berikut **belum diputuskan** dan perlu dijawab sebelum atau selama fase design/implementasi:

1. **[TEKNIS]** Apakah Midtrans API mendukung **automatic disbursement/payout** ke rekening Mitra, atau payout harus dilakukan manual (transfer manual oleh Admin/Owner di luar sistem, lalu status "withdrawn" diupdate manual)? — Perlu riset API Midtrans di fase design.
2. **Registrasi Mitra**: Apakah Mitra self-register (lalu diverifikasi/approve Admin) atau didaftarkan langsung oleh Admin (invite-only)? Data apa saja yang dibutuhkan (rekening bank untuk payout, dokumen identitas, dll)?
3. **Refund mechanism**: Kalau Admin approve refund, apakah refund dieksekusi otomatis via Midtrans API, atau manual transfer ke User?
4. **Order cancellation**: Apakah User bisa cancel order sebelum dibayar / setelah dibayar tapi sebelum Mitra mulai kerja? Apa konsekuensinya (refund otomatis)?
5. **No Mitra available**: Kalau tidak ada Mitra dengan skill yang cocok/available di jadwal yang dipilih User, apa yang terjadi? (auto-reject, Admin reschedule, dll)
6. **Multiple order per Mitra**: Apakah satu Mitra bisa pegang beberapa order dalam waktu bersamaan/hari yang sama, atau one-at-a-time?
7. **Evidence komplain**: Apakah User bisa upload foto/bukti tambahan saat mengajukan komplain, atau Admin hanya mengandalkan foto before/after dari Mitra?
8. **Role Owner/Admin di masa depan**: Kapan dan bagaimana pemisahan role Owner (financial report, config) vs Admin operasional (assign order) akan diaktifkan?
9. **Ekspansi lokasi**: Struktur data untuk lokasi (nama kompleks, unit/tower/lantai) — perlu dirancang supaya mudah ditambah lokasi baru di masa depan meski MVP cuma 1 lokasi?
10. **Alamat dalam 1 kompleks**: Bagaimana User menentukan alamat spesifik (nomor unit/tower/lantai) dalam Rusun ASN PUPR Pasar Jumat saat order?

---

## 8. Boundaries — Yang TIDAK Termasuk di Dokumen Ini

Sesuai scope `/sa:brainstorm`, dokumen ini **hanya requirements**, tidak termasuk:
- Desain arsitektur sistem / diagram (→ `/sa:design`)
- Skema database / API contract
- Kode implementasi (→ `/sa:implement`)
- Keputusan teknis detail (misal struktur service Go, desain tabel MySQL, strategi caching Redis)

**Next step yang disarankan**: `/sa:design` untuk merancang arsitektur sistem (termasuk menjawab open question #1 soal Midtrans disbursement), dilanjutkan `/sa:workflow` untuk implementation planning.
