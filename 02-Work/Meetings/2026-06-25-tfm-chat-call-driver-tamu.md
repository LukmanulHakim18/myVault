---
title: Meeting Prep - TFM Call & Chat Pengemudi & Tamu
date: '2026-06-25'
team: MRG
tags:
  - meeting-prep
  - tfm
  - chat
  - voip
  - birdtalk
  - integration
status: draft
---
# Meeting Prep - TFM Call & Chat Pengemudi & Tamu

**Tanggal:** 2026-06-25 (meeting jam 16:00)
**Pengundang:** Ardi
**Topik:** Diskusi lanjutan TFM untuk fitur **call & chat** antara pengemudi dan tamu
**Tim terdampak:** MRG, BBD, DA, TFM, BBOne core
**Konteks:** TFM = Taxi Fleet Management

---

## 🔑 Temuan Kunci: Platform Sudah Ada — BirdTalk

Bluebird **sudah punya chat platform produksi**: **BirdTalk** (squad **core**, PIC **Ivanov Adie Ananda** — ivanov.adie@bluebirdgroup.com). Maka ini **bukan greenfield — ini integrasi/ekstensi**.

| Komponen | Peran |
|---|---|
| `chat-service` (Go 1.23, gRPC :16969, REST :8443) | Orchestration: **1 chat room per order**, quick-message template, system broadcast, token lifecycle |
| `chat-engine` (Tinode) | Backbone real-time — one-on-one, group, **voice/video call**, presence, delivery receipt |
| `notification-service` | Push FCM (`SaveDeviceToken` sudah ada) |

**Fakta penting dari KB (services/core/chat-service):**
- Chat **customer↔driver per-order sudah jalan** (SLO 99.9%, p99 <200ms, Tier P1, Production).
- **VoIP/call sudah ada sebagai feature flag** (`CHAT_VOIP_FF_KEY` via Flipt) — "call" kemungkinan = *aktivasi/integrasi*, bukan bangun dari nol.
- Model identitas BirdTalk = `customer↔driver`. **"Tamu" vs "customer"** jadi pertanyaan identitas terbesar.
- Message broker pluggable (Kafka / GCP PubSub / RabbitMQ via YAML).
- Notification host nunjuk `bbone-notification-token` → BBOne = namespace platform/notifikasi pusat.
- API existing relevan: `Register`, `CreateRoom`, `SendMessage`, `GetParticipants`, `SaveDeviceToken`, `SystemSendMessage`, `RemoveSession`, CRUD `QuickMessage`.

> **Implikasi:** keputusan "build vs buy" praktis terjawab → **reuse BirdTalk**. Meeting ini soal **bagaimana MRG/BBD/DA/TFM nyolok ke chat-service core**.

---

## 🗺️ Peta Dampak Lintas-Tim → Pertanyaan Integrasi

> ⚠️ Peran sebagian tim ini **hipotesis dari KB** — konfirmasi peta ini sendiri di awal meeting.

| Tim | Peran (hipotesis) | Pertanyaan integrasi kritis |
|---|---|---|
| **core (BirdTalk)** | Owner `chat-service` + `chat-engine` | Perlu perubahan kontrak gRPC (`CreateRoom`/`Register`) untuk identitas "tamu" TFM? Kapasitas nambah dari volume TFM? PIC Ivanov hadir? |
| **BBOne core** | Platform pusat + notification (config `bbone-notification-token`) | Source-of-truth order/trip event di BBOne? Routing push notif lewat sini? |
| **BBD** | Dispatch / job (KB: "BBD CreateNewJob", "BBD SOAP dispatch") | **Event apa yang memicu `CreateRoom`?** Saat job dibuat / driver assigned? BBD yang trigger ke chat-service? Identitas driver dari sini? |
| **TFM** | Taxi Fleet Management — konteks order ini | Order TFM beda alur dari order MyBlueBird standar? Siapa "tamu" di TFM (mungkin bukan user app)? |
| **DA** | *(KONFIRMASI — Driver App? Data/Analytics?)* | Kalau Driver App: integrasi SDK chat sisi driver. Kalau Analytics: logging/retensi chat. **Tanya: DA itu apa.** |
| **MRG** *(tim kita)* | *(KONFIRMASI titik singgung)* | Kenapa MRG kena? Orkestrasi multi-fleet/order? Sisi customer app? |

---

## ☎️ Garpu Kritis: In-app VoIP vs PSTN + Number Masking

Dua jalur ini **beda total** (infra, vendor, biaya, regulasi):

| | **A. In-app VoIP (WebRTC/Tinode)** | **B. PSTN call + number masking** |
|---|---|---|
| Cara kerja | Panggilan data via app, reuse Tinode existing | Panggilan seluler asli, nomor disamarkan via telco/SIP |
| Syarat "tamu" | **Harus punya app & online** | Bisa **tanpa app** (telpon biasa) |
| Vendor | Reuse Tinode + TURN/STUN | Butuh provider (Twilio/Vonage/telco lokal) — **biaya per menit** |
| Cocok kalau | Tamu = user MyBlueBird | Tamu = penumpang non-app / dibooking-kan |

> **Pertanyaan pembuka call:** "Tamu" pasti punya app & online? Kalau tidak → wajib jalur B, dan VoIP Tinode existing **tidak cukup**.

---

## ❓ Daftar Pertanyaan Kritis

### P0 — Wajib dijawab hari ini
1. **Reuse BirdTalk atau bangun baru?** Kalau reuse → ini meeting integrasi, bukan desain platform.
2. **Peta ownership di atas benar?** Siapa producer event, siapa consumer, siapa owner kontrak.
3. **Definisi "tamu"** di TFM — sama dengan `customer` BirdTalk, atau aktor baru (model identitas/registrasi baru)?
4. **Trigger `CreateRoom`** — event mana (job created / driver assigned) dan dari sistem mana (BBD? BBOne?).
5. **Scope call:** jalur A atau B? VoIP FF existing itu sudah WebRTC Tinode atau placeholder? (tanya core/Ivanov)
6. **Number masking wajib** untuk chat & call? Provider mana yang dipakai/dievaluasi?
7. **Diskusi lanjutan** — keputusan apa yang sudah closed di sesi sebelumnya (jangan buka ulang).

### P1 — Desain integrasi
8. Protokol antar-tim ke chat-service: gRPC langsung atau event (Kafka/PubSub — sudah pluggable)?
9. **Quick-message template** TFM — siapa manage (CRUD `QuickMessage` sudah ada)?
10. Reachability "tamu" kalau bukan user app / app background — fallback (SMS/PSTN)?
11. **Recording panggilan** untuk dispute/audit — perlu? Consent + retensi + kepatuhan (data sensitif).
12. **Incoming call saat app background/locked** — CallKit (iOS) / ConnectionService (Android), wake via push.
13. **Jaringan jelek** (basement/terowongan) — fallback call gagal → PSTN atau drop ke chat?
14. **TURN/STUN** (kalau jalur A) — siapa sediakan, host di mana (sejalan K8s)?
15. **Retensi & audit chat** untuk dispute — kebijakan TFM vs default BirdTalk (MySQL/Mongo).
16. **Dampak SLO core** — volume TFM mempengaruhi p99 <200ms / 99.9%?
17. **Model biaya** — VoIP data (murah) vs PSTN per-menit (cost, skalakan ke volume fleet TFM).

### P2 — Lanjutan
18. **Regulasi telephony Indonesia** + consent rekaman — libatkan legal?
19. **Metrik sukses** — % trip pakai chat/call, turunnya telpon CS, waktu pickup?
20. **Pilot** — fleet/area mana dulu, phased rollout?

---

## 🎯 Saran Taktis Meeting

- **30 detik pertama:** *"Sebelum bahas desain — ini reuse BirdTalk (chat-service core) atau bangun baru? Dan boleh diluruskan peran masing-masing tim: siapa producer event, siapa consumer?"*
- **Pastikan PIC core (Ivanov) hadir** — semua pertanyaan call/VoIP & kontrak gRPC mengarah ke mereka.
- **Titik keputusan terbesar:** definisi "tamu" (#3), garpu call A/B (#5), number masking (#6). Ketiganya nentuin effort & timeline.
- Kalau jalur **B (PSTN masking)** → muncul **aktor baru: vendor telephony** → tambah scope integrasi, procurement, biaya (mungkin alasan DA/MRG kena dampak).

---

## 📚 Referensi KB (Bluelink)

- `services/core/chat-service/meta-spec/README.md` — BirdTalk orchestration, API contracts, config
- `services/core/chat-engine/meta-spec/README.md` — Tinode backbone (voice/video call)
- `services/core/chat-dashboard/meta-spec/README.md` — dashboard (stub)
- PIC core: Ivanov Adie Ananda — ivanov.adie@bluebirdgroup.com

---

## ✅ Action Items (isi saat/sesudah meeting)

- [ ] Konfirmasi peta ownership 5 tim (producer/consumer event)
- [ ] Klarifikasi definisi "tamu" TFM
- [ ] Tentukan garpu call: A (Tinode VoIP) vs B (PSTN masking)
- [ ] Konfirmasi DA & titik singgung MRG
- [ ] Cek status VoIP feature flag ke core/Ivanov
