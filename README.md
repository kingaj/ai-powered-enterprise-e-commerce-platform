# AI-Powered Enterprise E-Commerce Platform

A production-grade, event-driven e-commerce platform: Java 21 · Spring Boot 4 · Spring Cloud · Spring AI · Kafka ·
PostgreSQL/pgvector · Elasticsearch · Redis · Keycloak · Docker/Kubernetes · AWS & Azure.

> **Status:** Phase 1 (architecture & planning). Folder skeleton only — no code yet.

## Architecture

```mermaid
flowchart TB
    subgraph Clients["Clients"]
        direction LR
        WEB["Web Store"]
        MOB["Mobile App"]
        CHAT["AI Chat Assistant"]
    end

    subgraph Edge["Edge and Security"]
        direction LR
        GW["API Gateway<br/>routing · JWT check<br/>rate limiting · CORS"]
        KC["Keycloak<br/>OAuth2 / OpenID Connect"]
        DISC["Discovery Service<br/>Eureka locally<br/>Kubernetes DNS in cloud"]
    end

    subgraph Services["Microservices — Spring Boot 4 · Java 21"]
        direction LR
        subgraph Commerce["Commerce"]
            direction TB
            IDS["Identity"]
            CUS["Customer<br/>profile · wishlist"]
            CAT["Product Catalog<br/>pricing · reviews"]
            INV["Inventory<br/>stock reservations"]
            CART["Shopping Cart"]
            ORD["Order<br/>saga orchestrator"]
            PAY["Payment"]
        end
        subgraph Platform["Platform"]
            direction TB
            SRCH["Search<br/>keyword + semantic"]
            FILE["File Management<br/>uploads · imports · reports"]
            NOTIF["Notification<br/>email"]
        end
        subgraph AISvc["AI — Spring AI"]
            direction TB
            ASSIST["AI Shopping Assistant<br/>agent · tools · RAG support"]
            REC["AI Recommendations"]
        end
    end

    BUS[["Apache Kafka + Schema Registry<br/>order saga · search indexing · emails · file jobs<br/>transactional outbox · Debezium CDC · dead-letter topics"]]

    subgraph Data["Data Stores — each service owns its own data"]
        direction LR
        PG[("PostgreSQL<br/>one database per service")]
        PGV[("pgvector<br/>RAG embeddings")]
        ES[("Elasticsearch<br/>search index")]
        RD[("Redis<br/>cache · carts · rate limits")]
        OBJ[("S3 / Azure Blob<br/>files")]
    end

    subgraph External["External Services"]
        direction LR
        LLM{{"LLM provider<br/>Ollama · Claude · OpenAI"}}
        STRIPE{{"Stripe<br/>test mode"}}
        MAIL{{"Email<br/>Mailpit · SES · Azure"}}
        CLAM{{"ClamAV<br/>virus scan"}}
    end

    Clients -->|HTTPS| Edge
    Edge -->|REST| Services
    Services <-->|async events| BUS
    BUS ~~~ Data
    Services --> Data
    Services --> External
```

**How the services talk:** the client waits for an answer → REST through the gateway.
Multi-step or background work → Kafka events. This covers the order saga, updating the search index,
emails, recommendations and file processing.

**Cross-cutting:** OpenTelemetry tracing · Prometheus + Grafana · centralized logs (ELK/OpenSearch) ·
Docker Compose locally · Kubernetes + Helm · GitHub Actions CI/CD · Terraform on AWS (primary) and Azure.

### Checkout flow (order saga)

```mermaid
sequenceDiagram
    autonumber
    actor C as Customer
    participant O as Order Service
    participant K as Kafka
    participant I as Inventory Service
    participant P as Payment Service
    participant N as Notification Service

    C->>O: POST /orders (Idempotency-Key header)
    O->>K: OrderCreated + ReserveInventory
    K->>I: ReserveInventory
    I->>K: InventoryReserved
    K->>O: InventoryReserved
    O->>K: ProcessPayment
    K->>P: ProcessPayment
    P->>K: PaymentCompleted
    K->>O: PaymentCompleted
    O->>K: OrderConfirmed
    K->>N: OrderConfirmed
    N-->>C: Confirmation email
    Note over O,P: On failure the order service runs compensations:<br/>release the stock reservation and/or refund, then mark the order CANCELLED
```

Full details: [docs/DEVELOPMENT_ROADMAP.md](docs/DEVELOPMENT_ROADMAP.md).
The image in `docs/architecture/diagrams/aiecommerce-architecture-reference.jpeg` was the original inspiration only.
It is not this project's architecture: it shows Pinecone, Snowflake, Kubeflow and Feast, which this project does not use.

## Start here

| Document | Purpose |
|---|---|
| [docs/PROJECT_STRUCTURE.md](docs/PROJECT_STRUCTURE.md) | Repository layout and per-service package structure |
| [docs/DEVELOPMENT_ROADMAP.md](docs/DEVELOPMENT_ROADMAP.md) | Architecture summary, tech choices, 18-phase roadmap, decisions awaiting approval |
| [docs/PHASE_TASKS.md](docs/PHASE_TASKS.md) | Every task per phase: what, how, and done-when criteria |
