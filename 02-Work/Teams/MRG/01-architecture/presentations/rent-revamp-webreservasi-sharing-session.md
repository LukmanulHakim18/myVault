---
title: Rent Revamp & Web Reservasi — Sharing Session
type: presentation
status: ready
team: MRG
date: '2026-07-08'
owner: Lukmanul Hakim
tags:
  - mrg
  - presentation
  - sharing-session
  - rent-revamp
  - web-reservasi
  - architecture
  - payment
related:
  - rent-revamp-push-ho/README
  - reserved-budget-ecv/README
  - 2025-01-27 - ECV for GB Rent Revamp Discussion
---
# Rent Revamp & Web Reservasi — Sharing Session

Deck presentasi (HTML, self-contained) untuk sharing session arsitektur MRG.

- **File deck**: [[rent-revamp-webreservasi-sharing-session.html]]
- **Artifact (online)**: https://claude.ai/code/artifact/8e0a407a-2635-4f45-8160-ea03a48e7a69

## Navigasi deck
- `←` / `→` atau `Space` — pindah slide
- `F` — fullscreen · tombol **◐ Tema** — light/dark · dots — loncat slide · swipe di layar sentuh

## Struktur (12 slide)
1. Title
2. Agenda
3. Overview — dua platform, satu backend MRG
4. Produk & Use Case — Web = Goldenbird (ATF/hourly/daily); Rent = BB & SB hourly, GB hourly+daily
5. Perbandingan — Web vs Rent Revamp (gateway, produk, mode order, payment, push HO)
6. Two-Level ID — Booking → Order, prinsip *no ID waste*
7. Context Diagram — 4 zone + 6 key flow (rebuild dari `web-reservas.drawio.svg`)
8. Order Flow — 5 langkah end-to-end
9. Reserved Budget ECV — pre-order charge (deployed)
10. Push HO — ECV vs Non-ECV; UPG push untuk SB & BB, sisanya standard MRG
11. Lessons + Roadmap (open items target v6.24)
12. Closing / Q&A

## Fakta kunci pembeda (dikonfirmasi 2026-07-08)
- **Gateway**: Web via `ocelot` · Rent Revamp via MRG API Gateway (di-hit mobile MyBB)
- **User & login**: Web **tanpa login** — semua booking dipetakan ke 1 user teknis `BB00000001` (agar kompatibel skema DB) · Rent Revamp **wajib login** (user MyBB asli)
- **Produk**: Web = Goldenbird saja (ATF, hourly, daily) · Rent = BB & SB (hourly), GB (hourly, daily)
- **Mode**: Immediate hanya SB & BB · schedule tanpa limit tanggal, **daily max 6 hari**
- **Payment**: Web = Nicepay (payment link) · Rent → GB: cc + cash (wallet ke depan); SB & BB: semua kecuali `cc_cp` & `ecv`
- **Push HO**: UPG untuk SB & BB, sisanya standard MRG (`is_push_ho`)
- **Tabel order**: sama untuk keduanya, dikelola di `orderorchestrator`

## Sumber materi
- [[rent-revamp-push-ho/README|Push HO Rent Revamp — Payment Flow Design]]
- [[reserved-budget-ecv/README|Reserved Budget ECV — Pre-Order Flow Design]]
- [[2025-01-27 - ECV for GB Rent Revamp Discussion|ECV Meeting Notes]]
- ID Generator — Web Reservasi & Rent Revamp implementation
- Context diagram: `web-reservas.drawio.svg` (page `fase-1` & `documented`)

---
**Owner**: Lukmanul Hakim, MRG Team · **Last Updated**: 2026-07-08
