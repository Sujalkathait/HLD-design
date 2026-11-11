# High Level Design · E-Commerce Platform

Microservices architecture for an e-commerce platform like Amazon or Flipkart.

---

## How It Works

A customer opens the app, browses products, adds items to cart, places an order, makes payment, and receives the product. 10 independent services handle each part, talking to each other through Kafka so no service blocks another.

> **Solid lines** = User is waiting for a response
>
> **Dotted lines** = Happens in background through Kafka

---

## Architecture Diagram

```mermaid
flowchart LR

    %% ===================== LEGEND =====================
    subgraph LEGEND["Legend"]
        direction TB
        L1["Solid Line = User is Waiting"]
        L2["Dotted Line = Happens in Background"]
    end

    %% ===================== CLIENT TO GATEWAY =====================
    CLIENT["Client<br/>Website · Mobile App"] --> DNS["DNS"] --> CDN["CDN<br/>CloudFront"] --> WAF["Firewall"] --> LB["Load Balancer"]
    CDN --> S3[("S3<br/>Images")]

    GW["API Gateway<br/>Token Check<br/>Rate Limiting<br/>Routing"]
    LB --> GW

    %% ===================== USER SERVICE =====================
    subgraph USER_BLOCK["User Service"]
        direction TB
        USER_S["User Service"]
        USER_REDIS[("Redis<br/>Cache")]
        USER_PG[("PostgreSQL<br/>Primary")]
        USER_PG_R[("PostgreSQL<br/>Replica")]
        USER_S --> USER_REDIS
        USER_S --> USER_PG
        USER_PG -. replication .-> USER_PG_R
    end

    %% ===================== PRODUCT SERVICE =====================
    subgraph PRODUCT_BLOCK["Product Service"]
        direction TB
        PRODUCT_S["Product Service"]
        PRODUCT_REDIS[("Redis<br/>Cache")]
        PRODUCT_MONGO[("MongoDB<br/>Primary")]
        PRODUCT_MONGO_R[("MongoDB<br/>Replica")]
        PRODUCT_S --> PRODUCT_REDIS
        PRODUCT_S --> PRODUCT_MONGO
        PRODUCT_MONGO -. replication .-> PRODUCT_MONGO_R
    end

    %% ===================== SEARCH SERVICE =====================
    subgraph SEARCH_BLOCK["Search Service"]
        direction TB
        SEARCH_S["Search Service"]
        SEARCH_REDIS[("Redis<br/>Cache")]
        SEARCH_ES[("Elasticsearch<br/>Primary")]
        SEARCH_ES_R[("Elasticsearch<br/>Replica")]
        SEARCH_S --> SEARCH_REDIS
        SEARCH_S --> SEARCH_ES
        SEARCH_ES -. replication .-> SEARCH_ES_R
    end

    %% ===================== RECOMMENDATION SERVICE =====================
    subgraph RECO_BLOCK["Recommendation Service"]
        direction TB
        RECO_S["Recommendation Service"]
        RECO_REDIS[("Redis<br/>Cache")]
        RECO_CASS[("Cassandra<br/>Node")]
        RECO_CASS_R[("Cassandra<br/>Replica")]
        RECO_S --> RECO_REDIS
        RECO_S --> RECO_CASS
        RECO_CASS -. replication .-> RECO_CASS_R
    end

    %% ===================== CART SERVICE =====================
    subgraph CART_BLOCK["Cart Service"]
        direction TB
        CART_S["Cart Service"]
        CART_REDIS[("Redis<br/>Cache")]
        CART_MONGO[("MongoDB<br/>Primary")]
        CART_MONGO_R[("MongoDB<br/>Replica")]
        CART_S --> CART_REDIS
        CART_S --> CART_MONGO
        CART_MONGO -. replication .-> CART_MONGO_R
    end

    %% ===================== ORDER SERVICE =====================
    subgraph ORDER_BLOCK["Order Service"]
        direction TB
        ORDER_S["Order Service"]
        ORDER_PG[("PostgreSQL<br/>Primary")]
        ORDER_PG_R[("PostgreSQL<br/>Replica")]
        ORDER_S --> ORDER_PG
        ORDER_PG -. replication .-> ORDER_PG_R
    end

    %% ===================== PAYMENT SERVICE =====================
    subgraph PAYMENT_BLOCK["Payment Service"]
        direction TB
        PAYMENT_S["Payment Service"]
        PAYMENT_PG[("PostgreSQL<br/>Primary")]
        PAYMENT_PG_R[("PostgreSQL<br/>Replica")]
        PAYMENT_S --> PAYMENT_PG
        PAYMENT_PG -. replication .-> PAYMENT_PG_R
    end

    %% ===================== KAFKA =====================
    subgraph KAFKA_BLOCK["Kafka · Message Queue"]
        direction TB
        KAFKA[("Kafka · 3 Servers<br/>Topics: orders · payments<br/>catalog · inventory<br/>shipments · notifications")]
        RETRY["Retry Queue"]
        DLQ[("Dead Letter Queue<br/>Failed Messages")]
        KAFKA -. "failed" .-> RETRY
        RETRY -. "retry" .-> KAFKA
        RETRY -. "gave up" .-> DLQ
    end

    %% ===================== INVENTORY SERVICE =====================
    subgraph INVENTORY_BLOCK["Inventory Service"]
        direction TB
        INVENTORY_S["Inventory Service"]
        INVENTORY_REDIS[("Redis<br/>Cache")]
        INVENTORY_PG[("PostgreSQL<br/>Primary")]
        INVENTORY_PG_R[("PostgreSQL<br/>Replica")]
        INVENTORY_S --> INVENTORY_REDIS
        INVENTORY_S --> INVENTORY_PG
        INVENTORY_PG -. replication .-> INVENTORY_PG_R
    end

    %% ===================== SHIPPING SERVICE =====================
    subgraph SHIPPING_BLOCK["Shipping Service"]
        direction TB
        SHIPPING_S["Shipping Service"]
        SHIPPING_PG[("PostgreSQL<br/>Primary")]
        SHIPPING_PG_R[("PostgreSQL<br/>Replica")]
        SHIPPING_S --> SHIPPING_PG
        SHIPPING_PG -. replication .-> SHIPPING_PG_R
    end

    %% ===================== NOTIFICATION SERVICE =====================
    subgraph NOTIFICATION_BLOCK["Notification Service"]
        direction TB
        NOTIFICATION_S["Notification Service"]
        NOTIFICATION_CASS[("Cassandra<br/>Primary")]
        NOTIFICATION_CASS_R[("Cassandra<br/>Replica")]
        NOTIFICATION_S --> NOTIFICATION_CASS
        NOTIFICATION_CASS -. replication .-> NOTIFICATION_CASS_R
    end

    %% ===================== GATEWAY TO SERVICES =====================
    GW --> USER_S
    GW --> PRODUCT_S
    GW --> SEARCH_S
    GW --> RECO_S
    GW --> CART_S
    GW --> ORDER_S
    CART_S --> ORDER_S
    ORDER_S --> PAYMENT_S

    %% ===================== SERVICES TO KAFKA =====================
    PRODUCT_S -. "catalog" .-> KAFKA
    ORDER_S -. "orders" .-> KAFKA
    PAYMENT_S -. "payments" .-> KAFKA
    INVENTORY_S -. "inventory" .-> KAFKA
    SHIPPING_S -. "shipments" .-> KAFKA

    %% ===================== KAFKA TO SERVICES =====================
    KAFKA -. "catalog" .-> SEARCH_S
    KAFKA -. "orders · catalog" .-> RECO_S
    KAFKA -. "orders" .-> INVENTORY_S
    KAFKA -. "payments" .-> SHIPPING_S
    KAFKA -. "payments · shipments" .-> ORDER_S
    KAFKA -. "orders · payments · shipments" .-> NOTIFICATION_S

    %% ===================== STYLES =====================
    classDef svc fill:#1e1b4b,stroke:#818cf8,color:#e0e7ff,stroke-width:2px
    classDef pg fill:#1e3a8a,stroke:#60a5fa,color:#dbeafe
    classDef mongo fill:#14532d,stroke:#4ade80,color:#dcfce7
    classDef redis fill:#7f1d1d,stroke:#f87171,color:#fee2e2
    classDef cass fill:#164e63,stroke:#22d3ee,color:#cffafe
    classDef elastic fill:#365314,stroke:#a3e635,color:#ecfccb
    classDef kafkaNode fill:#4c0519,stroke:#fb7185,color:#ffe4e6,stroke-width:3px
    classDef dlqNode fill:#0c4a6e,stroke:#38bdf8,color:#e0f2fe
    classDef edgeNode fill:#27272a,stroke:#a1a1aa,color:#f4f4f5
    classDef gwNode fill:#312e81,stroke:#a5b4fc,color:#eef2ff,stroke-width:3px

    class USER_S,PRODUCT_S,SEARCH_S,RECO_S,CART_S,ORDER_S,PAYMENT_S,INVENTORY_S,SHIPPING_S,NOTIFICATION_S svc
    class USER_PG,USER_PG_R,ORDER_PG,ORDER_PG_R,PAYMENT_PG,PAYMENT_PG_R,INVENTORY_PG,INVENTORY_PG_R,SHIPPING_PG,SHIPPING_PG_R pg
    class PRODUCT_MONGO,PRODUCT_MONGO_R,CART_MONGO,CART_MONGO_R mongo
    class USER_REDIS,PRODUCT_REDIS,SEARCH_REDIS,RECO_REDIS,CART_REDIS,INVENTORY_REDIS redis
    class RECO_CASS,RECO_CASS_R,NOTIFICATION_CASS,NOTIFICATION_CASS_R cass
    class SEARCH_ES,SEARCH_ES_R elastic
    class KAFKA kafkaNode
    class DLQ,RETRY dlqNode
    class CLIENT,DNS,CDN,S3,WAF,LB edgeNode
    class GW gwNode
```

---

## What Each Service Does

| Service | What It Does | Database |
| --- | --- | --- |
| **User Service** | Sign up, login, manage profiles | PostgreSQL + Redis |
| **Product Service** | Product catalog — names, prices, images | MongoDB + Redis |
| **Search Service** | Search products, filters, autocomplete | Elasticsearch + Redis |
| **Recommendation Service** | "You might also like" suggestions | Cassandra + Redis |
| **Cart Service** | Add to cart, update quantities | MongoDB + Redis |
| **Order Service** | Create orders, track order status | PostgreSQL |
| **Payment Service** | Process payments, handle refunds | PostgreSQL |
| **Inventory Service** | Track stock, reserve on order | PostgreSQL + Redis |
| **Shipping Service** | Create shipments, track delivery | PostgreSQL |
| **Notification Service** | Send order and shipping updates via email and SMS | Cassandra |

---

## How Kafka Connects Services

When Order Service creates a new order, it puts a message on the `orders` topic. Inventory Service picks it up when ready. No service blocks another.

| Topic | Who Sends | Who Receives |
| --- | --- | --- |
| `orders` | Order Service | Inventory, Recommendation, Notification |
| `payments` | Payment Service | Shipping, Order, Notification |
| `catalog` | Product Service | Search, Recommendation |
| `inventory` | Inventory Service | — |
| `shipments` | Shipping Service | Order, Notification |

**3 Kafka Servers** — every message is copied to all 3. If one dies, the other two still have the data.

---

## Dead Letter Queue

When a message fails:

1. **Fails** → goes to Retry Queue
2. **Retry Queue** tries again
3. **If all retries fail** → goes to Dead Letter Queue
4. **Dead Letter Queue** stores it so engineers can fix the issue and replay it

No message is ever lost.

---

## Color Guide

| Color | What It Means |
| --- | --- |
| 🟣 Purple | Services |
| 🔵 Blue | PostgreSQL |
| 🟢 Green | MongoDB |
| 🔴 Red | Redis Cache |
| 🩵 Cyan | Cassandra |
| 🟡 Yellow-Green | Elasticsearch |
| 🩷 Pink | Kafka |
| ⬜ Gray | Client, DNS, CDN, Firewall, Load Balancer |
