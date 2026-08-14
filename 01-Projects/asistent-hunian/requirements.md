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
- **FR-U05 Order**: User pilih service, **pilih tanggal & jam** (scheduling terjadi di titik order, sebelum bayar), isi **alamat** (field bebas: Tower/Unit/Lantai — tidak ada validasi ketersediaan Mitra di titik ini), lalu bayar via QRIS dinamis (Midtrans). Kalau QRIS tidak dibayar sampai **expiry default Midtrans**, order otomatis pindah status `expired` (lihat FR-C06) dan slot jadwal free lagi — User perlu order ulang.
- **FR-U06 List order aktif**: Menampilkan order dengan status: `waiting_for_payment`, `scheduled`, `in_progress`. Order `expired` **tidak** termasuk di list ini (dianggap gagal/batal, bukan aktif maupun selesai).
- **FR-U07 List order selesai**: Order dengan status `complete` (termasuk yang sudah/belum direview/komplain).
- **FR-U08 Order detail**: Detail satu order — service, harga, jadwal, Mitra assigned, status, foto bukti before/after (kalau sudah ada).
- **FR-U09 Rating**: Setelah order complete, User bisa kasih rating 1–5 bintang (opsional). **Kalau rating ≤ 3, komentar wajib diisi.** Rating bukan syarat wajib untuk order dianggap selesai di sisi User. **Begitu rating diberikan, window complain untuk order tsb otomatis tertutup** — User tidak bisa ajukan complain lagi setelah rating (lihat FR-U10).
- **FR-U10 Complain**: User bisa ajukan komplain dalam window 3 hari setelah order complete, **selama belum memberi rating untuk order tsb** (rating menutup window complain — lihat FR-U09), dengan **upload foto/bukti tambahan (opsional)**. Complain masuk status pending untuk direview manual oleh Admin.

### 3.2 Admin

- **FR-A01 Login**.
- **FR-A02 List Mitra**: Melihat daftar Mitra terdaftar beserta kategori/skill masing-masing.
- **FR-A03 List order aktif**: Melihat semua order yang perlu ditindaklanjuti (termasuk yang sudah dibayar dan butuh pairing).
- **FR-A04 Pairing order ke Mitra**: Setelah order berstatus `PAID`, Admin pilih Mitra yang sesuai (difilter berdasarkan kategori/skill yang match dengan service order) untuk di-assign. Sistem **memvalidasi tidak ada bentrok jadwal** — rentang waktu order (jam mulai + estimasi durasi service, lihat FR-A05) tidak boleh tumpang tindih dengan order aktif (`scheduled`/`in_progress`) lain milik Mitra yang sama; assign ditolak kalau bentrok. **Assign dianggap final** — tidak ada langkah accept/reject dari Mitra; notifikasi WA bersifat info, bukan permintaan konfirmasi (lihat Decision Log #29).
- **FR-A05 Kelola service catalog**: Admin bisa tambah/edit/**nonaktifkan** service — set harga yang ditampilkan ke User, set payout untuk Mitra (**diinput manual**, tidak pakai rumus persentase otomatis), set kategori/skill service tsb, dan set **estimasi durasi kerja aktif** (tidak termasuk waktu tunggu/curing pasif seperti keringnya karpet — dipakai untuk validasi bentrok jadwal Mitra di FR-A04). Service **tidak bisa dihapus permanen** (soft-delete via nonaktifkan saja), supaya order lama yang reference ke service tsb tetap valid (lihat NFR-06).
- **FR-A06 Review Complain**: Admin melihat komplain masuk (termasuk foto bukti dari User kalau ada), investigasi manual, putuskan **refund penuh/sebagian atau tolak**. Selama status pending, dana payout Mitra terkait tetap ditahan (block withdrawal). Kalau refund disetujui, dana diambil dari **margin platform** (tidak memotong payout Mitra) dan ditransfer **manual** oleh Admin di luar sistem — sistem hanya mencatat status refund. **Tidak ada SLA/timeout eksplisit** untuk review ini — payout tertahan sampai Admin memutuskan, kapan pun itu (lihat Decision Log #26).

**Catatan — No-show**: Penanganan no-show (Mitra tidak datang / User tidak di lokasi saat jadwal) ditangani **manual oleh Admin di luar sistem**, sama seperti order cancellation (#14) — tidak ada status order atau FR khusus untuk no-show di MVP (lihat Decision Log #27).

### 3.3 Mitra

- **FR-M01 Registrasi Mitra**: Mitra **self-register** (data diri, kategori/skill, rekening bank) dengan status awal `pending_approval`. Admin review dan approve/reject sebelum Mitra bisa menerima order.
- **FR-M02 Start order**: Mitra mulai kerja setelah **upload foto bukti kondisi "before"** — wajib, jadi syarat status order berubah ke `in_progress`.
- **FR-M03 Complete order**: Mitra selesaikan order setelah **upload foto bukti hasil "after"** — wajib, jadi syarat status order berubah ke `complete`. Untuk servis dengan hasil yang butuh waktu tunggu pasif (mis. cuci karpet yang keringnya lama), foto "after" diambil **begitu kerja aktif selesai** (belum tentu hasil akhir sudah kering/jadi) — Mitra tidak perlu balik lagi, dan langsung available untuk order lain setelahnya.
- **FR-M04 Withdrawal**: Mitra bisa request withdraw payout setelah **salah satu terpenuhi lebih dulu**: (a) User memberi rating, atau (b) window komplain 3 hari lewat tanpa komplain diajukan. Kalau ada komplain pending, withdrawal untuk order tsb diblokir sampai Admin memutuskan. Pencairan dana **manual** oleh Admin/Owner di luar sistem (transfer bank) untuk MVP — sistem update status `withdrawn` setelah transfer dilakukan.
- **FR-M05 Login**: Mitra login dengan **email + password**, kredensial yang sama diisi saat registrasi (FR-M01) — pola auth sama seperti User (FR-U03) dan Admin (FR-A01) (lihat Decision Log #30).

### 3.4 Konfigurasi (Config)

- **FR-C01 Platform fee**: Flat **Rp4.000** per transaksi, dihitung otomatis oleh sistem (tidak diinput manual per service).
- **FR-C02 Window komplain & rating**: 3 hari sejak order `complete`, dapat dikonfigurasi.
- **FR-C03 Payment gateway**: QRIS dinamis via **Midtrans**.
- **FR-C04 Notifikasi**: Email ke User (registrasi, verifikasi, status order, dll). WhatsApp ke Mitra (konfirmasi order start & complete).
- **FR-C05 Service & skill category**: Master data kategori service (Cleaning, Install Heater, dll) yang juga dipakai sebagai skill tag Mitra untuk keperluan pairing (FR-A04).
- **FR-C06 Order expiry QRIS**: Order dengan status `waiting_for_payment` yang tidak dibayar sampai **expiry default Midtrans** otomatis pindah status `expired` lewat webhook Midtrans — tidak ada timer/durasi custom yang dikonfigurasi manual oleh Admin.
- **FR-C07 Reset Password**: User, Admin, dan Mitra bisa request reset password lewat **email reset link** self-service — reuse pola verifikasi email User (FR-U02) (lihat Decision Log #31).

---

## 4. Non-Functional Requirements

- **NFR-01 Tech stack** *(constraint dari user, bukan hasil eksplorasi)*: Go (backend), MySQL (primary datastore), Redis (caching/session/queue).
- **NFR-02 Scale**: MVP untuk 1 lokasi (1 kompleks hunian) — volume User & Mitra relatif kecil, tidak butuh desain untuk skala multi-kota di awal.
- **NFR-03 Data integrity dana**: Karena ada model escrow (dana ditahan sebelum payout ke Mitra), sistem harus punya audit trail yang jelas untuk setiap perubahan status dana (paid → held → unlocked → withdrawn / refunded).
- **NFR-04 Auditability foto bukti**: Foto before/after dari Mitra harus tersimpan permanen (tidak bisa dihapus/replace setelah submit) karena jadi evidence untuk resolusi komplain.
- **NFR-05 Security**: Password hashing, session/token management untuk 3 jenis actor (User, Admin, Mitra) — kemungkinan besar butuh auth terpisah per role.
- **NFR-06 Referential integrity histori order**: Order harus tetap bisa menampilkan detail service (nama, harga, payout) walau service tsb sudah dinonaktifkan Admin — service catalog tidak boleh hard-delete (lihat Decision Log #21).

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
| 13 | Registrasi Mitra | Self-register (isi data, kategori/skill, rekening bank) → status `pending_approval` → Admin approve/reject |
| 14 | Order cancellation | Belum didukung di MVP — pembatalan ditangani manual oleh Admin di luar sistem |
| 15 | No Mitra match | Tidak ada validasi ketersediaan Mitra saat checkout. Order tetap `scheduled` tanpa Mitra ter-assign, Admin follow-up manual |
| 16 | Concurrency Mitra | One Mitra satu order aktif per jam yang tumpang tindih — divalidasi sistem saat Admin pairing |
| 17 | Disbursement Mitra | Manual transfer oleh Admin/Owner di luar sistem untuk MVP; status `withdrawn` diupdate manual. Midtrans **Payouts API** (dulu Iris) sudah mendukung transfer real-time ke rekening bank — dipertimbangkan untuk integrasi di fase berikutnya |
| 18 | Refund | Manual transfer oleh Admin ke User untuk MVP, bukan via Midtrans refund API |
| 19 | Bukti complain | User bisa upload foto/bukti tambahan (opsional) saat ajukan complain, selain foto before/after dari Mitra |
| 20 | Struktur alamat | Field bebas teks (Tower/Unit/Lantai) diisi User saat order — bukan struktur dropdown relasional, supaya mudah diperluas ke multi-lokasi nanti |
| 21 | Soft-delete service catalog | Service **tidak pernah di-hard-delete**. Admin hanya bisa nonaktifkan (`is_active = false`) — service hilang dari catalog yang dilihat User, tapi record tetap ada supaya order lama yang reference ke service tsb tidak orphan/berdiri sendiri |
| 22 | Rating vs. hak complain | Begitu User kasih rating, window complain untuk order tsb **otomatis tertutup**. User harus pilih: rating dulu (berarti puas, payout Mitra langsung unlock) atau tahan dulu kalau masih ragu/ingin komplain |
| 23 | Sumber dana refund | Refund ke User **selalu dari margin platform** (Owner), tidak memotong payout Mitra — dua arah uang (refund ke User vs payout ke Mitra) independen |
| 24 | Durasi service | Admin input **estimasi durasi kerja aktif** (menit/jam) per service saat kelola catalog (FR-A05) — **tidak termasuk waktu tunggu/curing pasif** (mis. keringnya karpet setelah dicuci). Dipakai sistem untuk hitung rentang waktu order saat validasi bentrok jadwal Mitra (FR-A04) |
| 25 | Foto "after" untuk servis dengan waktu tunggu | Diambil **begitu kerja aktif selesai** (hasil belum tentu kering/jadi sepenuhnya, mis. karpet masih basah), bukan menunggu Mitra balik lagi. Order langsung `complete`, Mitra langsung available untuk order lain. Kualitas hasil akhir dinilai User sendiri lewat rating |
| 26 | SLA review complain oleh Admin | **Tidak ada SLA/timeout eksplisit.** Admin (Owner sendiri) review kapan saja secara manual; payout Mitra tetap tertahan sampai keputusan keluar, seberapa pun lamanya. Tidak ada auto-resolve, supaya tidak ada keputusan refund/reject tanpa investigasi |
| 27 | Penanganan no-show (Mitra tidak datang / User tidak di lokasi) | Ditangani **manual oleh Admin di luar sistem** — User/Mitra kontak Admin langsung (WA/telepon), Admin putuskan & proses refund manual kalau perlu. Tidak ada status atau FR khusus untuk no-show di MVP, konsisten dengan pola order cancellation (#14) |
| 28 | Expiry order QRIS belum dibayar | Pakai **expiry default dari Midtrans** (bukan custom timer sistem). Sistem dengarkan webhook expired dari Midtrans, update status order -> `expired`. Tidak perlu konfigurasi durasi manual oleh Admin |
| 29 | Mitra accept/reject order | **Assign dianggap final** — tidak ada konfirmasi accept/reject dari Mitra. Notifikasi WA (FR-A04) bersifat info, bukan permintaan konfirmasi. Kalau Mitra berhalangan mendadak, ditangani manual (kontak Admin di luar sistem, konsisten #14/#27) |
| 30 | Kredensial login Mitra | **Email + password**, sama seperti User dan Admin (FR-U01–U03, FR-A01) — satu pola auth untuk semua role. Notifikasi operasional Mitra tetap lewat WhatsApp (FR-C04), terpisah dari mekanisme login |
| 31 | Forgot/reset password | **Email reset link**, self-service, berlaku untuk User, Admin, dan Mitra — reuse pola verifikasi email User (FR-U02) |

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

Resolusi sesi lanjutan (2026-08-11) — lihat Decision Log #13–20 di §5 untuk detail keputusan:

1. ✅ **[TEKNIS] Disbursement Midtrans** — Diriset: Midtrans **Payouts API** (dulu Iris) mendukung transfer real-time ke rekening bank (`create transfer` → `approve transfer`). Keputusan MVP: tetap **manual** dulu (lihat #17); integrasi Payouts API jadi kandidat improvement fase berikutnya.
2. ✅ **Registrasi Mitra** — Self-register + approval Admin. Lihat #13.
3. ✅ **Refund mechanism** — Manual transfer oleh Admin, bukan via Midtrans API. Lihat #18.
4. ✅ **Order cancellation** — Belum didukung di MVP, ditangani manual oleh Admin. Lihat #14.
5. ✅ **No Mitra available** — Tidak ada validasi ketersediaan saat checkout; order tetap pending, Admin follow-up manual. Lihat #15.
6. ✅ **Multiple order per Mitra** — One-at-a-time, divalidasi sistem (tolak assign kalau jadwal bentrok). Lihat #16.
7. ✅ **Evidence komplain** — User bisa upload foto/bukti tambahan (opsional). Lihat #19.
8. ⏳ **Role Owner/Admin di masa depan** — **Masih terbuka / deferred.** Bukan blocker MVP karena Owner memegang Admin sendiri untuk sekarang. Revisit saat sudah merekrut Admin operasional terpisah — perlu desain role permission granular (financial report vs operational assign) di fase itu.
9. ✅ **Ekspansi lokasi** — Arah keputusan: alamat disimpan sebagai field bebas (bukan struktur relasional dropdown), supaya gampang ditambah field "kompleks" saat ekspansi ke lokasi lain. Detail skema tabel tetap didesain nanti di `/sa:design`. Lihat #20.
10. ✅ **Alamat dalam 1 kompleks** — Field bebas teks: Tower/Unit/Lantai, diisi User saat order. Lihat #20.

---

## 8. Boundaries — Yang TIDAK Termasuk di Dokumen Ini

Sesuai scope `/sa:brainstorm`, dokumen ini **hanya requirements**, tidak termasuk:
- Desain arsitektur sistem / diagram (→ `/sa:design`)
- Skema database / API contract
- Kode implementasi (→ `/sa:implement`)
- Keputusan teknis detail (misal struktur service Go, desain tabel MySQL, strategi caching Redis)

**Next step yang disarankan**: `/sa:design` untuk merancang arsitektur sistem (termasuk menjawab open question #1 soal Midtrans disbursement), dilanjutkan `/sa:workflow` untuk implementation planning.
