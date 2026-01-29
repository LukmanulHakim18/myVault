---
title: 'Meeting Prep: Server Busy Feature Kickoff'
date: '2025-01-27'
attendees:
  - MRG Team
  - BBD Team
status: preparation
tags:
  - meeting
  - mrg
  - bbd
  - server-busy
---
# Meeting Prep: Server Busy Feature Kickoff

**Objective**: Align MRG & BBD team untuk implementasi Server Busy feature  
**Related RFC**: [[RFC-Server-Busy-Dispatcher-Threshold]]  
**Supporting Analysis**: [[RFC-Server-Busy-Architecture-Analysis]]

---

## 1. Data yang Perlu Dikumpulkan SEBELUM Meeting

### 1.1 Dari Tim BBD (Legacy System)

| Data                                                                             | Purpose                    | PIC |
| -------------------------------------------------------------------------------- | -------------------------- | --- |
| **Waiting order tracking** - Apakah BBD sudah track waiting orders per area?     | Assess existing capability |     |
| **Area definition** - Bagaimana BBD define "area" untuk dispatch?                | Align area concept         |     |
| **Current waiting count** - Berapa rata-rata & peak waiting orders per area?     | Set initial threshold      |     |
| **Existing pre-order check** - Apakah sudah ada validation sebelum order dibuat? | Find integration point     |     |
| **Database/Redis access** - Storage apa yang dipakai BBD untuk realtime data?    | Technical decision         |     |

### 1.2 Dari Tim MRG

| Data                                                                      | Purpose                     | PIC  |
| ------------------------------------------------------------------------- | --------------------------- | ---- |
| **Pre-order flow** - Di step mana sebaiknya check dilakukan?              | Determine integration point |      |
| **Current BBD integration** - Endpoint apa yang dipanggil untuk dispatch? | Understand current contract |      |
| **Error handling** - Bagaimana MRG handle error dari BBD saat ini?        | Assess impact               |      |
| **Channel breakdown** - Traffic distribution per channel (MyBB, Web, etc) | Rollout planning            |      |
| **Loyalty data integration** - Bagaimana MRG dapat data loyalty?          | Loyalty bypass feature      |      |

### 1.3 Dari Tim Marketing (via MRG)

| Data                                                                                   | Purpose                  | PIC |
| -------------------------------------------------------------------------------------- | ------------------------ | --- |
| **Loyalty level definition** - Explorer, Traveler, Voyager, Elite                      | Define bypass levels     |     |
| **User distribution per level** - Berapa % user di tiap level?                         | Estimate bypass impact   |     |
| **API/Data source** - Dari mana MRG bisa dapat loyalty level user?                     | Technical integration    |     |
| **Business rules** - Level mana yang boleh bypass? (Voyager & Elite?)                  | Configuration            |     |

### 1.4 Historical Data (dari Monitoring/Analytics)

| Data                                                                 | Purpose                    | Source     |
| -------------------------------------------------------------------- | -------------------------- | ---------- |
| **Peak waiting orders** - Max waiting orders per area historically   | Set initial threshold      | Grafana/DB |
| **Order distribution by area** - Top 10 busiest areas                | Prioritize area setup      | Analytics  |
| **Dispatch success rate by area** - Which areas have lowest success? | Identify problem areas     | Analytics  |
| **Average dispatch duration** - Per area, per service type           | Validate timeout config    | Monitoring |
| **Cancel rate breakdown** - By reason (user, timeout, etc)           | Baseline metrics           | Analytics  |
| **Traffic pattern** - Peak hours, peak days                          | Dynamic threshold planning | Analytics  |

---

## 2. Agenda Meeting

### Part 1: Problem Alignment (10 min)
- [ ] Review incident 12 Jan 2025 (77k users traffic spike)
- [ ] Confirm pain points dari masing-masing tim
- [ ] Align on goals: reduce waiting order buildup, improve customer experience

### Part 2: Current State Review (15 min)
- [ ] BBD: Explain current dispatch flow
- [ ] BBD: Confirm existing timeout & retry behavior
- [ ] MRG: Explain current pre-order flow
- [ ] Identify gaps

### Part 3: Proposed Solution Walkthrough (15 min)
- [ ] Present RFC high-level design
- [ ] Mechanism A: Server Busy Check (pre-order)
- [ ] Mechanism B: Dispatch Retry (existing, need alignment)
- [ ] Q&A

### Part 4: Technical Decisions (20 min)
- [ ] Decide: Area definition approach
- [ ] Decide: Where to store waiting count (Redis location)
- [ ] Decide: Who owns threshold configuration
- [ ] Decide: API contract between MRG ↔ BBD

### Part 5: Action Items & Timeline (10 min)
- [ ] Assign ownership per component
- [ ] Agree on timeline per phase
- [ ] Schedule follow-up meetings

---

## 3. Key Questions yang HARUS Terjawab

### 3.1 Untuk Tim BBD

```
1. WAITING ORDER TRACKING
   - Apakah BBD sudah track jumlah order yang sedang waiting dispatch?
   - Jika ya, di mana disimpan? (Redis/DB/Memory)
   - Apakah di-track per area atau global?

2. AREA CONCEPT
   - Bagaimana BBD define "area" untuk nearby driver search?
   - Apakah sudah ada area_id atau zone concept?
   - Apakah pakai geohash, polygon, atau radius?

3. THRESHOLD CHECK FEASIBILITY
   - Apakah bisa add endpoint baru untuk check waiting count?
   - Atau lebih baik MRG langsung akses Redis yang sama?
   - Latency concern untuk pre-order check?

4. DATA AVAILABILITY
   - Data apa yang bisa di-expose untuk threshold decision?
   - Apakah ada historical data waiting count per area?

5. INTEGRATION EFFORT
   - Seberapa besar effort untuk implement di legacy system?
   - Apakah ada constraint atau blocker?
   - Timeline estimate?
```

### 3.2 Untuk Tim MRG

```
1. PRE-ORDER FLOW
   - Step-by-step pre-order flow saat ini?
   - Di mana integration point yang tepat untuk threshold check?
   - Apakah perlu call BBD atau bisa check sendiri?

2. ERROR HANDLING
   - Bagaimana handle SERVER_BUSY error ke MyBB?
   - Apakah perlu new error code?
   - Retry behavior di MyBB side?

3. MULTI-SERVICE SUPPORT
   - Apakah Rent & Golden Bird flow sama dengan Ride?
   - Differences yang perlu dihandle?
   - Order Orchestrator readiness?
```

### 3.3 Decisions yang Harus Diputuskan di Meeting

| # | Decision | Options | Impact |
|---|----------|---------|--------|
| D1 | Area definition approach | Geohash vs Predefined zones vs Hybrid | Complexity, flexibility |
| D2 | Threshold check location | New BBD endpoint vs MRG direct Redis access | Latency, ownership |
| D3 | Waiting count tracking | BBD track & expose vs Shared Redis | Data consistency |
| D4 | Threshold config storage | BBD config vs Central config service | Maintainability |
| D5 | Initial threshold value | Fixed vs Per-area based on historical data | Risk, accuracy |
| D6 | Rollout scope Phase 1 | RIDE only vs All services | Risk, timeline |
| D7 | Rollout strategy | Shadow mode vs Pilot areas vs Full rollout | Risk, speed |
| D8 | Loyalty bypass config | Which levels can bypass, max bypass % | Customer experience, capacity |
| D9 | Rejection approach | Hard vs Soft vs Zone-based vs Dynamic | Customer experience, churn risk |

---

## 4. Pre-Meeting Checklist

### Tim BBD
- [ ] Prepare current dispatch flow diagram
- [ ] Gather metrics: avg dispatch time, success rate per area
- [ ] Identify legacy system constraints
- [ ] Estimate effort untuk new endpoint

### Tim MRG  
- [ ] Prepare current pre-order flow diagram
- [ ] List all integration points dengan BBD
- [ ] Identify Order Orchestrator readiness untuk Rent/GB
- [ ] Prepare traffic data per channel

### Meeting Organizer (Lukmanul)
- [ ] Share RFC document sebelum meeting
- [ ] Prepare screen untuk diagram walkthrough

---

## 5. Expected Outcomes

Setelah meeting, harus punya clarity untuk:

1. **Technical approach** - Bagaimana exactly flow akan bekerja
2. **Ownership matrix** - Siapa develop apa
3. **API contract draft** - Request/response format
4. **Timeline commitment** - Target date per phase
5. **Blockers identified** - Apa yang bisa jadi hambatan
6. **Follow-up actions** - Next steps yang jelas

---

## 6. Parking Lot (untuk meeting berikutnya)

Topik yang mungkin perlu meeting terpisah:
- [ ] Detailed API contract review
- [ ] Load testing strategy
- [ ] Monitoring & alerting setup
- [ ] Rollout & feature flag strategy
- [ ] MyBB UI/UX untuk server busy message

---

## Notes During Meeting

*(Isi saat meeting berlangsung)*

### Attendees
| Nama | Tim | Role |
|------|-----|------|
| | | |
| | | |
| | | |
| | | |

---

## Decision Matrix

### D1: Area Definition Approach

| Option               | Pros                   | Cons                       |
| -------------------- | ---------------------- | -------------------------- |
| **Geohash**          | Automatic, scalable    | Kurang precise di boundary |
| **Predefined Zones** | Kontrol penuh, precise | Manual setup, maintenance  |
| **Hybrid**           | Flexible               | Lebih kompleks             |

**Keputusan**: _______________________________________________

**Alasan**: _______________________________________________

**PIC**: __________________ **Target Date**: __________________

---

### D2: Threshold Check Location

| Option                                 | Pros                                  | Cons                                    |
| -------------------------------------- | ------------------------------------- | --------------------------------------- |
| **New BBD Endpoint**                   | BBD owns data, single source of truth | Latency tambahan, BBD effort            |
| **MRG Direct Redis**                   | Faster, less dependency               | Data sync concern, siapa maintain Redis |
| **Shared Redis (BBD write, MRG read)** | Balance ownership                     | Perlu agree on schema                   |

**Keputusan**: _______________________________________________

**Alasan**: _______________________________________________

**PIC**: __________________ **Target Date**: __________________

---

### D3: Waiting Count Tracking

| Option | Pros | Cons |
|--------|------|------|
| **BBD Track & Expose** | BBD sudah punya data order | Perlu new tracking logic |
| **Shared Redis** | Real-time, accessible by both | Siapa maintain consistency |
| **Existing BBD Data** | No new development | Mungkin tidak per-area |

**Keputusan**: _______________________________________________

**Alasan**: _______________________________________________

**PIC**: __________________ **Target Date**: __________________

---

### D4: Threshold Configuration Storage

| Option | Pros | Cons |
|--------|------|------|
| **BBD Config** | BBD owns dispatch logic | Coupling dengan legacy |
| **Central Config Service** | Reusable, single source | New dependency |
| **Database Table** | Simple, queryable | Need admin UI |

**Keputusan**: _______________________________________________

**Alasan**: _______________________________________________

**PIC**: __________________ **Target Date**: __________________

---

### D5: Initial Threshold Value

| Option                             | Pros           | Cons                       |
| ---------------------------------- | -------------- | -------------------------- |
| **Fixed (semua area sama)**        | Simple rollout | Tidak optimal per area     |
| **Per-area (based on historical)** | Accurate       | Perlu data analysis dulu   |
| **Conservative high, lalu tuning** | Safe rollout   | Mungkin tidak efektif awal |

**Keputusan**: _______________________________________________

**Initial Value**: _____________ orders per area

**Alasan**: _______________________________________________

**PIC**: __________________ **Target Date**: __________________

---

### D6: Phase 1 Rollout Scope

| Option              | Pros                        | Cons                         |
| ------------------- | --------------------------- | ---------------------------- |
| **RIDE only**       | Lower risk, faster delivery | Partial solution             |
| **RIDE + DELIVERY** | Cover high volume services  | More coordination            |
| **All services**    | Complete solution           | Higher risk, longer timeline |

**Keputusan**: _______________________________________________

**Services included**: _______________________________________________

**Alasan**: _______________________________________________

**PIC**: __________________ **Target Date**: __________________

---

### D7: Rollout Strategy

| Option                             | Pros               | Cons             |
| ---------------------------------- | ------------------ | ---------------- |
| **Shadow mode first**              | Safe, collect data | Delay impact     |
| **Pilot 2-3 areas**                | Real validation    | Limited coverage |
| **Full rollout with feature flag** | Fast               | Higher risk      |

**Keputusan**: _______________________________________________

**Pilot Areas (jika applicable)**: _______________________________________________

**Alasan**: _______________________________________________

**PIC**: __________________ **Target Date**: __________________

---

### D8: Loyalty Bypass Configuration

| Option                         | Pros                           | Cons                  |
| ------------------------------ | ------------------------------ | --------------------- |
| **ELITE only**                 | Most exclusive, minimal load   | Very limited coverage |
| **VOYAGER + ELITE**            | Balance exclusivity & coverage | Moderate extra orders |
| **TRAVELER + VOYAGER + ELITE** | Wider coverage                 | More orders bypass    |
| **Configurable per area**      | Flexible                       | Complex management    |

**Keputusan**: _______________________________________________

**Bypass Levels**: ☐ EXPLORER  ☐ TRAVELER  ☐ VOYAGER  ☐ ELITE

**Max Bypass Percentage**: _____________% extra capacity

**Alasan**: _______________________________________________

**PIC**: __________________ **Target Date**: __________________

---

### D9: Rejection Approach (lihat [[RFC-Server-Busy-Architecture-Analysis]])

| Option                            | Pros                         | Cons                          |
| --------------------------------- | ---------------------------- | ----------------------------- |
| **Hard Rejection**                | Simple, clear cut            | Customer churn ke competitor  |
| **Soft Rejection + Transparency** | Customer informed, better UX | More complex UI               |
| **Zone-Based (GREEN/YELLOW/RED)** | Gradual, flexible            | More config needed            |
| **Dynamic Ratio (supply-demand)** | Most fair, adaptive          | Need driver availability data |

**Keputusan**: _______________________________________________

**Jika Zone-Based, threshold per zone:**
- GREEN max: _________ orders
- YELLOW max: _________ orders  
- RED max: _________ orders (priority only)
- BLOCKED: > RED max

**Alasan**: _______________________________________________

**PIC**: __________________ **Target Date**: __________________

---

## Decision Summary Table

| # | Decision | Choice | PIC | Target Date |
|---|----------|--------|-----|-------------|
| D1 | Area Definition | | | |
| D2 | Check Location | | | |
| D3 | Waiting Count Tracking | | | |
| D4 | Config Storage | | | |
| D5 | Initial Threshold | | | |
| D6 | Phase 1 Scope | | | |
| D7 | Rollout Strategy | | | |
| D8 | Loyalty Bypass Config | | | |
| D9 | Rejection Approach | | | |

---

## Action Items

| # | Action Item | Owner | Due Date | Status |
|---|-------------|-------|----------|--------|
| 1 | | | | ⬜ |
| 2 | | | | ⬜ |
| 3 | | | | ⬜ |
| 4 | | | | ⬜ |
| 5 | | | | ⬜ |
| 6 | | | | ⬜ |
| 7 | | | | ⬜ |
| 8 | | | | ⬜ |

---

## Open Issues / Blockers

| # | Issue | Impact | Owner | Resolution |
|---|-------|--------|-------|------------|
| 1 | | | | |
| 2 | | | | |
| 3 | | | | |

---

## Follow-up Meetings

| Topic | Attendees | Proposed Date |
|-------|-----------|---------------|
| API Contract Review | | |
| Technical Deep Dive | | |
| | | |

---

## Next Steps

1. _______________________________________________
2. _______________________________________________
3. _______________________________________________

