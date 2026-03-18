---
tags:
  - project
  - trading
  - automation
status: development-ready
created: '2026-01-20'
updated: '2026-01-21'
---
# 🤖 Multi-Account Trading Robot

> Personal Project - Indonesia Stocks
> Mode: Always-On, UI-based automation

---

## System Overview

```mermaid
flowchart TB
    subgraph INPUT["📥 INPUT"]
        SERVER["Owner Server<br/>(Tasks)"]
    end
    
    subgraph ROBOT["🤖 TRADING ROBOT"]
        ENGINE["Robot Engine"]
        
        subgraph BROWSERS["Chromium Profiles"]
            B1["ACC_01"]
            B2["ACC_02"]
            BN["ACC_N"]
        end
        
        STATE[("Local State<br/>Store")]
    end
    
    subgraph BROKERS["🏦 BROKER WEB UI"]
        BR1["Broker 1"]
        BR2["Broker 2"]
        BRN["Broker N"]
    end
    
    subgraph OUTPUT["📤 OUTPUT"]
        EVENTS["Events<br/>(TP_HIT, CL_HIT)"]
    end
    
    SERVER -->|"Pull Tasks"| ENGINE
    ENGINE --> B1 & B2 & BN
    B1 <--> BR1
    B2 <--> BR2
    BN <--> BRN
    ENGINE <--> STATE
    ENGINE -->|"Report"| EVENTS
```

---

## 📚 Documentation

### Core Design
- [[00-README|Overview]]
- [[01-Architecture|Architecture]]
- [[02-Security-Model|Security Model]]

### Task & State Management
- [[03-Task-Format|Task JSON Format]]
- [[04-State-Machine|State Machine Design]]
- [[05-Timeout-Retry|Timeout & Retry]]
- [[06-Partial-Fill-TPCL|Partial Fill & TP/CL Detection]]

### Operations
- [[07-Heartbeat-Monitoring|Heartbeat & Monitoring]]
- [[08-Concurrency-Model|Concurrency Model]]
- [[09-Logging-Audit|Logging & Audit Trail]]
- [[10-Alerting|Alerting Mechanism]]
- [[11-Market-Hours|Market Hours & Special Cases]]
- [[12-UI-Detection|UI Element Detection]]
- [[13-Session-Management|Session Management]]

### Risk & Planning
- [[14-Known-Risks|Known Risks & Future Improvements]]

### 🚀 Development Phase (NEW)
- [[15-Agent-Team-Structure|Agent Team Structure]]
- [[16-Dashboard-Requirements|Dashboard Requirements]]
- [[17-API-Contract|API Contract]]
- [[18-Testing-Strategy|Testing Strategy]]

### 🏦 Broker Specifications
- [[19-Stockbit-Broker-Flow|Stockbit Broker Flow]] - Standard Engine

---

## ✅ Design Complete

| # | Topic | Status |
|---|-------|--------|
| 1 | Architecture | ✅ Single-Account per Container |
| 2 | Timeout & Retry | ✅ Done |
| 3 | Partial Fill & TP/CL | ✅ Done |
| 4 | Cancel Order Flow | ✅ Covered |
| 5 | Retry & Failure Policy | ✅ Covered |
| 6 | Concurrency Model | ✅ Container-level (Simplified) |
| 7 | Logging & Audit Trail | ✅ Done |
| 8 | Alerting Mechanism | ✅ Done |
| 9 | Market Hours & Special Cases | ✅ Done |
| 10 | UI Element Detection | ✅ Done |
| 11 | Session Management | ✅ Done |
| 12 | Agent Team Structure | ✅ Done |
| 13 | Dashboard Requirements | ✅ Done |
| 14 | API Contract | ✅ Done |
| 15 | Testing Strategy | ✅ Done |
| 16 | Stockbit Broker Flow | ✅ Standard Engine |

---

## 📅 Timeline

| Date | Activity |
|------|----------|
| 2026-01-20 | Initial design discussion |
| 2026-01-20 | Migrated to Obsidian |
| 2026-01-20 | Added Mermaid diagrams |
| 2026-01-20 | Concurrency Model finalized (Multi-account) |
| 2026-01-20 | Logging & Audit Trail finalized |
| 2026-01-20 | Alerting Mechanism finalized |
| 2026-01-20 | Market Hours & Special Cases finalized |
| 2026-01-20 | UI Element Detection finalized |
| 2026-01-20 | Session Management finalized |
| 2026-01-20 | 🎉 All design topics completed! |
| 2026-01-20 | Added Known Risks & Future Improvements |
| 2026-01-21 | Agent Team Structure defined |
| 2026-01-21 | Dashboard Requirements defined |
| 2026-01-21 | API Contract defined |
| 2026-01-21 | Testing Strategy defined |
| 2026-01-21 | 🚀 Ready for Development Phase! |
| 2026-02-11 | **Major Revision: Single-Account per Container** |
| 2026-02-11 | Architecture simplified to container-level concurrency |
| 2026-02-11 | Stockbit Broker Flow documented as standard engine |
| 2026-02-11 | ✅ Architecture finalized for implementation |
