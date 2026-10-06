---
date: '2026-07-10'
tags:
  - reliability
  - kafka
  - patroni
  - mybb-portal
type: meeting-analysis
---

# Meeting Notes — Jumat, 10 Juli 2026

**Sumber:** Outlook Calendar (Chrome MCP)

## Ringkasan
5 meeting terjadwal (di luar yang Canceled). Ada 1 bentrok jadwal pukul 14:00–15:00.

## Jadwal

| Waktu | Meeting | Lokasi/Organizer | Catatan |
|---|---|---|---|
| 10:15–11:30 | Architecture Board (recurring) | Hybrid, Ruang Bali Lt 4 / Dyah Ayu Pravitaningsih | Bahas arsitektur sistem & issue teknis mingguan |
| 14:00–15:00 | **Reliability Improvement Alignment (DBA/DevOps MRG/Product)** | Ruang Meeting 5 Lt 2 Gedung Baru / **Lukmanul Hakim (organizer)** | Follow-up directive VP IT |
| 14:00–15:00 | Update Release Plan Rent Revamp | Ruang CIO / Junesly Milano | ⚠️ Bentrok jadwal dengan meeting di atas |
| 15:00–16:00 | Improvement Data Privacy di WebApp MyBB Portal | Ruang Meeting 2 Lt 5 Gedung Baru / Aldian Aby Dharmala | Ref: ticket ITRFD-469 |
| 16:00–17:00 | Update Arsitek (recurring) | Meja VP / Dyah Ayu Pravitaningsih | Attendee termasuk Happy Kushendraji |

## Detail: Reliability Improvement Alignment (Meeting Utama, Anda organizer)

**Attendees:** Aldian Aby Dharmala, Alfian Maulana Malik, Isfan Azhabil, Imam Aji Nugroho, Febry Rizky Wardani, Zainur Ridho

**Tujuan Meeting:**
1. Reliability Kafka — evaluasi kecukupan outbox pattern & circuit breaker saat ini untuk menangani downtime Kafka sesaat, serta kebutuhan Kafka cluster HA untuk mengurangi blast radius kegagalan broker.
2. Menyepakati jadwal patch database yang butuh downtime — koordinasi dengan tim DBA untuk eksekusi via planned switchover (Patroni HA), target RTO turun dari kondisi manual failover saat ini (~15–30 menit) menuju <5 menit.
3. Menyelaraskan prioritas tech debt dari kedua isu di atas dengan roadmap tim Product.

**Konteks:** Follow-up langsung dari directive VP IT (9 Juli 2026) — reliability jadi prioritas Q2, unknown error harus zero, tech debt urgent diselesaikan lebih dulu. Meeting ini menyasar 2 item dari tech-debt.md: Kafka Multi-Zone dan reliability database (Patroni HA switchover).

**Pertanyaan / Follow-up untuk dibahas:**
- Apakah reliability improvement (Kafka multi-AZ & Patroni switchover) bisa dilakukan **tanpa downtime**? Jika tidak bisa, **berapa lama downtime yang dibutuhkan**?
- Bagaimana kelanjutan **POC pemindahan Kafka tanpa downtime** yang pernah dilakukan sebelumnya? Perlu status update dari POC tersebut sebelum menentukan pendekatan final.

## Detail: Improvement Data Privacy di WebApp MyBB Portal
- Ticket: ITRFD-469
- Attendees: Aldian Aby Dharmala, Isfan Azhabil, Alfian Maulana Malik, Lukmanul Hakim, Fionna Benita, +2 lainnya

## Catatan / Flag
- ⚠️ **Bentrok jadwal 14:00–15:00**: "Reliability Improvement Alignment" (Anda organizer) vs "Update Release Plan Rent Revamp" (organizer: Junesly Milano, Ruang CIO). Perlu diputuskan prioritas kehadiran atau delegasi.
- Meeting Canceled hari ini (tidak dimasukkan tabel): [CR-Happy Weekend] Uncommon Daily Standup, [CR-Happy Weeekend] BackEnd Daily StandUp, [CR-Happy Weekend] Backlog Grooming, [MyBB] BE Weekly Code Review, Tech Grooming X Round Table.

## Related
- [[2026-07-09-meeting-vp-reliability]]


**Hasil Meeting:**
- Disepakati: butuh **downtime dengan maintenance mode active di mobile app** untuk eksekusi reliability improvement ini (Kafka switching & Patroni switchover).
- Akan dilakukan **simulasi di env staging** terlebih dahulu untuk mengukur durasi eksekusi mekanisme sebelum eksekusi di production.

**Action Items:**
- [ ] **Oman** — Validasi apakah waktu downtime bisa dipangkas ke **5 menit**.
- [ ] **Alfian (TL MRG)** — List semua service yang menggunakan Kafka, untuk keperluan switching config.
- [ ] **Adam (DevOps)** — Sediakan environment staging, support Alfian dalam pekerjaannya.
- [ ] **Mas Aby (Product)** — Tentukan kapan waktu eksekusi (jadwal downtime).
- [ ] **Lukmanul** — Buat **MOP (Metode Operasi Prosedur)** untuk task ini.
- [ ] Lakukan simulasi di env staging, ukur durasi eksekusi mekanisme.
