# 📇 VAULT MASTER INDEX

**Version:** 2.0  
**Last Updated:** 2026-01-28  
**Purpose:** Single source of truth untuk struktur vault - untuk Claude DAN Lukmanul Hakim

---

# PART 1: FOR CLAUDE 🤖

> **CRITICAL:** Read this section FIRST before any file operations!

```yaml
# ============================================================================
# OBSIDIAN VAULT STRUCTURE - MANDATORY READING FOR CLAUDE
# ============================================================================
# CRITICAL RULE: Read this file FIRST before creating/moving/searching files
# This prevents misplaced files and ensures proper organization
# ============================================================================

vault_name: "myVault"
owner: "Lukmanul Hakim"
role: "Architecture Engineer - Bluebird Group"
teams:
  - "MRG (Meta Reservation Gateway)"
  - "UPG (Universal Payment Gateway)"

# ============================================================================
# CRITICAL RULES FOR CLAUDE
# ============================================================================

critical_rules:
  1: "ALWAYS read this file BEFORE creating/moving files"
  2: "DEPLOYMENTS go in 02-Work/Deployments/ ONLY"
  3: "Work content goes in 02-Work/ and appropriate subfolders"
  4: "Check folder purpose before creating files"
  5: "Follow existing folder structure - do NOT create new top-level folders"
  6: "When in doubt, ask Lukmanul which folder to use"
  7: "Keep year correct (2026, not 2025!)"
  8: "Update all three levels for deployments: detail → monthly → summary"

# ============================================================================
# DEPLOYMENT WORKFLOW (MOST IMPORTANT)
# ============================================================================

deployment_workflow:
  location: "02-Work/Deployments/"
  
  before_deployment:
    1: "Read 02-Work/Deployments/README.md for workflow"
    2: "Check deployment-overtime-summary.md for existing records"
    3: "Update 2026-[MM]-deployment-log.md with new deployment info"
  
  after_deployment:
    1: "Update 2026-[MM]-deployment-log.md with actual hours"
    2: "Update monthly-deployment-overtime-2026.md monthly totals"
    3: "Update deployment-overtime-summary.md master dashboard"
  
  files_to_update:
    - "2026-01-deployment-log.md (Detail level)"
    - "monthly-deployment-overtime-2026.md (Monthly level)"
    - "deployment-overtime-summary.md (Master level)"

# ============================================================================
# QUICK REFERENCE - WHERE TO PUT WHAT
# ============================================================================

content_placement:
  architecture_docs: "02-Work/Architecture/"
  rfcs: "02-Work/Teams/{MRG|UPG}/01-architecture/rfcs/"
  adrs: "02-Work/Teams/{MRG|UPG}/01-architecture/adrs/"
  design_docs: "02-Work/Teams/{MRG|UPG}/01-architecture/design-docs/"
  code_reviews: "02-Work/Code-Reviews/"
  deployments: "02-Work/Deployments/"
  overtime_tracking: "02-Work/Deployments/"
  meetings: "02-Work/Meetings/"
  team_mrg: "02-Work/Teams/MRG/"
  team_upg: "02-Work/Teams/UPG/"
  technical_specs: "02-Work/Technical-Specs/"
  go_programming: "02-Work/Go-Programming/"
  domain_knowledge: "02-Work/Transportation-Domain/"
  personal_projects: "01-Projects/"
  learning: "03-Learning/"
  personal: "07-Personal/"
  archive: "08-Archive/"

# ============================================================================
# COMMON MISTAKES TO AVOID
# ============================================================================

common_mistakes:
  - mistake: "Creating deployment files in root or wrong folder"
    correct: "Always use 02-Work/Deployments/"
  
  - mistake: "Using year 2025 instead of 2026"
    correct: "Current year is 2026"
  
  - mistake: "Not checking existing structure before creating files"
    correct: "Read this index first, then check folder structure"
  
  - mistake: "Creating new top-level folders without permission"
    correct: "Use existing folder structure"
  
  - mistake: "Forgetting to update all three deployment levels"
    correct: "Update detail log → monthly → summary"
  
  - mistake: "Putting Bluebird work in 01-Projects/"
    correct: "Bluebird work goes to 02-Work/, personal projects to 01-Projects/"

# ============================================================================
# FOLDER STRUCTURE OVERVIEW
# ============================================================================

folders:
  - path: "00-Inbox"
    purpose: "Temporary storage for quick captures"
    claude_action: "AVOID creating files here unless explicitly requested"
  
  - path: "01-Projects"
    purpose: "PERSONAL projects ONLY (non-Bluebird)"
    claude_action: "Use for side projects, freelance, business ventures"
    note: "Bluebird work goes to 02-Work/, NOT here!"
  
  - path: "02-Work"
    purpose: "⭐ MAIN work folder - All Bluebird work content"
    claude_action: "PRIMARY folder for work-related content"
    subfolders:
      - "Architecture/ - Architecture docs, RFCs, design decisions"
      - "Architecture-Office/ - Org-wide architecture standards"
      - "Code-Reviews/ - MR reviews, code quality"
      - "Deployments/ - ⭐ DEPLOYMENT OVERTIME TRACKING"
      - "Go-Programming/ - Go patterns, best practices"
      - "Meetings/ - Meeting notes, action items"
      - "Shared/ - Cross-team documentation"
      - "Teams/MRG/ - Meta Reservation Gateway docs"
      - "Teams/UPG/ - Universal Payment Gateway docs"
      - "Technical-Specs/ - Feature specs, API specs"
      - "Transportation-Domain/ - Domain knowledge"
  
  - path: "03-Learning"
    purpose: "Learning materials, courses, tutorials"
    claude_action: "Use for learning and skill development"
  
  - path: "04-Resources"
    purpose: "Templates, references, tools"
    claude_action: "Use for general resources and templates"
  
  - path: "05-Team"
    purpose: "Team management and leadership"
    claude_action: "Use for team processes, 1-on-1s"
  
  - path: "06-Career"
    purpose: "Career planning and development"
    claude_action: "Use for career goals, reviews"
  
  - path: "07-Personal"
    purpose: "Personal notes (non-work)"
    claude_action: "Use for personal/non-work content only"
  
  - path: "08-Archive"
    purpose: "Archived and historical content"
    claude_action: "AVOID creating new files - archive only"
```

---

# PART 2: FOR HUMANS 👨‍💻

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

### 📂 Deployments (OVERTIME TRACKING)

**Location:** `02-Work/Deployments/`

**Files:**
- `deployment-overtime-summary.md` - Master Dashboard
- `monthly-deployment-overtime-2026.md` - 2026 Monthly Summary
- `2026-01-deployment-log.md` - Current Month Detail
- `README.md` - Folder Guide & Workflow

**Workflow:**
1. **Before Deploy:** Update deployment-log dengan info deployment
2. **After Deploy:** Update actual hours di 3 level
   - Detail log (monthly)
   - Monthly summary (yearly)
   - Master dashboard (all-time)

---

### 🏗️ Teams Structure

#### MRG (Meta Reservation Gateway)
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

**Existing Services:**
- `booking-service` - Order/booking management
- `id-generator` - ID generation system ✅ (documented)
- `notification-service` - Multi-channel notifications
- `reservation-service` - Reservation handling
- `vehicle-service` - Fleet management

**Recent Design Docs:**
- ✅ Optimalisasi Placement - Drop-off & Pick-up Point Accuracy (Completed)

---

#### UPG (Universal Payment Gateway)
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

### 🏛️ Architecture Office
Cross-team architectural concerns.

| Tipe Dokumen | Path |
|--------------|------|
| **Architecture Reviews** | `02-Work/Architecture-Office/architecture-reviews/` |
| **Incidents & Postmortems** | `02-Work/Architecture-Office/incidents/` |
| **Initiatives** | `02-Work/Architecture-Office/initiatives/` |
| **Technical Debt** | `02-Work/Architecture-Office/technical-debt/` |
| **Metrics** | `02-Work/Architecture-Office/metrics/` |
| **Roadmap** | `02-Work/Architecture-Office/roadmap/` |

**Recent Major Initiatives:**
- ✅ GCP to Huawei Cloud Migration (Jul-Aug 2025)

---

### 🔧 Shared Resources
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

### Deployment Overtime
→ `02-Work/Deployments/` - Update 3 levels (detail → monthly → summary)

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

## 🎯 How to Use This Index

### For Claude:
1. ✅ Read PART 1 (YAML section) BEFORE any file operation
2. ✅ Follow critical rules
3. ✅ Check content_placement for correct location
4. ✅ Avoid common mistakes

### For Lukmanul Hakim:
1. ✅ Reference PART 2 for detailed structure
2. ✅ Use naming conventions consistently
3. ✅ Follow templates for new documents
4. ✅ Keep counters updated (RFC, ADR)

---

## 🔄 Maintenance

**Update this file when:**
- New top-level folder added
- Major restructuring
- New subfolder in 02-Work/
- New workflow or process
- Critical rules change
- RFC/ADR counters increment

---

## 📞 Related Files

- [[README]] - Vault setup guide & getting started
- [[dashboard]] - Personal operational dashboard
- [[02-Work/Deployments/README]] - Deployment workflow guide

---

**Version:** 2.0  
**Created:** 2026-01-28  
**Last Updated:** 2026-01-28  
**Maintained By:** Claude (Assistant)  
**Reviewed By:** Lukmanul Hakim  
**Update Frequency:** As needed when structure changes

---

*This is the single source of truth for vault structure. All other index files are deprecated.*
