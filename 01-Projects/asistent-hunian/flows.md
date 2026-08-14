# Sequence Diagrams — Asisten Hunian

> Turunan dari `requirements.md` (hasil `/sa:brainstorm`, 2026-08-10).
> Dikelompokkan per actor. Setiap diagram mengacu ke FR/US terkait.
> Bagian yang masih bergantung pada Open Question (`requirements.md` §7) ditandai ⚠️.

---

## 1. User

### 1.1 Registrasi & Verifikasi Email
*FR-U01, FR-U02*

```mermaid
sequenceDiagram
    actor User
    participant Sistem
    participant Email as Email Service

    User->>Sistem: Submit form registrasi (email, password, no HP*)
    Sistem->>Sistem: Validasi format no HP (regex, tanpa OTP)
    Sistem->>Sistem: Hash password, simpan user (status: unverified)
    Sistem->>Email: Kirim email verifikasi (link/kode)
    Email-->>User: Email verifikasi diterima
    User->>Sistem: Klik link / submit kode verifikasi
    Sistem->>Sistem: Update status user -> verified
    Sistem-->>User: Verifikasi sukses, bisa login
```

### 1.2 Browse Catalog, Order & Pembayaran QRIS
*FR-U03, FR-U04, FR-U05*

```mermaid
sequenceDiagram
    actor User
    participant Sistem
    participant Midtrans

    User->>Sistem: Login
    Sistem-->>User: Session/token
    User->>Sistem: GET service catalog (per kategori)
    Sistem-->>User: List service + harga
    User->>Sistem: Pilih service + tanggal & jam
    User->>Sistem: Isi alamat (Tower/Unit/Lantai - field bebas)
    User->>Sistem: Submit order (status: waiting_for_payment)
    Sistem->>Midtrans: Request QRIS dinamis (amount = harga service)
    Midtrans-->>Sistem: QR code / payment token
    Sistem-->>User: Tampilkan QR code
    alt bayar sebelum expiry
        User->>Midtrans: Scan & bayar QRIS
        Midtrans-->>Sistem: Webhook notifikasi payment success
        Sistem->>Sistem: Update order status -> scheduled (PAID)
        Sistem-->>User: Notifikasi order terjadwal
    else expiry default Midtrans terlewati tanpa bayar
        Midtrans-->>Sistem: Webhook notifikasi expired
        Sistem->>Sistem: Update order status -> expired (slot jadwal free lagi)
        Sistem-->>User: Notifikasi order expired, silakan order ulang
    end
```

### 1.3 Lihat Order Aktif / Selesai
*FR-U06, FR-U07, FR-U08*

```mermaid
sequenceDiagram
    actor User
    participant Sistem

    User->>Sistem: GET list order (filter: aktif / selesai)
    Sistem-->>User: List order + status ringkas
    User->>Sistem: GET order detail (order_id)
    Sistem-->>User: Detail (service, harga, jadwal, Mitra, status, foto before/after jika ada)
```

### 1.4 Rating Order
*FR-U09*

```mermaid
sequenceDiagram
    actor User
    participant Sistem

    User->>Sistem: GET order detail (status: complete)
    Sistem-->>User: Detail order + foto before/after
    User->>Sistem: Submit rating (1-5) [+ komentar wajib jika <=3]
    alt rating <= 3 tanpa komentar
        Sistem-->>User: Reject - komentar wajib
    else valid
        Sistem->>Sistem: Simpan rating
        Sistem->>Sistem: Tutup window complain untuk order ini
        Sistem->>Sistem: Unlock payout Mitra untuk order ini
        Sistem-->>User: Rating tersimpan
    end
```

### 1.5 Ajukan Complain
*FR-U10*

```mermaid
sequenceDiagram
    actor User
    participant Sistem
    actor Admin

    User->>Sistem: Ajukan complain + foto/bukti tambahan (opsional), dalam window 3 hari sejak complete
    Sistem->>Sistem: Cek: sudah pernah rating order ini? Window komplain masih berlaku?
    alt sudah rating, atau window sudah lewat
        Sistem-->>User: Reject - hak complain sudah tertutup (rating diberikan / window habis)
    else belum rating & masih dalam window
        Sistem->>Sistem: Simpan complain (status: pending_review)
        Sistem->>Sistem: Block withdrawal payout Mitra untuk order ini
        Sistem-->>Admin: Notifikasi complain baru masuk
        Sistem-->>User: Complain diterima, menunggu review
    end
```

---

## 2. Admin

### 2.1 Pairing Mitra ke Order
*FR-A03, FR-A04*

```mermaid
sequenceDiagram
    actor Admin
    participant Sistem
    participant WA as WhatsApp Gateway
    actor Mitra

    Admin->>Sistem: Login
    Admin->>Sistem: GET list order aktif (status scheduled/PAID, belum ter-assign)
    Sistem-->>Admin: List order + detail
    Admin->>Sistem: GET list Mitra (filter kategori/skill match service)
    Sistem-->>Admin: List Mitra tersedia
    Admin->>Sistem: Assign Mitra X ke Order Y
    Sistem->>Sistem: Cek bentrok jadwal Mitra X (jam mulai + durasi service vs order aktif lain)
    alt bentrok jadwal
        Sistem-->>Admin: Reject - Mitra sudah punya order aktif di jam tsb
    else tidak bentrok
        Sistem->>Sistem: Update order.mitra_id (status order tetap scheduled)
        Note over Sistem,Mitra: Assign final -- tidak ada accept/reject dari Mitra
        Sistem->>WA: Kirim notifikasi WA ke Mitra (order baru, info saja)
        WA-->>Mitra: Notifikasi order diterima
        Sistem-->>Admin: Konfirmasi pairing sukses
    end
```

### 2.2 Kelola Service Catalog
*FR-A05*

```mermaid
sequenceDiagram
    actor Admin
    participant Sistem

    Admin->>Sistem: Login
    Admin->>Sistem: Tambah/edit service (nama, kategori/skill, harga User, payout Mitra - manual, estimasi durasi)
    Sistem->>Sistem: Validasi & simpan service
    Sistem-->>Admin: Service tersimpan, tampil di catalog User
```

### 2.3 Review Complain
*FR-A06, US-04*

```mermaid
sequenceDiagram
    actor Admin
    participant Sistem
    actor User
    actor Mitra

    Admin->>Sistem: GET list complain (status: pending_review)
    Sistem-->>Admin: List complain + detail order + foto before/after
    Note over Admin,Sistem: Tidak ada SLA/timeout eksplisit — payout Mitra tertahan sampai Admin memutuskan, kapan pun itu
    Admin->>Admin: Investigasi manual
    Admin->>Sistem: Putuskan (refund penuh / sebagian / tolak)
    alt refund
        Note over Sistem: Dana refund dari margin platform — payout Mitra tidak dipotong
        Admin->>Admin: Transfer manual ke User (di luar sistem)
        Admin->>Sistem: Update status refund -> transferred
        Sistem-->>User: Notifikasi refund diproses
        Sistem->>Sistem: Update complain status -> resolved_refund
    else tolak
        Sistem-->>User: Notifikasi complain ditolak
        Sistem->>Sistem: Update complain status -> resolved_rejected
        Sistem->>Sistem: Unlock payout Mitra untuk order ini
    end
```

---

## 3. Mitra

### 3.1 Registrasi Mitra
*FR-M01*

```mermaid
sequenceDiagram
    actor Mitra
    participant Sistem
    actor Admin

    Mitra->>Sistem: Self-register (data diri, kategori/skill, rekening bank)
    Sistem->>Sistem: Simpan Mitra (status: pending_approval)
    Sistem-->>Admin: Notifikasi Mitra baru menunggu approval
    Admin->>Sistem: Review & approve/reject
    Sistem->>Sistem: Update status Mitra -> active/rejected
    Sistem-->>Mitra: Notifikasi hasil approval
```

### 3.2 Start Order (Upload Foto Before)
*FR-M02*

```mermaid
sequenceDiagram
    actor Mitra
    participant Sistem

    Mitra->>Sistem: GET order assigned (status: scheduled)
    Sistem-->>Mitra: Detail order
    Mitra->>Sistem: Upload foto "before" + request start
    alt foto belum diupload
        Sistem-->>Mitra: Reject - foto before wajib
    else foto ada
        Sistem->>Sistem: Simpan foto (permanen), update status -> in_progress
        Sistem-->>Mitra: Order in_progress
    end
```

### 3.3 Complete Order (Upload Foto After)
*FR-M03*

```mermaid
sequenceDiagram
    actor Mitra
    participant Sistem
    actor User

    Mitra->>Sistem: Upload foto "after" + request complete<br/>(begitu kerja aktif selesai - hasil belum tentu kering/jadi)
    alt foto belum diupload
        Sistem-->>Mitra: Reject - foto after wajib
    else foto ada
        Sistem->>Sistem: Simpan foto (permanen), update status -> complete
        Sistem->>Sistem: Mulai window komplain/rating (3 hari)
        Sistem->>Sistem: Mitra otomatis available lagi untuk order lain
        Sistem-->>Mitra: Notifikasi WA order complete
        Sistem-->>User: Notifikasi order selesai (bisa rating/complain)
    end
```

> Catatan: untuk servis dengan waktu tunggu pasif (mis. cuci karpet), Mitra tidak perlu balik lagi setelah hasil kering — foto "after" cukup diambil begitu kerja aktif kelar (lihat Decision Log #24–25 di `requirements.md`).

### 3.4 Withdrawal Payout
*FR-M04*

```mermaid
sequenceDiagram
    actor Mitra
    participant Sistem
    actor Admin

    Mitra->>Sistem: Request withdrawal
    Sistem->>Sistem: Cek unlock condition per order<br/>(rating ada ATAU window 3 hari lewat tanpa complain)
    alt ada complain pending
        Sistem-->>Mitra: Reject - dana masih ditahan (complain pending review)
    else unlocked
        Sistem->>Sistem: Hitung total payout unlocked, catat request (status: requested)
        Sistem-->>Admin: Notifikasi ada request withdrawal
        Admin->>Admin: Transfer manual ke rekening Mitra (di luar sistem)
        Admin->>Sistem: Update status payout -> withdrawn
        Sistem-->>Mitra: Notifikasi withdrawal sukses
    end
```

### 3.5 Login Mitra
*FR-M05*

```mermaid
sequenceDiagram
    actor Mitra
    participant Sistem

    Mitra->>Sistem: Submit email + password
    Sistem->>Sistem: Verifikasi kredensial (sama pola auth dengan User/Admin)
    alt kredensial valid
        Sistem-->>Mitra: Session/token
    else invalid
        Sistem-->>Mitra: Reject - email/password salah
    end
```

> Catatan: Mitra login pakai email + password, kredensial sama yang diisi saat registrasi (FR-M01). Notifikasi operasional (order baru, dll) tetap lewat WhatsApp — terpisah dari mekanisme login (lihat Decision Log #30).

> Catatan: Midtrans **Payouts API** (dulu Iris) mendukung disbursement otomatis real-time ke rekening bank — dipertimbangkan sebagai improvement pasca-MVP untuk menggantikan langkah transfer manual Admin di atas.

---

## 4. Shared (Semua Role)

### 4.1 Reset Password
*FR-C07 — berlaku untuk User, Admin, Mitra*

```mermaid
sequenceDiagram
    actor Actor as User/Admin/Mitra
    participant Sistem
    participant Email as Email Service

    Actor->>Sistem: Request "lupa password" (submit email)
    Sistem->>Sistem: Cek email terdaftar
    Sistem->>Email: Kirim link reset password
    Email-->>Actor: Email reset diterima
    Actor->>Sistem: Klik link, submit password baru
    Sistem->>Sistem: Validasi token reset, hash & update password
    Sistem-->>Actor: Password berhasil direset, bisa login
```

> Catatan: Self-service, reuse pola verifikasi email User (FR-U02) — satu mekanisme untuk ketiga role (lihat Decision Log #31).

---

## Status Keputusan

Semua flow di atas sudah mencerminkan keputusan final di `requirements.md` §5 (Decision Log #13–31), hasil resolusi Open Questions §7 (2026-08-11) dan sesi lanjutan grey-area (2026-08-14). Satu-satunya item yang masih terbuka: **#8 — pemisahan role Owner/Admin** (deferred, belum berdampak ke diagram MVP karena Owner memegang Admin sendiri).
