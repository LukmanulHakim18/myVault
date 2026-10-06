---
date: '2026-06-30'
location: Ruang Bunaken Lt 4
meeting: Update Arsitek
organizer: Dyah Ayu Pravitaningsih
status: Confirmed
time: '16:00-17:00'
type: meeting-notes
---

# MoM — Update Arsitek (30 Juni 2026)

**Waktu:** 16:00–17:00
**Lokasi:** Ruang Bunaken Lt 4
**Organizer:** Dyah Ayu Pravitaningsih

## Konteks

Pembahasan dari VP terkait rencana mengurangi **regression bugs** (deploy sesuatu tapi menyenggol hal lain).

## Framework & Standarisasi

- Shift-left testing
- Static analysis → migrasi GOPATH ke Go Modules
- Dev self testing
- ~~Early QA Improvement~~ *(dicoret/dibatalkan)*
- Dependency mapping
- Feature flags
- Automating 80% of Core Business Path Regression Tests by Q4

## Code Health Standards

- **Mandatory Unit Testing:** Ambang batas minimum coverage 70% untuk setiap Pull Request baru.
- **Risk-Based Code Review:** Reviewer wajib memvalidasi dampak perubahan pada modul terkait (side-effect checklist).
- **Continuous Refactoring:** Alokasi 10% kapasitas sprint untuk pembersihan technical debt yang berpotensi regresi.

## Monitoring

- *(belum dirinci — to be filled)*

## Action Items

- [ ] 

## Related

- [[2026-W27-weekly-meetings-report]]
