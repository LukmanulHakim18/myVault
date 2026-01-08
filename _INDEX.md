---
title: Obsidian Vault Index
type: index
updated: '2025-01-05'
---
# 📚 Obsidian Vault Index

> **Untuk Claude**: File ini adalah referensi utama struktur vault. Baca file ini untuk menentukan lokasi penyimpanan dokumen baru.

**Last Updated**: 2025-01-05

---

## 🗂️ Vault Structure Overview

| Folder | Tujuan | Contoh Isi |
|--------|--------|------------|
| `00-Inbox/` | Quick capture, unsorted | Catatan cepat belum diproses |
| `01-Projects/` | **Personal projects** di luar Bluebird | Side projects, freelance, bisnis, content creation |
| `02-Work/` | **Dokumentasi kerja Bluebird** | MRG, UPG, Architecture Office |
| `03-Learning/` | Pembelajaran | Articles, Books, Courses, Tech Talks |
| `04-Resources/` | Templates & referensi | Templates, cheatsheets |
| `05-Team/` | Tim & kolaborasi | Meetings, Knowledge Sharing |
| `06-Career/` | Karir | Performance reviews, goals |
| `07-Personal/` | Personal | Finance, Health, Life Goals |
| `08-Archive/` | Arsip | Old & deprecated docs |

---

## 🚀 01-Projects (Personal Projects)

> **PENTING**: Folder ini HANYA untuk project di luar Bluebird!
> Dokumentasi kerja Bluebird masuk ke `02-Work/`

### Struktur
```
01-Projects/
├── Active/        # Project yang sedang dikerjakan
├── Completed/     # Project selesai
└── On-Hold/       # Project ditunda
```

### Kategori Project
- **Side Projects** - Coding/app personal
- **Content Creation** - Blog, YouTube, podcast
- **Business** - Usaha/bisnis personal
- **Open Source** - Kontribusi OSS
- **Freelance** - Project freelance

### Cara Menambah Project Baru
```
01-Projects/Active/{nama-project}/
├── README.md          # Overview, goals, status
├── docs/              # Dokumentasi
├── notes/             # Catatan development
└── resources/         # Assets, references
```

### Tags untuk Project
- `#project` - Semua project
- `#side-project` - Side project coding
- `#business` - Project bisnis
- `#content` - Content creation
- `#oss` - Open source
- `#active` / `#completed` / `#on-hold` - Status

---

## 🏢 02-Work (Bluebird Documentation)

### MRG (Meta Reservation Gateway)
Backend untuk MyBluebird App - ride-hailing, airport transfer, rental.

| Tipe Dokumen | Path | Keterangan |
|--------------|------|------------|
| **RFC** | `02-Work/Teams/MRG/01-architecture/rfcs/` | Request for Comments fitur baru |
| **ADR** | `02-Work/Teams/MRG/01-architecture/adrs/` | Architecture Decision Records |
| **Design Docs** | `02-Work/Teams/MRG/01-architecture/design-docs/` | Detailed design documents |
| **Diagrams** | `02-Work/Teams/MRG/01-architecture/diagrams/` | Architecture diagrams |
| **Service Docs** | `02-Work/Teams/MRG/02-services/{service-name}/` | Per-service documentation |
| **API Contracts** | `02-Work/Teams/MRG/03-apis/contracts/` | API specifications |
| **REST APIs** | `02-Work/Teams/MRG/03-apis/rest-apis/` | REST endpoint docs |
| **gRPC Services** | `02-Work/Teams/MRG/03-apis/grpc-services/` | gRPC service definitions |
| **Events** | `02-Work/Teams/MRG/03-apis/events/` | Event schemas & docs |
| **Runbooks** | `02-Work/Teams/MRG/05-runbooks/` | Operational guides |

**Existing Services**:
- `booking-service` - Order/booking management
- `id-generator` - ID generation system ✅ (documented)
- `notification-service` - Multi-channel notifications
- `reservation-service` - Reservation handling
- `vehicle-service` - Fleet management

**Recent Design Docs**:
- ✅ [[02-Work/Teams/MRG/01-architecture/design-docs/optimalisasi-placement/README|Optimalisasi Placement]] - Drop-off & Pick-up Point Accuracy (Completed)

---

### UPG (Universal Payment Gateway)
Backend payment untuk MyBluebird App.

| Tipe Dokumen | Path | Keterangan |
|--------------|------|------------|
| **RFC** | `02-Work/Teams/UPG/01-architecture/rfcs/` | Request for Comments |
| **ADR** | `02-Work/Teams/UPG/01-architecture/adrs/` | Architecture Decision Records |
| **Design Docs** | `02-Work/Teams/UPG/01-architecture/design-docs/` | Detailed design documents |
| **Diagrams** | `02-Work/Teams/UPG/01-architecture/diagrams/` | Architecture diagrams |
| **Service Docs** | `02-Work/Teams/UPG/02-services/{service-name}/` | Per-service documentation |
| **API Contracts** | `02-Work/Teams/UPG/03-apis/contracts/` | API specifications |
| **Runbooks** | `02-Work/Teams/UPG/05-runbooks/` | Operational guides |

---

### Architecture Office
Cross-team architectural concerns.

| Tipe Dokumen | Path |
|--------------|------|
| **Architecture Reviews** | `02-Work/Architecture-Office/architecture-reviews/` |
| **Incidents & Postmortems** | `02-Work/Architecture-Office/incidents/` |
| **Initiatives** | `02-Work/Architecture-Office/initiatives/` |
| **Technical Debt** | `02-Work/Architecture-Office/technical-debt/` |
| **Metrics** | `02-Work/Architecture-Office/metrics/` |
| **Roadmap** | `02-Work/Architecture-Office/roadmap/` |

**Recent Major Initiatives**:
- ✅ [[02-Work/Architecture-Office/initiatives/gcp-to-hwc-migration/INDEX|GCP to Huawei Cloud Migration]] (Jul-Aug 2025)
  - [[02-Work/Architecture-Office/incidents/gcp-hwc-migration/README|Incident Reports]]
  - [[02-Work/Architecture-Office/incidents/gcp-hwc-migration/RUNBOOK|Migration Runbook]]

---

### Shared Resources
Standards dan patterns untuk semua tim.

| Tipe Dokumen | Path |
|--------------|------|
| **Coding Standards** | `02-Work/Shared/standards/` |
| **Design Patterns** | `02-Work/Shared/patterns/` |
| **Integration Guides** | `02-Work/Shared/integration/` |
| **Glossary** | `02-Work/Shared/glossary/` |
| **Go Programming** | `02-Work/Go-Programming/` |

---

## 📝 Naming Conventions

### RFC Files
```
RFC-{NNNN}-{judul-singkat}.md
Contoh: RFC-0001-id-generation-system.md
```

### ADR Files
```
ADR-{NNNN}-{judul-singkat}.md
Contoh: ADR-0001-use-postgresql-for-booking.md
```

### Design Docs
```
{feature-name}/README.md
Contoh: optimalisasi-placement/README.md
```

### Service Documentation
```
{service-name}/
├── README.md              # Overview
├── 00-project/            # Project management
├── 01-architecture/       # Architecture docs
├── 02-implementation/     # Implementation details
├── 03-api/                # API documentation
├── 04-database/           # Database schemas
├── 04-testing/            # Test documentation
└── 05-operations/         # Ops & runbooks
```

---

## 📋 Templates

| Template | Path | Gunakan Untuk |
|----------|------|---------------|
| RFC | `04-Resources/Templates/rfc-template.md` | Fitur baru, perubahan arsitektur |
| ADR | `04-Resources/Templates/adr-template.md` | Keputusan arsitektur |
| Service Doc | `04-Resources/Templates/service-doc-template.md` | Dokumentasi service baru |
| Code Review | `04-Resources/Templates/code-review-template.md` | Review code |
| Meeting Notes | `04-Resources/Templates/meeting-notes-template.md` | Notulen meeting |

---

## 🔧 Quick Reference for Claude

### Menambah Feature/Design Doc di MRG
```
02-Work/Teams/MRG/01-architecture/design-docs/{feature-name}/
├── README.md
├── diagrams/
└── ... (supporting docs)
```
Tag dengan: `#mrg`, `#design-doc`, `#completed` (jika selesai)

### Menambah Feature/Design Doc di UPG
```
02-Work/Teams/UPG/01-architecture/design-docs/{feature-name}/
├── README.md
├── diagrams/
└── ... (supporting docs)
```
Tag dengan: `#upg`, `#design-doc`, `#completed` (jika selesai)

### Menambah Personal Project
```
01-Projects/Active/{project-name}/
├── README.md
└── ... (project files)
```
Tag dengan: `#project`, `#side-project` / `#business` / `#content`

### Dokumentasi Service Baru
→ `02-Work/Teams/{MRG|UPG}/02-services/{service-name}/` dengan struktur folder standard

### Postmortem Incident
→ `02-Work/Architecture-Office/incidents/postmortems/`

---

## 📊 Document Registry

### RFC Counter
- **MRG**: Next available → `RFC-0001`
- **UPG**: Next available → `RFC-0001`

### ADR Counter
- **MRG**: Next available → `ADR-0001`
- **UPG**: Next available → `ADR-0001`

> ⚠️ **Note**: Update counter setelah membuat dokumen baru

---

## 🏷️ Tag System

### Work Tags
- `#mrg` - Tim MRG
- `#upg` - Tim UPG
- `#architecture` - Architecture docs
- `#design-doc` - Design documents
- `#rfc` - Request for Comments
- `#adr` - Architecture Decision Records
- `#completed` - Dokumentasi selesai

### Project Tags
- `#project` - Personal project
- `#side-project` - Side project coding
- `#business` - Project bisnis
- `#content` - Content creation
- `#active` / `#on-hold` - Status project

### Personal Tags
- `#finance` - Keuangan
- `#career` - Karir
- `#learning` - Pembelajaran

---

*Index ini di-generate untuk membantu navigasi vault. Update manual jika ada perubahan struktur major.*
