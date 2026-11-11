# High Level Design · E-Commerce Platform

Scalable microservices architecture for an e-commerce platform like Amazon or Flipkart.

---

## How It Works

A customer opens the app, browses products, adds items to cart, places an order, makes payment, and receives the product. Behind the scenes, 10 independent services handle each part of this journey, talking to each other through Apache Kafka so no service blocks another.

> **Solid lines** = Synchronous — the user is waiting for a response
>
> **Dotted lines** = Asynchronous — happens in the background via Kafka

---

## Architecture Diagram

```mermaid
flowchart LR

    %% ===================== LEGEND =====================
    subgraph LEGEND["Legend"]
        direction TB
        L1["Solid Line = Synchronous · User Waiting"]
        L2["Dotted Line = Asynchronous · Via Kafka"]
    end

    %% ===================== EDGE LAYER =====================
    subgraph EDGE["Edge Layer"]
        direction TB
        CLIENT["Client<br/>Website · Mobile App · Admin Panel"]
        DNS["DNS"]
        CDN["Content Delivery Network<br/>CloudFront"]
        S3[("S3 Storage<br/>Images · Static Files")]
        WAF["Web Application Firewall"]
        LB["Load Balancer"]
        CLIENT --> DNS --> CDN --> WAF --> LB
        CDN --> S3
    end

    GW["API Gateway<br/>Token Validation<br/>Rate Limiting<br/>Request Routing"]
    LB --> GW

    %% ===================== USER SERVICE =====================
    subgraph USER_BLOCK["User Service"]
        direction TB
        USER_S["User Service"]
        USER_REDIS[("Redis<br/>Session · Token")]
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
        PRODUCT_REDIS[("Redis<br/>Catalog Cache")]
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
        SEARCH_REDIS[("Redis<br/>Query Cache")]
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
        RECO_REDIS[("Redis<br/>Top-N Cache")]
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
        CART_REDIS[("Redis<br/>Active Carts")]
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

    PAYMENT_PROVIDER["Payment Provider<br/>Razorpay · Stripe<br/>External"]
    PAYMENT_S --> PAYMENT_PROVIDER

    %% ===================== KAFKA =====================
    subgraph KAFKA_BLOCK["Apache Kafka · Message Queue"]
        direction TB
        KAFKA[("Kafka · 3 Brokers<br/>Topics: orders · payments · catalog<br/>inventory · shipments · notifications")]
        RETRY["Retry Queue<br/>Waits Longer Each Retry"]
        DLQ[("Dead Letter Queue<br/>Failed Messages")]
        KAFKA -. "message failed" .-> RETRY
        RETRY -. "retry" .-> KAFKA
        RETRY -. "all retries failed" .-> DLQ
    end

    %% ===================== INVENTORY SERVICE =====================
    subgraph INVENTORY_BLOCK["Inventory Service"]
        direction TB
        INVENTORY_S["Inventory Service"]
        INVENTORY_REDIS[("Redis<br/>Stock Cache")]
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

    COURIER["Courier · Logistics API<br/>External"]
    SHIPPING_S --> COURIER

    %% ===================== NOTIFICATION SERVICE =====================
    subgraph NOTIFICATION_BLOCK["Notification Service"]
        direction TB
        NOTIFICATION_S["Notification Service"]
        NOTIFICATION_CASS[("Cassandra<br/>Primary")]
        NOTIFICATION_CASS_R[("Cassandra<br/>Replica")]
        NOTIFICATION_S --> NOTIFICATION_CASS
        NOTIFICATION_CASS -. replication .-> NOTIFICATION_CASS_R
    end

    EMAIL_SMS["Email · SMS<br/>External"]
    NOTIFICATION_S --> EMAIL_SMS

    %% ===================== SYNCHRONOUS FLOWS =====================
    GW --> USER_S
    GW --> PRODUCT_S
    GW --> SEARCH_S
    GW --> RECO_S
    GW --> CART_S
    GW --> ORDER_S
    CART_S --> ORDER_S
    ORDER_S --> PAYMENT_S

    %% ===================== ASYNC PRODUCERS =====================
    PRODUCT_S -. "catalog" .-> KAFKA
    ORDER_S -. "orders" .-> KAFKA
    PAYMENT_S -. "payments" .-> KAFKA
    INVENTORY_S -. "inventory" .-> KAFKA
    SHIPPING_S -. "shipments" .-> KAFKA

    %% ===================== ASYNC CONSUMERS =====================
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
    classDef ext fill:#422006,stroke:#fb923c,color:#ffedd5,stroke-dasharray:5 5
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
    class PAYMENT_PROVIDER,COURIER,EMAIL_SMS ext
    class KAFKA kafkaNode
    class DLQ,RETRY dlqNode
    class CLIENT,DNS,CDN,S3,WAF,LB edgeNode
    class GW gwNode
```

---

## What Each Service Does

| Service | What It Does | Database | Why This Database |
| --- | --- | --- | --- |
| **User Service** | Sign up, login, manage profiles, sessions | PostgreSQL + Redis | Reliable user data + fast session lookups |
| **Product Service** | Product catalog — names, prices, images, variants | MongoDB + Redis | Flexible product schemas + hot product cache |
| **Search Service** | Full-text product search, filters, autocomplete | Elasticsearch + Redis | Built for fast text search + frequent query cache |
| **Recommendation Service** | "You might also like" suggestions | Cassandra + Redis | Handles massive click-tracking writes + top-N cache |
| **Cart Service** | Add to cart, update quantities, persist across devices | MongoDB + Redis | Flexible cart structure + active cart quick access |
| **Order Service** | Create orders, track status, coordinate the order flow | PostgreSQL | Orders must never be inconsistent |
| **Payment Service** | Process payments, handle refunds | PostgreSQL | Money transactions need strong consistency |
| **Inventory Service** | Track stock, reserve on order, release on cancellation | PostgreSQL + Redis | Accurate counts + real-time stock display |
| **Shipping Service** | Create shipments, generate tracking, update delivery status | PostgreSQL | Reliable shipment records |
| **Notification Service** | Send order confirmations, receipts, shipping updates | Cassandra | High-volume notification logs |

---

## How Kafka Connects Everything

When the Order Service creates a new order, it doesn't call the Inventory Service directly. Instead it puts a message on the `orders` topic. The Inventory Service picks it up when it's ready. No service blocks another.

| Topic | Who Sends | Who Receives |
| --- | --- | --- |
| `orders` | Order Service | Inventory, Recommendation, Notification |
| `payments` | Payment Service | Shipping, Order, Notification |
| `catalog` | Product Service | Search, Recommendation |
| `inventory` | Inventory Service | *(future consumers)* |
| `shipments` | Shipping Service | Order, Notification |

**3 Kafka Brokers** — every message is copied to all 3 servers. If one dies, the other two still have the data.

---

## Dead Letter Queue

When a message fails, it doesn't disappear:

1. **First attempt fails** → goes to the Retry Queue
2. **Retry Queue** waits longer each time and sends it back
3. **After all retries fail** → moves to Dead Letter Queue
4. **Dead Letter Queue** stores it permanently for engineers to review and fix

**No message is ever lost.**

---

## How an Order Is Placed (Step by Step)

| Step | What Happens | If It Fails |
| --- | --- | --- |
| 1 | User clicks "Place Order" → Order Service creates order | — |
| 2 | Kafka tells Inventory Service → stock reserved | Cancel order |
| 3 | Order Service calls Payment Service → payment charged | Release stock |
| 4 | Kafka tells Shipping Service → shipment created | Refund payment + release stock |
| 5 | Kafka tells Notification Service → email and SMS sent | Log failure, retry later |

If any step fails, all previous steps are undone automatically.

---

## Color Guide

| Color | What It Represents |
| --- | --- |
| 🟣 Purple | Microservices |
| 🔵 Blue | PostgreSQL databases |
| 🟢 Green | MongoDB databases |
| 🔴 Red | Redis caches |
| 🩵 Cyan | Cassandra databases |
| 🟡 Yellow-Green | Elasticsearch |
| 🟠 Orange (dashed border) | External services |
| 🩷 Pink | Apache Kafka |
| ⬜ Gray | Edge layer |
