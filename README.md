# High Level Design · E-Commerce Platform

Microservices architecture for an e-commerce platform like Amazon or Flipkart, designed for **1 Million daily active users**.

---

## Functional Requirements

*What the system should do:*

1. User can **sign up, log in, and manage profile**
2. User can **browse products** by category
3. User can **search products** with filters (price, rating, brand)
4. User gets **product recommendations** ("You might also like")
5. User can **add products to cart** and update quantities
6. User can **place an order** from cart
7. User can **make payment** (UPI, card, net banking)
8. User can **track order status** (placed → shipped → delivered)
9. User receives **email and SMS notifications** (order confirmation, shipping update)
10. Admin can **add, update, delete products** and manage inventory

---

## Non-Functional Requirements

*How well the system should perform:*

| Requirement | Target |
| --- | --- |
| **Response Time** | < 200ms for product pages, < 500ms for search |
| **Availability** | 99.9% uptime (max ~8 hours downtime per year) |
| **Scalability** | Handle 1M+ daily active users, scale up during sales |
| **Throughput** | ~120 requests/second normal, ~1200 requests/second peak |
| **Data Consistency** | Orders and payments must never be lost or duplicated |
| **Fault Tolerance** | If one service goes down, others keep working |
| **Security** | All data encrypted, token-based authentication |

---

## Capacity Estimation · 1 Million Users

### Traffic

| Metric | Calculation | Result |
| --- | --- | --- |
| Daily Active Users | — | 1,000,000 |
| Requests per user per day | ~10 pages per session | 10,000,000 requests/day |
| Requests per second (average) | 10M ÷ 86,400 | **~120 requests/sec** |
| Peak traffic (3x average) | 120 × 3 | **~360 requests/sec** |
| Flash sale peak (10x average) | 120 × 10 | **~1,200 requests/sec** |

### Storage

| Data | Calculation | Result |
| --- | --- | --- |
| User profiles | 1M users × 2 KB each | **2 GB** |
| Product catalog | 500K products × 5 KB each | **2.5 GB** |
| Product images | 500K products × 5 images × 500 KB | **1.25 TB** |
| Orders per year | 50K orders/day × 3 KB × 365 days | **55 GB/year** |
| Cart data | 200K active carts × 1 KB | **200 MB** |

### Bandwidth

| Direction | Calculation | Result |
| --- | --- | --- |
| Incoming (user requests) | 120 req/sec × 2 KB each | **240 KB/sec** |
| Outgoing (product pages + images) | 120 req/sec × 500 KB each | **60 MB/sec** |

---

## How the Diagram Arrows Work

| Arrow Type | Meaning | Example |
| --- | --- | --- |
| **Solid line** `──────→` | User is waiting for a response. The request goes and comes back before anything else happens. | User clicks "Place Order" → Order Service creates order → User sees "Order Placed" |
| **Dotted line** `- - - →` | Happens in background. The user doesn't wait. One service puts a message in Kafka, another picks it up later. | Order placed → Kafka message → Inventory Service reserves stock (user doesn't wait for this) |
| **Dotted "replication"** | Database copies its data to a backup server. If the main server dies, the backup takes over. | PostgreSQL Primary → PostgreSQL Replica |

---

## Architecture Diagram

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

## Step-by-Step: What Happens When a User Places an Order

| Step | What Happens | Who Does It |
| --- | --- | --- |
| 1 | User opens the website | Client |
| 2 | Request goes through DNS → Cloudflare → Load Balancer → API Gateway | Edge layer |
| 3 | API Gateway checks if the user's token is valid | API Gateway |
| 4 | User browses products — data comes from Redis cache (fast) or MongoDB (if not cached) | Product Service |
| 5 | User searches "laptop" — Elasticsearch finds matching products | Search Service |
| 6 | User sees "You might also like" — based on past browsing | Recommendation Service |
| 7 | User adds a laptop to cart — saved in Redis for fast access, backed up in MongoDB | Cart Service |
| 8 | User clicks "Place Order" — Order Service creates the order in PostgreSQL | Order Service |
| 9 | Order Service calls Payment Service — payment is charged | Payment Service |
| 10 | Order Service puts a message on Kafka `orders` topic | Kafka |
| 11 | Inventory Service picks up the message and reserves stock | Inventory Service |
| 12 | Payment Service puts a message on Kafka `payments` topic | Kafka |
| 13 | Shipping Service picks it up and creates a shipment | Shipping Service |
| 14 | Notification Service sends email and SMS to the user | Notification Service |
| 15 | User sees "Order Placed Successfully" on screen | Client |

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
| **Notification Service** | Send updates via email and SMS | Cassandra |

---

## How Kafka Connects Services

When Order Service creates an order, it puts a message in Kafka. Other services pick it up in the background. The user doesn't wait.

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
| ⬜ Gray | Client, DNS, Cloudflare, Load Balancer |
