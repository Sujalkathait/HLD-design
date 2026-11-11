# E-Commerce Platform · High Level Design (HLD)

Comprehensive microservices architecture for an e-commerce platform (like Amazon or Flipkart), designed to support **1 Million daily active users**.

---

## Architecture Diagram

![E-Commerce Architecture Diagram](../../images/ecommerce-architecture.jpg)

> **Source File**: [`ecommerce-hld.mmd`](./ecommerce-hld.mmd)

---

## Functional Requirements

1. **User Authentication & Profiles**: Sign up, log in, session handling, profile updates.
2. **Product Catalog**: Browse categories, view product details, pricing, variants, and stock.
3. **Search & Discovery**: Full-text keyword search with filters (category, price, ratings, brand) and autocomplete.
4. **Recommendations**: Personalized product suggestions ("You might also like") based on browsing history.
5. **Shopping Cart**: Add/remove items, adjust quantities, sync cart across devices.
6. **Order Placement**: Place orders from cart, generate invoice, calculate totals.
7. **Payment Processing**: Handle payments (UPI, credit/debit card, net banking), handle refunds.
8. **Inventory Management**: Track stock levels in real time, reserve stock on order placement, release stock on cancellation.
9. **Shipping & Fulfillment**: Generate tracking numbers, update delivery status (placed → packed → shipped → delivered).
10. **Notifications**: Send real-time updates via Email and SMS on key order lifecycle events.

---

## Non-Functional Requirements

| Metric | Target | Rationale |
| --- | --- | --- |
| **Latency** | < 200ms page load, < 500ms search | Keeps customer drop-off low |
| **Availability** | 99.9% uptime | Critical for revenue; downtime loses sales |
| **Scalability** | 1M+ DAU | Ability to auto-scale horizontally during sales events |
| **Throughput** | ~120 req/s normal, ~1,200 req/s peak | Accommodates 10x spikes during peak flash sales |
| **Data Consistency** | Strong consistency for payments/orders | Zero tolerance for duplicate charges or overselling |
| **Fault Isolation** | Resilient decoupled microservices | If recommendation or notification fails, checkout still succeeds |
| **Security** | HTTPS, Cloudflare WAF, Token auth | Protect customer payment details and user data |

---

## Capacity Estimation (1 Million Users)

### Traffic Estimation
- **Daily Active Users (DAU)**: 1,000,000 users/day
- **Average Requests / User**: ~10 interactions/day = 10,000,000 requests/day
- **Average QPS**: $10,000,000 \div 86,400 \approx$ **120 requests/second**
- **Peak Traffic (3x)**: **~360 requests/second**
- **Flash Sale Spike (10x)**: **~1,200 requests/second**

### Storage Estimation
- **User Records**: $1\text{M} \times 2\text{ KB} =$ **2 GB**
- **Product Catalog**: $500\text{K products} \times 5\text{ KB} =$ **2.5 GB**
- **Product Images (S3)**: $500\text{K products} \times 5\text{ images} \times 500\text{ KB} =$ **1.25 TB**
- **Orders (PostgreSQL)**: $50\text{K orders/day} \times 3\text{ KB} \times 365\text{ days} =$ **~55 GB/year**
- **Active Carts (Redis)**: $200\text{K active carts} \times 1\text{ KB} =$ **200 MB**

---

## How the Diagram Arrows Work

| Arrow Type | Flow Type | Meaning |
| --- | --- | --- |
| **Solid line** `──────→` | **Synchronous** | The user is actively waiting for an immediate response (e.g., Client → API Gateway → Order Service). |
| **Dotted line** `- - - →` | **Asynchronous** | Happens in the background via Apache Kafka without blocking the user (e.g., Order Service → Kafka → Inventory Service). |
| **Dotted "replication"** | **Data Sync** | Database primary continuously streams updates to read replicas for failover and read scaling. |

---

## Interactive Mermaid Diagram

```mermaid
flowchart LR

    %% ===================== CLIENT TO GATEWAY =====================
    CLIENT["Client<br/>Website · Mobile App"] --> DNS["DNS"] --> CDN["Cloudflare<br/>CDN + Firewall"] --> LB["Load Balancer"]
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

## Step-by-Step Order Lifecycle

1. **User Request**: User connects via DNS → Cloudflare CDN/WAF → Load Balancer → API Gateway.
2. **Auth Verification**: API Gateway verifies JWT token and enforces rate limits.
3. **Product Exploration**: Product data is fetched instantly from Redis cache (or MongoDB if cache miss).
4. **Search Queries**: Elasticsearch delivers fast full-text filtering and autocomplete.
5. **Recommendations**: Cassandra tracks user activities to generate "You might also like" feeds.
6. **Cart Management**: Cart updates write to Redis for zero latency, backed up in MongoDB.
7. **Order Creation**: User clicks "Place Order"; Cart Service triggers Order Service to write an order record in PostgreSQL.
8. **Payment Handling**: Order Service calls Payment Service synchronously to charge the payment.
9. **Event Publishing**: Order Service publishes a message to the Kafka `orders` topic.
10. **Inventory Reservation**: Inventory Service consumes `orders` and decrements stock.
11. **Shipment Creation**: Payment Service publishes to `payments` topic; Shipping Service consumes it and generates tracking.
12. **Notifications**: Notification Service listens to Kafka topics and sends Email/SMS updates.
13. **Order Confirmation**: Customer receives instant confirmation on screen.

---

## Microservices Breakdown

| Service | Primary Responsibility | Storage / Cache | Key Reasoning |
| --- | --- | --- | --- |
| **User Service** | User profiles, authentication | PostgreSQL + Redis | ACID compliance for credentials; Redis for sessions |
| **Product Service** | Product catalog and metadata | MongoDB + Redis | Schema flexibility for varied product categories |
| **Search Service** | Catalog search and filtering | Elasticsearch + Redis | Inverted index for fast full-text queries |
| **Recommendation** | Personalized product feeds | Cassandra + Redis | High write-throughput for browsing signals |
| **Cart Service** | Cart state management | MongoDB + Redis | Fast read/write in Redis; persistent cart in Mongo |
| **Order Service** | Order creation & lifecycle | PostgreSQL | Strict ACID consistency for financial order records |
| **Payment Service** | Payment authorization & refunds | PostgreSQL | Strong transactional consistency for balance & logs |
| **Inventory Service** | Real-time stock counting | PostgreSQL + Redis | Strict counters to prevent overselling |
| **Shipping Service** | Dispatch & tracking updates | PostgreSQL | Reliable relational shipment records |
| **Notification Service** | Customer notifications (SMS/Email) | Cassandra | High append volume for notification history |

---

## Message Queue & Dead Letter Queue

Apache Kafka acts as the distributed event backbone:
- **Topics**: `orders`, `payments`, `catalog`, `inventory`, `shipments`, `notifications`
- **Replication Factor**: 3 brokers ensure high availability with zero data loss if one node fails.
- **Retry Mechanism**: Failed events route to a **Retry Queue** for transient error recovery.
- **Dead Letter Queue (DLQ)**: If retries are exhausted, events are moved to the DLQ for manual inspection and replay.
