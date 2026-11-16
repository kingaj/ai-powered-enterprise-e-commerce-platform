# Project Structure

> Status: **Phase 1 – proposed, awaiting approval.** Folders exist as an empty skeleton (`.gitkeep`).
> No `pom.xml` or code has been generated yet. That starts in Phase 2, after the open decisions in
> [DEVELOPMENT_ROADMAP.md](DEVELOPMENT_ROADMAP.md#7-decisions-that-need-your-approval) are approved.

## 1. Repository layout (monorepo, Maven multi-module)

```text
ai-powered-enterprise-e-commerce-platform/
├── pom.xml                         # (Phase 2) root aggregator + parent: Java/Spring versions, plugins, quality gates
├── README.md
├── .github/
│   └── workflows/                  # (Phase 15) CI/CD: build, test, Sonar, scans, image build, deploy
│
├── platform-libs/                  # Shared TECHNICAL libraries only. Never shared domain entities.
│   ├── platform-bom/               # Bill of Materials: pins every 3rd-party version in one place
│   ├── common-core/                # ApiResponse/ErrorResponse records, ErrorCode, base exceptions
│   ├── common-web/                 # Global exception handler, correlation-ID filter, OpenAPI defaults, validation
│   ├── common-security/            # JWT resource-server config, role constants, method-security helpers
│   ├── common-kafka/               # Producer/consumer defaults, outbox table + relay, idempotent-consumer support, DLT
│   ├── common-observability/       # Micrometer/OTel defaults, MDC logging, structured JSON log config
│   ├── event-schemas/              # Avro .avsc contracts + generated classes (the ONLY cross-service model)
│   └── common-test/                # Testcontainers base classes (Postgres, Kafka, Redis, ES), test fixtures
│
├── services/                       # One independently deployable Spring Boot app per folder
│   ├── api-gateway/                # Spring Cloud Gateway: routing, JWT validation, rate limiting, CORS (acts as BFF)
│   ├── discovery-service/          # Eureka (local/Compose). Replaced by Kubernetes DNS in K8s profiles
│   ├── identity-service/           # Registration, login facade over Keycloak, user lifecycle events
│   ├── customer-service/           # Customer profile, addresses, preferences, wishlist
│   ├── product-catalog-service/    # Products, categories, attributes, pricing, reviews & ratings, AI description drafts
│   ├── inventory-service/          # Stock levels, reservations (locking), releases
│   ├── cart-service/               # Shopping cart (Redis-backed with Postgres snapshot)
│   ├── order-service/              # Order lifecycle + Saga orchestrator + order tracking
│   ├── payment-service/            # Payment gateway sandbox (Stripe test mode), idempotency keys, refunds
│   ├── notification-service/       # Email (Mailpit locally / SES / ACS), templates, delivery log
│   ├── search-service/             # Elasticsearch index owner: full-text, facets, autocomplete, hybrid semantic search
│   ├── file-service/               # Pre-signed uploads (S3/Azure Blob), chunked/resumable, virus scan, CSV/Excel import, reports
│   ├── ai-recommendation-service/  # Recommendations from purchase history + embeddings, "why recommended"
│   └── ai-assistant-service/       # Shopping assistant, controlled agent (tool calling), RAG customer support
│
├── infrastructure/
│   ├── docker/                     # (Phase 2) docker-compose files + container config
│   │   ├── postgres/init/          # One database + one user per service, pgvector extension
│   │   ├── kafka/connectors/       # Debezium outbox connector JSON (Phase 7)
│   │   ├── keycloak/               # Realm export: clients, roles, test users
│   │   ├── elasticsearch/          # Index templates, analyzers, synonyms
│   │   └── observability/          # prometheus/, grafana/dashboards/, otel-collector/, logging/ (ELK or OpenSearch)
│   ├── k8s/                        # (Phase 15) Kustomize base + overlays: local (kind), aws, azure
│   ├── helm/charts/                # (Phase 15) One reusable "spring-service" chart + umbrella chart
│   └── terraform/                  # (Phases 16-17) aws/ and azure/ IaC — create on demand, destroy after use
│
├── tests/
│   ├── e2e/                        # Black-box journey tests against the running stack (register → order → email)
│   ├── contract/                   # Consumer-driven contracts (Spring Cloud Contract or Pact)
│   └── performance/                # k6 and/or Gatling load scenarios
│
├── docs/
│   ├── PROJECT_STRUCTURE.md        # this file
│   ├── DEVELOPMENT_ROADMAP.md      # phase-wise roadmap
│   ├── PHASE_TASKS.md              # phase-wise task breakdown with how-to
│   ├── architecture/diagrams/      # Mermaid sources + reference image
│   ├── adr/                        # Architecture Decision Records (ADR-0001 ...)
│   ├── runbooks/                   # Production scenarios / troubleshooting playbooks
│   └── learning/                   # TECHNICAL_GAP_TRACKER.md, interview question banks per phase
│
└── scripts/                        # Dev helper scripts (start-infra.sh, seed-data.sh, cleanup-cloud.sh)
```

## 2. Inside one service (example: `order-service`)

Package-by-feature on the outside, light layering (ports & adapters) on the inside. We avoid full
hexagonal ceremony where a service is plain CRUD — the code-quality rules ask for "no unnecessary abstractions".

```text
services/order-service/
├── pom.xml
├── Dockerfile                                  # multi-stage, layered jar, non-root user
├── src/main/java/com/aiecommerce/order/
│   ├── OrderServiceApplication.java
│   ├── api/                                    # INBOUND adapters
│   │   ├── OrderController.java
│   │   ├── dto/                                # request/response records (never entities)
│   │   └── mapper/                             # MapStruct mappers
│   ├── application/                            # use cases, transactions live here
│   │   ├── OrderService.java
│   │   └── saga/                               # OrderSagaOrchestrator, saga state machine
│   ├── domain/                                 # entities, value objects, domain rules, repository interfaces
│   │   ├── Order.java, OrderItem.java, OrderStatus.java
│   │   └── OrderRepository.java
│   └── infrastructure/                         # OUTBOUND adapters
│       ├── persistence/                        # JPA specifics, custom queries
│       ├── messaging/                          # Kafka listeners, outbox writer
│       ├── client/                             # REST clients (RestClient/HTTP interface) + resilience
│       └── config/
├── src/main/resources/
│   ├── application.yml                         # 12-factor: everything overridable by env vars
│   ├── application-local.yml
│   └── db/migration/V1__create_orders.sql      # Flyway
└── src/test/java/com/aiecommerce/order/
    ├── unit/                                   # JUnit 5 + Mockito
    ├── integration/                            # @SpringBootTest + Testcontainers
    └── contract/                               # producer-side contract tests
```

Base package: `com.aiecommerce.<service>`. Maven groupId: `com.aiecommerce`.

## 3. Rules that the structure enforces

| Rule | How |
|---|---|
| No shared database | Each service gets its own Postgres database + user; credentials only for its own DB |
| No shared JPA entities | `platform-libs` contains only technical code; cross-service data travels as Avro events or REST DTOs |
| Independently deployable | Each service has its own `pom.xml`, `Dockerfile`, Helm values, and version |
| One place for versions | `platform-bom` + root `pom.xml` `<dependencyManagement>` |
| Config outside code | `application.yml` reads env vars; secrets from `.env` (git-ignored) locally, Secrets Manager / Key Vault in cloud |

## 4. Why a monorepo?

- **Chosen:** one repo, Maven multi-module, but each service builds/deploys on its own (CI uses path filters).
- **Alternative:** repo-per-service (polyrepo). Better team autonomy at scale, but for one engineer it multiplies
  CI setup, version bumps of shared libs, and cross-cutting refactors.
- **Real world:** many companies (Google, Uber, Shopify) run monorepos; others use polyrepos with an internal
  artifact registry. The service boundaries matter more than the repo layout.

## 5. Mapping to the reference image (`docs/architecture/diagrams/aiecommerce-architecture-reference.jpeg`)

| Image block | In this project |
|---|---|
| Frontend (mobile/web/chatbot) | Out of scope for the backend roadmap; APIs + optional thin demo UI later |
| BFF API layer (GraphQL/REST) | `api-gateway` (REST). GraphQL BFF is an optional stretch task |
| Product / Order / User | `product-catalog`, `order`, `identity` + `customer` services |
| Event bus (Kafka) | Kafka (KRaft) + Schema Registry |
| AI/ML: real-time inference | `ai-assistant-service`, `ai-recommendation-service` via Spring AI |
| Vector DB (Pinecone) | **pgvector** (RAG) + Elasticsearch `dense_vector` (product hybrid search) — see ADR in Phase 1 |
| Feature store (Feast), Lakehouse (Snowflake), MLOps (Kubeflow) | Out of scope: we use hosted/open LLMs, not training our own models. Noted as future learning |
