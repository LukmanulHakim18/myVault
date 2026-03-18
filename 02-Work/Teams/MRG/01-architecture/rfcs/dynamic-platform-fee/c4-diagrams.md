---
title: Dynamic Platform Fee - C4 Diagrams
tags:
  - architecture
  - mrg
  - dynamic-platform-fee
  - c4
  - diagram
created: '2026-02-18'
status: final
team: MRG
updated: '2026-03-02'
---
# Dynamic Platform Fee — C4 Diagrams

**Updated:** 2026-03-02  
**Status:** Final

---

## 1. System Context Diagram

```mermaid
C4Context
    title System Context - Dynamic Platform Fee

    Person(user, "MyBB User", "End user yang menggunakan aplikasi")
    
    System(mybb, "MyBB App", "Mobile application")
    System(dpf, "Dynamic Platform Fee", "Fee calculation engine")
    System(si, "Service Info", "Fleet & pricing service")
    
    System_Ext(oo, "Order Orchestrator", "Order management")
    System_Ext(li, "Loyalty Inquiry", "Marketing loyalty service")
    System_Ext(us, "User Service", "User profile management")
    
    Rel(user, mybb, "Uses")
    Rel(mybb, si, "Request fleet list")
    Rel(si, dpf, "Calculate platform fee")
    Rel(dpf, oo, "Get order counter")
    Rel(dpf, li, "Get loyalty tier")
    Rel(us, dpf, "User-Info header (is_foreign)")
```

---

## 2. Container Diagram

```mermaid
C4Container
    title Container Diagram - Dynamic Platform Fee

    Container(api, "DPF API", "Go", "REST API untuk fee calculation")
    ContainerDb(db, "PostgreSQL", "Database", "Rules, config, service config")
    ContainerDb(cache, "Redis", "Cache", "Fee calculation cache")
    
    Container_Ext(oo, "Order Orchestrator", "Go", "Order counter API")
    Container_Ext(li, "Loyalty Inquiry", "Go", "Loyalty tier API")
    
    Rel(api, db, "Read rules & config")
    Rel(api, cache, "Cache lookup/store")
    Rel(api, oo, "GET /v1/order/counter/{bbid}")
    Rel(api, li, "GET /api/v1/customer/{bbid}")
```

---

## 3. Sequence Diagram — Fee Calculation Flow

```mermaid
sequenceDiagram
    participant C as Client (MyBB App)
    participant SI as service-info
    participant DPF as dynamic-platform-fee
    participant OO as order-orchestrator
    participant LI as loyalty-inquiry
    participant Redis as Redis
    participant DB as PostgreSQL

    C->>SI: Request Fleet List
    Note over SI: Header: User-Info.is_foreign
    
    par Parallel Execution
        SI->>SI: Get base fare & fleet data
    and
        SI->>DPF: POST /v1/platform-fee/calculate
        
        par DPF Data Fetching
            DPF->>OO: GET /v1/order/counter/{bbid}
            OO-->>DPF: total_completed_order
        and
            DPF->>LI: GET /api/v1/customer/{bbid}
            LI-->>DPF: current_tier
        end
        
        DPF->>DPF: Check new user (order <= 1)
        
        alt New User
            DPF-->>SI: final_fee = 0
        else Not New User
            DPF->>Redis: GET dpf:fee:{combination_key}
            
            alt Cache HIT
                Redis-->>DPF: cached fee result
            else Cache MISS
                DPF->>DB: Load rules & config
                DB-->>DPF: rules, global_config, service_config
                DPF->>DPF: Evaluate modifiers
                DPF->>DPF: Calculate raw fee
                DPF->>DPF: Round up to nearest 1000
                DPF->>DPF: Apply cap & floor
                DPF->>Redis: SET dpf:fee:{key} TTL 1h
            end
            
            DPF-->>SI: Fee response + breakdown
        end
    end
    
    SI->>SI: Merge base fare + platform fee
    SI-->>C: Fleet list with final platform fee
```

---

## 4. Sequence Diagram — Foreign Flag Detection

```mermaid
sequenceDiagram
    participant U as User
    participant App as MyBB App
    participant UPG as UPG
    participant WF as Webhook Forwarder
    participant US as User Service

    alt Add Payment Card
        U->>App: Add credit card
        App->>UPG: Register card
        UPG->>UPG: Process with provider
        UPG->>WF: Callback with card metadata
        Note over WF: card.country_code = "US"
        WF->>WF: Check country_code != "ID"
        WF->>US: PUT /v1/user/{bbid}/foreign-flag
        US->>US: Set is_foreign = true
        US-->>WF: 200 OK
    else Change Phone Number
        U->>App: Change phone to +1xxx
        App->>US: Update phone number
        US->>US: Check phone prefix != 62
        US->>US: Set is_foreign = true
    end
```

---

## 5. Entity Relationship Diagram

```mermaid
erDiagram
    dpf_rules {
        BIGSERIAL id PK
        VARCHAR group_name
        VARCHAR parameter
        VARCHAR condition_key
        JSONB condition_value
        INT adjustment_amount
        INT priority
        BOOLEAN is_active
        VARCHAR_ARRAY service_types
        TEXT description
        TIMESTAMP created_at
        TIMESTAMP updated_at
    }

    dpf_global_config {
        VARCHAR config_key PK
        INT int_value
        BOOLEAN bool_value
        VARCHAR string_value
        VARCHAR value_type
        TEXT description
        TIMESTAMP updated_at
    }

    dpf_service_config {
        BIGSERIAL id PK
        VARCHAR service_type
        INT base_platform_fee
        INT cap_platform_fee
        BOOLEAN is_active
        TEXT description
        TIMESTAMP updated_at
    }

    dpf_config_history {
        BIGSERIAL id PK
        VARCHAR table_name
        VARCHAR record_key
        TEXT old_value
        TEXT new_value
        VARCHAR changed_by
        TIMESTAMP changed_at
    }

    dpf_rules ||--o{ dpf_config_history : "audit"
    dpf_global_config ||--o{ dpf_config_history : "audit"
    dpf_service_config ||--o{ dpf_config_history : "audit"
```

---

## 6. Fee Calculation Flow

```mermaid
flowchart TD
    A[Start] --> B{New User?<br/>order <= 1}
    B -->|Yes| C[Return fee = 0]
    B -->|No| D[Get base_platform_fee]
    
    D --> E{Foreign User?<br/>is_foreign && order >= 6}
    E -->|Yes| F[+ foreign_surcharge]
    E -->|No| G[Skip]
    
    F --> H{Corporate?<br/>payment = ECV/voucher/cc_cp}
    G --> H
    
    H -->|Yes| I[+ corporate_surcharge]
    H -->|No| J[Skip]
    
    I --> K{Foreign Card?<br/>country != ID}
    J --> K
    
    K -->|Yes| L[+ bin_country_surcharge]
    K -->|No| M[Skip]
    
    L --> N{AMEX?}
    M --> N
    
    N -->|Yes| O[+ amex_surcharge]
    N -->|No| P[Skip]
    
    O --> Q{Has Loyalty Tier?}
    P --> Q
    
    Q -->|Yes| R[- loyalty_discount]
    Q -->|No| S[Skip]
    
    R --> T[Calculate raw_fee]
    S --> T
    
    T --> U[Round up to nearest 1000]
    U --> V{rounded > cap?}
    V -->|Yes| W[final = cap]
    V -->|No| X{rounded < base?}
    X -->|Yes| Y[final = base]
    X -->|No| Z[final = rounded]
    
    W --> END[Return final_fee]
    Y --> END
    Z --> END
    C --> END
```

---

## 7. Caching Strategy

```mermaid
flowchart LR
    subgraph Cache Key
        A[service_type] --> K[dpf:fee:]
        B[user_category] --> K
        C[loyalty_tier] --> K
        D[payment_type] --> K
        E[urgency_type] --> K
    end
    
    K --> F["dpf:fee:RIDE_BLUE_ARGO:foreign:ELITE:cc:immediate"]
    
    subgraph TTL
        F --> G[1 hour]
    end
    
    subgraph Invalidation
        H[Rule change] --> I[SCAN dpf:fee:*]
        I --> J[DEL matched keys]
    end
```

---

## 8. Phase 2 Integration (Future)

```mermaid
sequenceDiagram
    participant DPF as dynamic-platform-fee
    participant PS as Polygon Service
    participant DS as Demand Service

    Note over DPF,DS: Phase 2 - External Integration
    
    par Pickup Zone Check
        DPF->>PS: POST /v1/polygon/check
        Note right of DPF: {"lat": -6.2, "lng": 106.8}
        PS-->>DPF: {"zone": "APSH"}
    and
        DPF->>DS: GET /v1/demand/{area}
        DS-->>DPF: {"category": "Extreme"}
    end
    
    DPF->>DPF: Apply zone surcharge
    DPF->>DPF: Apply surge surcharge
```

---

## 9. Related Documents

- [[RFC-001-dynamic-platform-fee-service]]
- [[RFC-002-order-orchestrator-counter]]
- [[RFC-003-user-service-foreign-flag]]
