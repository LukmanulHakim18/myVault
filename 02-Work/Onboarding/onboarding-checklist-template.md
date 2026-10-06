---
type: template
category: onboarding
created: '2026-01-28'
purpose: Standardized onboarding checklist untuk new engineers
---
# Onboarding Checklist: [NAMA ENGINEER]

**Start Date:** [TGL]  
**Role:** [Backend Engineer / DevOps / QA / etc]  
**Team:** [MRG / UPG / Other]  
**Assigned To:** Lukmanul Hakim  

---

## Phase 1: Account & Access Setup

### Email & OS Setup
- [ ] **Email Creation**
  - Internal: Request ke HR (jika tidak ada)
  - External: Setup Outlook + register sebagai OS user
  - Email: `[first.last]@bluebird.id` atau `[email]@outlook.com`

- [ ] **LDAP Registration**
  - File: `C:\Users\lukmanul.hakim\Downloads\Request LDAP.eml`
  - Copy template, update email, save & execute
  - Status: Pending / Completed
  - Date Completed: [TGL]

### Git Account
- [ ] **Service Ticket Creation**
  - Platform: https://bluebirdgroup.atlassian.net/servicedesk/customer/portal/5
  - Ticket Category: Level 1 Support
  - Helpdesk Agent: Bayu Yanuar Riski Miharjo (21164)
  - Label: `git-account`
  - Description Template:
    ```
    Dear tim support.
    Berikut email user OS Bluebird yang sudah teradaftar LDAP, 
    mohon agar di process pengajuan pembuatan account git:
    Email: [email]
    Terima kasih
    ```
  - Ticket ID: [ID]
  - Status: Pending / Approved
  - Date Completed: [TGL]

- [ ] **Git Access Verification**
  - Clone test repo: `git clone https://github.com/bluebird/test-repo.git`
  - Command: `git config --global user.email "[email]"`
  - Status: ✅ Working

### ClickUp Account
- [ ] **Service Ticket Creation**
  - Ticket Category: Level 1 Support
  - Support Engineer: Aris Fadillah
  - Label: `clickup-account`
  - Reference: Git ticket ID: [ID]
  - Status: Pending / Approved
  - Date Completed: [TGL]

- [ ] **ClickUp Access Verification**
  - Login to: https://bluebirdgroup.atlassian.net
  - Workspace access: Bluebird Engineering
  - Team assignment: [Team]
  - Status: ✅ Working

---

## Phase 2: Developer Environment Setup

### Local Environment
- [ ] **Clone repos**
  - MRG: `git clone https://github.com/bluebird/meta-reservation-gateway.git`
  - UPG: `git clone https://github.com/bluebird/universal-payment-gateway.git`
  - Other: [specify]

- [ ] **Setup Go environment**
  - Go version: 1.x (check team requirement)
  - Command: `go version`
  - Status: ✅ Verified

- [ ] **Setup Docker**
  - Docker Desktop / WSL2
  - Verify: `docker --version`
  - Status: ✅ Running

- [ ] **Database access**
  - PostgreSQL local: docker compose
  - Kafka local: docker compose
  - Redis local: docker compose
  - Status: ✅ Working

### IDE Setup
- [ ] **VS Code / GoLand configuration**
  - Extensions: Go, Docker, Git, etc
  - Linter: golangci-lint
  - Formatter: gofmt / goimports
  - Status: ✅ Configured

---

## Phase 3: Team & Knowledge Onboarding

### Team Introductions
- [ ] **First day meeting**
  - Who: Team lead + 2-3 engineers
  - Topics: Team structure, current projects, tech stack
  - Date: [TGL]
  - Status: Scheduled / Completed

- [ ] **1-on-1 with tech lead**
  - Focus: Team processes, expectations, learning path
  - Duration: 30 min
  - Date: [TGL]
  - Status: Scheduled / Completed

### Knowledge Transfer
- [ ] **Architecture overview**
  - RFC reading: [relevant RFCs for their area]
  - ADR reading: [relevant ADRs]
  - Design docs: [relevant design docs]
  - Status: ✅ Completed / In Progress

- [ ] **Codebase walkthrough**
  - Project structure: [service name]
  - Key modules: [modules]
  - Dependencies: [key libraries]
  - Date: [TGL]
  - Status: Scheduled / Completed

- [ ] **Process documentation**
  - Code review standards: ✅
  - Deployment procedure: ✅
  - Incident response: ✅
  - Meeting schedules: ✅
  - Status: Reviewed

### Slack & Communication
- [ ] **Join channels**
  - #bluebird-engineering
  - #team-[MRG/UPG]
  - #deployment
  - #incidents
  - #[other relevant]
  - Status: ✅ Joined

---

## Phase 4: First Task

### Onboarding Task
- [ ] **Assigned issue/task**
  - Task: [small, non-critical task]
  - ClickUp ID: [ID]
  - Expected outcome: [description]
  - Timeline: [days]

- [ ] **Code review + merge**
  - Reviewer: [tech lead / peer]
  - PR link: [PR URL]
  - Status: Pending / In Review / Merged
  - Date Completed: [TGL]

---

## Phase 5: Completion

- [ ] **Access verification** — all systems working
- [ ] **Knowledge check** — can navigate codebase
- [ ] **First task completed** — merged PR / working feature
- [ ] **Next sprint assignment** — ready for real work

**Status:** 🔄 In Progress / ✅ Completed  
**Date Completed:** [TGL]  
**Notes:** [any blockers or special notes]

---

## Contacts & Resources

**Git/ClickUp Support:**
- Helpdesk: Bayu Yanuar Riski Miharjo (bayu@bluebird.id)
- Support Engineer: Aris Fadillah

**Tech Leads:**
- MRG: [name]
- UPG: [name]

**Documentation:**
- Confluence: [link]
- Obsidian (architecture): [[architecture-index]]
- Code repo: https://github.com/bluebird

**Quick Links:**
- Service Desk: https://bluebirdgroup.atlassian.net/servicedesk/customer/portal/5
- ClickUp: https://bluebirdgroup.atlassian.net
