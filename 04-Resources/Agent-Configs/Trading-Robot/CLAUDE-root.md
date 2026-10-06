---
tags:
  - claude-code
  - agents
  - configuration
  - root
created: '2026-01-21'
---
# CLAUDE.md (Root Project Context)

> Copy file ini ke root folder project: `trading-robot-project/CLAUDE.md`

---

```markdown
# Trading Robot Project

## 🎯 Project Overview

Multi-account trading robot untuk Indonesia stocks market.
- **Mode:** Always-On, UI-based automation via Playwright
- **Target:** 10+ broker accounts (Stockbit, IPOT, Ajaib, dll)
- **Stack:** Go backend, Next.js dashboard, PostgreSQL

## 🏗️ Architecture

```
┌─────────────────────────────────────────────────────────────┐
│                      OWNER (You)                            │
│                          │                                  │
│                    ┌─────▼─────┐                           │
│                    │ Dashboard │ ◄── Next.js               │
│                    └─────┬─────┘                           │
│                          │ REST + WebSocket                │
│                    ┌─────▼─────┐                           │
│                    │  Owner    │ ◄── Go + Fiber            │
│                    │  Server   │                           │
│                    └─────┬─────┘                           │
│                          │                                 │
│              ┌───────────┼───────────┐                    │
│              │           │           │                    │
│         ┌────▼───┐  ┌────▼───┐  ┌────▼───┐               │
│         │Robot 1 │  │Robot 2 │  │Robot N │ ◄── Go +      │
│         │(ACC_01)│  │(ACC_02)│  │(ACC_N) │    Playwright │
│         └────┬───┘  └────┬───┘  └────┬───┘               │
│              │           │           │                    │
│         ┌────▼───┐  ┌────▼───┐  ┌────▼───┐               │
│         │Stockbit│  │  IPOT  │  │ Ajaib  │ ◄── Broker   │
│         │  Web   │  │  Web   │  │  Web   │    Web UI    │
│         └────────┘  └────────┘  └────────┘               │
└─────────────────────────────────────────────────────────────┘
```

## 📁 Repository Structure

```
trading-robot-project/
├── CLAUDE.md              # This file (shared context)
├── .agents/               # Agent configurations
│   ├── pm-agent.md
│   ├── be-agent.md
│   ├── fe-agent.md
│   └── qa-agent.md
├── docs/                  # Design documentation
│   ├── 01-Architecture.md
│   ├── 08-Concurrency-Model.md
│   ├── 12-UI-Detection.md
│   ├── 16-Dashboard-Requirements.md
│   ├── 17-API-Contract.md
│   ├── 18-Testing-Strategy.md
│   └── sprints/
│       └── sprint-1.md
├── robot-engine/          # Go - Playwright automation
├── owner-server/          # Go - REST API + WebSocket
├── dashboard/             # Next.js - Monitoring UI
└── e2e-tests/             # Playwright Test
```

## 👥 Agent Team

| Agent | Role | Domains |
|-------|------|---------|
| **PM Agent** | Orchestrator | `/docs`, sprints, coordination |
| **BE Agent** | Backend Engineer | `/robot-engine`, `/owner-server` |
| **FE Agent** | Frontend Engineer | `/dashboard` |
| **QA Agent** | QA Engineer | `/e2e-tests`, CI/CD |

## 📋 Key Documentation

### For All Agents
- `docs/17-API-Contract.md` - API specs (BE produces, FE consumes)
- `docs/sprints/sprint-X.md` - Current sprint tasks

### For BE Agent
- `docs/01-Architecture.md` - System design
- `docs/04-State-Machine.md` - Order states
- `docs/08-Concurrency-Model.md` - Worker design
- `docs/12-UI-Detection.md` - Selector strategy

### For FE Agent
- `docs/16-Dashboard-Requirements.md` - UI specs & wireframes

### For QA Agent
- `docs/18-Testing-Strategy.md` - Test approach

## 🔄 Git Workflow

```
main
  │
  ├── develop
  │     │
  │     ├── feat/db-schema      (BE)
  │     ├── feat/api-auth       (BE)
  │     ├── feat/dashboard      (FE)
  │     └── feat/ci-pipeline    (QA)
  │
  └── release/v1.0
```

### Branch Naming
- `feat/xxx` - New feature
- `fix/xxx` - Bug fix
- `refactor/xxx` - Code refactor
- `test/xxx` - Test additions

### PR Flow
1. Agent creates branch & works
2. Agent creates PR
3. PM reviews & merges to develop

## 🎯 Current Sprint

Check `docs/sprints/sprint-1.md` for current tasks.

## ⚠️ Important Rules

1. **API Contract is Source of Truth** - BE implements, FE consumes
2. **Tests Required** - No PR tanpa tests
3. **Documentation** - Update docs jika ada perubahan design
4. **Communication** - Via PR comments & sprint docs
```

---

## Usage

1. Copy content di atas (dalam code block)
2. Paste ke file `CLAUDE.md` di root project
3. Adjust sesuai kebutuhan
