# HLD-design

High-Level Design (HLD) architecture diagrams and system design documentation.

## Architectures

| System | Diagram File | Key Concepts |
| --- | --- | --- |
| E-Commerce Microservices | [`ecommerce-hld-microservices.mmd`](./ecommerce-hld-microservices.mmd) | API Gateway, Saga Orchestrator, Outbox Pattern, Kafka Event Backbone, Polyglot Persistence, OpenTelemetry |

---

## E-Commerce Microservices HLD

Source diagram: [`ecommerce-hld-microservices.mmd`](./ecommerce-hld-microservices.mmd)

### Overview

This design models an event-driven e-commerce microservices architecture separating synchronous user-facing flows from asynchronous event-driven workflows.

- **Solid arrows (`-->`)**: Synchronous calls (user waiting)
- **Dotted arrows (`-.->`)**: Asynchronous communication via Kafka (and database replication / telemetry)

### Architecture Components

#### 1. Edge & Security
- **Traffic Path**: `Client (Web / Mobile / Admin)` → `DNS` → `CDN CloudFront` → `WAF` → `Load Balancer` → `API GATEWAY`
- **Static Assets**: `CDN CloudFront` serves images from `S3 Images`
- **API Gateway**: Handles JWT validation, rate limiting, and routing

#### 2. Microservices & Data Stores
- **User Service**: Redis (`Session / Token`), PostgreSQL (`PRIMARY` + `REPLICA`)
- **Product Service**: Redis (`Catalog Cache`), MongoDB (`PRIMARY` + `REPLICA`)
- **Search Service**: Redis (`Query Cache`), Elasticsearch (+ `Elastic Replica`)
- **Recommendation Service**: Redis (`Top-N Cache`), Cassandra (`Node` + `Replica`)
- **Cart Service**: Redis (`Active Carts`), MongoDB (`PRIMARY` + `REPLICA`)
- **Order Service (Saga Orchestrator)**: PostgreSQL (`PRIMARY` with `orders + outbox table`, + `REPLICA`)
- **Payment Service**: Idempotency keys, PostgreSQL (`PRIMARY` with `payments + outbox table`, + `REPLICA`), external `Payment Provider (Razorpay / Stripe)`
- **Inventory Service**: Redis (`Stock Cache`), PostgreSQL (`PRIMARY` + `REPLICA`)
- **Shipping Service**: PostgreSQL (`PRIMARY` + `REPLICA`), external `Courier / Logistics API`
- **Notification Service**: Cassandra (`PRIMARY` + `REPLICA`), external `Email / SMS`

#### 3. Event Backbone (Kafka)
- **Cluster**: Replicated x3 Kafka cluster partitioned by entity ID with a consumer group per service
- **Topics**: `orders`, `payments`, `catalog`, `inventory`, `shipments`, `notifications`
- **Resilience**: Failed messages route to `Retry Topics (exponential backoff)` and either retry back to Kafka or move to the `Dead Letter Queue (DLQ)` once retries are exhausted
- **Producers (Outbox Relay)**:
  - `Product Service` → `catalog`
  - `Order Service` → `orders`
  - `Payment Service` → `payments`
  - `Inventory Service` → `inventory`
  - `Shipping Service` → `shipments`
- **Consumers**:
  - `Search Service` ← `catalog`
  - `Recommendation Service` ← `orders`, `catalog`
  - `Inventory Service` ← `orders`
  - `Shipping Service` ← `payments`
  - `Order Service` ← `payments`, `shipments`
  - `Notification Service` ← `orders`, `payments`, `shipments`

#### 4. Synchronous Flows
- `API GATEWAY` → `User Service`, `Product Service`, `Search Service`, `Recommendation Service`, `Cart Service`, `Order Service`
- `Cart Service` → `Order Service`
- `Order Service` → `Payment Service`

#### 5. Observability (OpenTelemetry)
- **Centralized Logging**: ELK / Loki (logs from Kafka)
- **Metrics & Alerting**: Prometheus + Grafana (metrics from Order Service) → PagerDuty / Slack
- **Distributed Tracing**: Jaeger (traces from API Gateway)

### Diagram

```mermaid
flowchart LR

    %% ===================== LEGEND =====================
    subgraph LEGEND["Legend"]
        direction TB
        L1["Solid line = Sync (user waiting)"]
        L2["Dotted line = Async (via Kafka)"]
    end

    %% ===================== EDGE =====================
    subgraph EDGE["Edge and Security"]
        direction TB
        CLIENT["Client<br/>Web / Mobile / Admin"]
        DNS["DNS"]
        CDN["CDN CloudFront"]
        S3[("S3 Images")]
        WAF["WAF"]
        LB["Load Balancer"]
        CLIENT --> DNS --> CDN --> WAF --> LB
        CDN --> S3
    end

    GW["API GATEWAY<br/>JWT Validation<br/>Rate Limiting<br/>Routing"]
    LB --> GW

    %% ===================== USER SERVICE =====================
    subgraph USER_SVC["User Service"]
        direction TB
        US["User Service"]
        US_R[("Redis<br/>Session / Token")]
        US_P[("PostgreSQL<br/>PRIMARY")]
        US_PR[("PostgreSQL<br/>REPLICA")]
        US --> US_R
        US --> US_P
        US_P -. replication .-> US_PR
    end

    %% ===================== PRODUCT SERVICE =====================
    subgraph PRODUCT_SVC["Product Service"]
        direction TB
        PS["Product Service"]
        PS_R[("Redis<br/>Catalog Cache")]
        PS_M[("MongoDB<br/>PRIMARY")]
        PS_MR[("MongoDB<br/>REPLICA")]
        PS --> PS_R
        PS --> PS_M
        PS_M -. replication .-> PS_MR
    end

    %% ===================== SEARCH SERVICE =====================
    subgraph SEARCH_SVC["Search Service"]
        direction TB
        SS["Search Service"]
        SS_R[("Redis<br/>Query Cache")]
        SS_E[("Elasticsearch")]
        SS_ER[("Elastic Replica")]
        SS --> SS_R
        SS --> SS_E
        SS_E -. replication .-> SS_ER
    end

    %% ===================== RECOMMENDATION SERVICE =====================
    subgraph RECO_SVC["Recommendation Service"]
        direction TB
        RS["Recommendation Service"]
        RS_R[("Redis<br/>Top-N Cache")]
        RS_C[("Cassandra<br/>Node")]
        RS_CR[("Cassandra<br/>Replica")]
        RS --> RS_R
        RS --> RS_C
        RS_C -. replication .-> RS_CR
    end

    %% ===================== CART SERVICE =====================
    subgraph CART_SVC["Cart Service"]
        direction TB
        CS["Cart Service"]
        CS_R[("Redis<br/>Active Carts")]
        CS_M[("MongoDB<br/>PRIMARY")]
        CS_MR[("MongoDB<br/>REPLICA")]
        CS --> CS_R
        CS --> CS_M
        CS_M -. replication .-> CS_MR
    end

    %% ===================== ORDER SERVICE =====================
    subgraph ORDER_SVC["Order Service - Saga Orchestrator"]
        direction TB
        OS["Order Service"]
        OS_P[("PostgreSQL PRIMARY<br/>orders + outbox table")]
        OS_PR[("PostgreSQL<br/>REPLICA")]
        OS --> OS_P
        OS_P -. replication .-> OS_PR
    end

    %% ===================== PAYMENT SERVICE =====================
    subgraph PAY_SVC["Payment Service"]
        direction TB
        PAY["Payment Service<br/>idempotency keys"]
        PAY_P[("PostgreSQL PRIMARY<br/>payments + outbox table")]
        PAY_PR[("PostgreSQL<br/>REPLICA")]
        PAY --> PAY_P
        PAY_P -. replication .-> PAY_PR
    end
    PROVIDER["Payment Provider<br/>Razorpay / Stripe<br/>(external)"]
    PAY --> PROVIDER

    %% ===================== KAFKA =====================
    subgraph KAFKA_BOX["Event Backbone"]
        direction TB
        KAFKA[("KAFKA<br/>Replicated x3<br/>Topics: orders, payments, catalog,<br/>inventory, shipments, notifications<br/>Consumer group per service<br/>Partition by entity id")]
        RETRY["Retry Topics<br/>exponential backoff"]
        DLQ[("Dead Letter Queue<br/>Failed Messages")]
        KAFKA -. failed .-> RETRY
        RETRY -. retries exhausted .-> DLQ
        RETRY -. retry .-> KAFKA
    end

    %% ===================== INVENTORY SERVICE =====================
    subgraph INV_SVC["Inventory Service"]
        direction TB
        INV["Inventory Service"]
        INV_R[("Redis<br/>Stock Cache")]
        INV_P[("PostgreSQL<br/>PRIMARY")]
        INV_PR[("PostgreSQL<br/>REPLICA")]
        INV --> INV_R
        INV --> INV_P
        INV_P -. replication .-> INV_PR
    end

    %% ===================== SHIPPING SERVICE =====================
    subgraph SHIP_SVC["Shipping Service"]
        direction TB
        SHIP["Shipping Service"]
        SHIP_P[("PostgreSQL<br/>PRIMARY")]
        SHIP_PR[("PostgreSQL<br/>REPLICA")]
        SHIP --> SHIP_P
        SHIP_P -. replication .-> SHIP_PR
    end
    COURIER["Courier / Logistics API<br/>(external)"]
    SHIP --> COURIER

    %% ===================== NOTIFICATION SERVICE =====================
    subgraph NOTIF_SVC["Notification Service"]
        direction TB
        NOTIF["Notification Service"]
        NOTIF_C[("Cassandra<br/>PRIMARY")]
        NOTIF_CR[("Cassandra<br/>REPLICA")]
        NOTIF --> NOTIF_C
        NOTIF_C -. replication .-> NOTIF_CR
    end
    MSG["Email / SMS<br/>(external)"]
    NOTIF --> MSG

    %% ===================== OBSERVABILITY =====================
    subgraph OBS["Observability - OpenTelemetry"]
        direction TB
        LOG["Centralized Logging<br/>ELK / Loki"]
        MET["Metrics<br/>Prometheus + Grafana"]
        TRC["Distributed Tracing<br/>Jaeger"]
        ALERT["Alerting<br/>PagerDuty / Slack"]
        MET --> ALERT
    end

    %% ===================== SYNC FLOW (user waiting) =====================
    GW --> US
    GW --> PS
    GW --> SS
    GW --> RS
    GW --> CS
    GW --> OS
    OS --> PAY
    CS --> OS

    %% ===================== ASYNC: PRODUCERS (outbox relay) =====================
    PS -. "catalog" .-> KAFKA
    OS -. "orders" .-> KAFKA
    PAY -. "payments" .-> KAFKA
    INV -. "inventory" .-> KAFKA
    SHIP -. "shipments" .-> KAFKA

    %% ===================== ASYNC: CONSUMERS =====================
    KAFKA -. "catalog" .-> SS
    KAFKA -. "orders, catalog" .-> RS
    KAFKA -. "orders" .-> INV
    KAFKA -. "payments" .-> SHIP
    KAFKA -. "payments, shipments" .-> OS
    KAFKA -. "orders, payments, shipments" .-> NOTIF

    %% ===================== TELEMETRY =====================
    GW -. traces .-> TRC
    KAFKA -. logs .-> LOG
    OS -. metrics .-> MET

    %% ===================== STYLES =====================
    classDef svc fill:#1e1b4b,stroke:#818cf8,color:#e0e7ff,stroke-width:2px
    classDef pg fill:#1e3a8a,stroke:#60a5fa,color:#dbeafe
    classDef mongo fill:#14532d,stroke:#4ade80,color:#dcfce7
    classDef redis fill:#7f1d1d,stroke:#f87171,color:#fee2e2
    classDef cass fill:#164e63,stroke:#22d3ee,color:#cffafe
    classDef elastic fill:#365314,stroke:#a3e635,color:#ecfccb
    classDef ext fill:#422006,stroke:#fb923c,color:#ffedd5,stroke-dasharray:5 5
    classDef kafka fill:#4c0519,stroke:#fb7185,color:#ffe4e6,stroke-width:3px
    classDef dlq fill:#0c4a6e,stroke:#38bdf8,color:#e0f2fe
    classDef edge fill:#27272a,stroke:#a1a1aa,color:#f4f4f5
    classDef gw fill:#312e81,stroke:#a5b4fc,color:#eef2ff,stroke-width:3px

    class US,PS,SS,RS,CS,OS,PAY,INV,SHIP,NOTIF svc
    class US_P,US_PR,OS_P,OS_PR,PAY_P,PAY_PR,INV_P,INV_PR,SHIP_P,SHIP_PR pg
    class PS_M,PS_MR,CS_M,CS_MR mongo
    class US_R,PS_R,SS_R,RS_R,CS_R,INV_R redis
    class RS_C,RS_CR,NOTIF_C,NOTIF_CR cass
    class SS_E,SS_ER elastic
    class PROVIDER,COURIER,MSG ext
    class KAFKA kafka
    class DLQ,RETRY dlq
    class CLIENT,DNS,CDN,S3,WAF,LB edge
    class GW gw
```
