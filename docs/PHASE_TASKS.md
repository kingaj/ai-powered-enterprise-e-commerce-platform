# Phase-wise Task Breakdown

> Companion docs: [DEVELOPMENT_ROADMAP.md](DEVELOPMENT_ROADMAP.md) · [PROJECT_STRUCTURE.md](PROJECT_STRUCTURE.md)
>
> **How to read a task**
> - **What** — the outcome and why it matters.
> - **How** — concrete steps to achieve it.
> - **Done when** — verifiable acceptance criteria (must be actually run, not assumed).
>
> Task IDs: `P<phase>-T<nn>`. Use them in branch names and commits, e.g. `feat(inventory): P4-T07 pessimistic lock reservation`.
> Status legend: ☐ todo · ◐ in progress · ☑ done

---

## Phase 1 — Requirements, Architecture, Technology Decisions, Setup

### ☐ P1-T01 — Install and verify the toolchain
- **What:** A reproducible dev machine. Java 17 is installed; Docker is missing.
- **How:**
  1. Install SDKMAN, then `sdk install java 21.0.x-tem` and `sdk use java 21.0.x-tem`; add `.sdkmanrc` to the repo.
  2. Install a Docker runtime (Docker Desktop / Colima / OrbStack); give it ≥ 8 GB RAM (12–16 GB ideal).
  3. Install `jq`, `httpie` or Postman/Bruno for API calls.
  4. Record versions in `docs/LOCAL_SETUP.md`.
- **Done when:** `java -version` → 21, `docker run hello-world` works, versions are documented.

### ☐ P1-T02 — Write functional & non-functional requirements
- **What:** A short, testable requirements list so scope doesn't creep.
- **How:** In `docs/ARCHITECTURE.md` add: user personas (guest, customer, admin, support agent), core use cases (browse → search → cart → checkout → track → review), NFRs with numbers (e.g. search p95 < 200 ms, checkout availability 99.9 %, 1M products, 2k concurrent users, 2 GB upload max).
- **Done when:** every NFR has a number and a phase where it is verified.

### ☐ P1-T03 — Pin compatible versions
- **What:** Avoid the #1 Spring pain: mismatched Boot/Cloud/AI versions.
- **How:** Check start.spring.io + Spring Cloud release-train table + Spring AI docs; pick Boot 4.0.x, matching Spring Cloud 2025.1.x, Spring AI 2.0.x; record Testcontainers, MapStruct, Resilience4j, Kafka client, ES client versions in a table in ADR-0001.
- **Done when:** a version matrix exists with sources linked.

### ☐ P1-T04 — Domain model & bounded contexts
- **What:** Event storming lite to validate service boundaries.
- **How:** Draw a Mermaid context map: contexts, aggregates (Order, Product, StockItem, Cart, Payment, Customer), upstream/downstream relationships, domain events. Confirm each of the 14 services has a business reason (table already in roadmap §2).
- **Done when:** context map + aggregate list are in ARCHITECTURE.md.

### ☐ P1-T05 — Architecture diagrams (Mermaid)
- **What:** The 9 diagrams the project requires.
- **How:** Create in `docs/ARCHITECTURE.md` (and specialised docs): high-level, microservices, database, Kafka event flow, auth flow (sequence), file upload (sequence), AI/RAG, cloud deployment, CI/CD.
- **Done when:** all 9 render on GitHub.

### ☐ P1-T06 — Write ADRs 0001–0012
- **What:** Record each major decision with context, options, decision, consequences.
- **How:** Use the template `docs/adr/0000-template.md` (Title, Status, Context, Options, Decision, Consequences). One file per ADR listed in the roadmap §4.
- **Done when:** 12 ADRs merged with status "Accepted" after your review.

### ☐ P1-T07 — Draft data ownership & event catalog
- **What:** Early versions of DATABASE_DESIGN.md and KAFKA_EVENTS.md.
- **How:** For each service list its tables (no FKs across services), and for each event list name, topic, key, producer, consumers, purpose.
- **Done when:** both docs exist; every event has exactly one owner/producer.

### ☐ P1-T08 — Baseline TECHNICAL_GAP_TRACKER.md
- **What:** Know where you start so progress is visible.
- **How:** Create `docs/learning/TECHNICAL_GAP_TRACKER.md` with the 18 areas; self-rate 1–5 each; columns: covered, understood, needs practice, implemented, interview readiness, next activities.
- **Done when:** baseline ratings filled in.

### ☐ P1-T09 — Repo hygiene
- **What:** Engineering standards from day one.
- **How:** Add `.gitignore` (Java/Maven/IDE/.env), `.editorconfig`, `CONTRIBUTING.md` (branching: trunk-based with short-lived feature branches, conventional commits), PR template, CODEOWNERS.
- **Done when:** files committed; branch protection on `main` enabled in GitHub.

---

## Phase 2 — Monorepo, Maven Multi-module, Docker Compose Infrastructure

### ☐ P2-T01 — Root POM and BOM
- **What:** One place for versions and plugins.
- **How:** Root `pom.xml` (packaging `pom`) with `spring-boot-starter-parent` or imported Spring Boot BOM, Spring Cloud BOM, Spring AI BOM, Testcontainers BOM; `<pluginManagement>` for compiler (`--release 21`, `-parameters`, MapStruct + Lombok annotation processors in correct order), surefire/failsafe split (`*Test` vs `*IT`), jacoco. `platform-bom` re-exports versions for services.
- **Done when:** `./mvnw -q verify` passes on an empty module tree.

### ☐ P2-T02 — Maven wrapper & build profiles
- **How:** `mvn wrapper:wrapper`; profiles `fast` (skip ITs), `ci` (all checks), `docker` (build images with Jib or Buildpacks/Dockerfile).
- **Done when:** `./mvnw -Pfast verify` and `./mvnw -Pci verify` both work.

### ☐ P2-T03 — `common-core` & `common-web`
- **What:** Consistent API responses and errors across all services.
- **How:** Records `ApiResponse<T>`, `ErrorResponse` aligned with RFC 9457 `ProblemDetail`; `ErrorCode` enum; `BusinessException`, `ResourceNotFoundException`; `@RestControllerAdvice` handling validation, not-found, conflict, optimistic-lock, generic errors (never leaking stack traces); `CorrelationIdFilter` putting `X-Correlation-Id` into MDC; auto-configuration via `META-INF/spring/...AutoConfiguration.imports`.
- **Done when:** unit tests for handler mapping each exception to correct status/body.

### ☐ P2-T04 — Docker Compose: data & messaging infra
- **What:** One command to start everything local.
- **How:** `infrastructure/docker/docker-compose.yml` with **profiles**: `core` (postgres w/ pgvector image, redis, kafka KRaft, schema-registry, kafka-ui), `search` (elasticsearch, kibana), `auth` (keycloak + its DB), `files` (minio, azurite, clamav), `mail` (mailpit), `obs` (later). Health checks on every container, named volumes, a dedicated network, all secrets from `.env` (git-ignored) with `.env.example` committed.
- **Done when:** `docker compose --profile core up -d` → all healthy; `docker compose config` validates.

### ☐ P2-T05 — Database-per-service bootstrap
- **How:** `postgres/init/01-create-databases.sh` creating `catalog_db`, `inventory_db`, `order_db`, … each with its own user/password from env; `CREATE EXTENSION vector` only in `ai_db`.
- **Done when:** each user can connect only to its own DB (verified with psql).

### ☐ P2-T06 — First runnable service template
- **What:** A reference service every later service copies.
- **How:** Scaffold `product-catalog-service` with Actuator, Flyway (`V1__init.sql`), HikariCP config (pool size, timeouts, leak detection), OpenAPI (springdoc), structured logging, multi-stage `Dockerfile` (build → layered jar → JRE 21 distroless/alpine, non-root user), `application.yml` using env vars.
- **Done when:** service starts, `/actuator/health` UP including DB, Flyway applied, image builds.

### ☐ P2-T07 — Code quality gates
- **How:** Checkstyle (Google style, tuned), SpotBugs, Spotless formatter, jacoco minimum coverage (start 60 %, raise later) bound into `ci` profile.
- **Done when:** a deliberately bad commit fails the build.

### ☐ P2-T08 — LOCAL_SETUP.md + scripts
- **How:** `scripts/start-infra.sh`, `scripts/stop-infra.sh`, `scripts/reset-data.sh`; doc lists ports, URLs (Kafka UI, Kibana, Keycloak, MinIO, Mailpit), and memory tips.
- **Done when:** a fresh clone follows the doc to a running stack.

---

## Phase 3 — API Gateway, Authentication, Authorization, Service Discovery

### ☐ P3-T01 — discovery-service (Eureka)
- **How:** Eureka server; services register with `eureka.client` in `local` profile only; `kubernetes` profile disables Eureka and uses DNS URLs.
- **Done when:** Eureka dashboard lists catalog-service; disabling profile still lets services start.

### ☐ P3-T02 — Keycloak realm as code
- **What:** Reproducible identity setup.
- **How:** Realm `aiecommerce` exported to `infrastructure/docker/keycloak/realm.json`: clients `web-app` (public, Auth Code + PKCE), `api-gateway` (confidential), `internal-services` (client credentials); realm roles `CUSTOMER`, `ADMIN`, `SUPPORT`; refresh token rotation ("Revoke refresh token" on, reuse 0); access token TTL 5 min; password policy; Argon2 hashing provider; test users.
- **Done when:** realm auto-imports on container start; token obtained via curl.

### ☐ P3-T03 — api-gateway
- **How:** Spring Cloud Gateway (reactive or MVC variant — choose and record): routes per service with `lb://` URIs, OAuth2 resource server JWT validation, token relay, `RequestRateLimiter` with Redis (per user / per IP), CORS config, correlation-ID propagation, request size limits, global error JSON, timeouts.
- **Done when:** protected route returns 401 without token, 200 with; 429 under burst.

### ☐ P3-T04 — `common-security`
- **How:** Auto-configured `SecurityFilterChain` for resource servers: stateless sessions, JWT → `GrantedAuthority` converter for Keycloak `realm_access.roles`, `@EnableMethodSecurity`, actuator exposure rules, `CurrentUser` helper.
- **Done when:** `@PreAuthorize("hasRole('ADMIN')")` tests pass with `jwt()` MockMvc post-processor.

### ☐ P3-T05 — identity-service
- **What:** Registration and user lifecycle.
- **How:** `POST /api/v1/auth/register` → validate → create user in Keycloak via Admin client (service account) → store minimal user record → publish `CustomerRegistered` (temporary simple publisher, upgraded in Phase 7). Login/refresh done directly against Keycloak (OIDC) — document why the service must not proxy passwords unless needed. Email verification flow.
- **Done when:** registration creates Keycloak user + event; duplicate email → 409.

### ☐ P3-T06 — Security tests & SECURITY.md draft
- **How:** Tests for 401/403/expired token/wrong audience; doc: auth flow sequence diagram, token lifetimes, when to prefer an external IdP (Keycloak/Entra/Cognito/Auth0) vs Spring Authorization Server.
- **Done when:** tests green; doc reviewed.

---

## Phase 4 — Customer, Product Catalog, Inventory Services

### ☐ P4-T01 — customer-service
- **How:** Consume `CustomerRegistered` → create profile; APIs: profile, addresses (max N, default flag), preferences (categories, brands), wishlist (add/remove/list paginated). Users can only access their own data (`@PreAuthorize` with `#customerId == authentication.name`) — prevents IDOR.
- **Done when:** IDOR test (user A reads user B) returns 403.

### ☐ P4-T02 — Catalog schema & migrations
- **How:** Tables: `category` (tree via `parent_id` + materialised path), `product`, `product_attribute` (JSONB with GIN index), `product_price`, `product_image`, `review`, `review_summary`. Indexes for every filter/sort used. `created_at/updated_at/created_by/updated_by` auditing via Spring Data `@EnableJpaAuditing`. `version` column for optimistic locking.
- **Done when:** Flyway migrations reviewed with `EXPLAIN ANALYZE` on key queries.

### ☐ P4-T03 — Catalog APIs
- **How:** Public read APIs (list with pagination/sort/filters via Specifications or Querydsl, keyset pagination for deep pages), admin write APIs (`/admin/products`), DTO records + MapStruct, Bean Validation, publish `ProductCreated/Updated/PriceChanged` events.
- **Done when:** no N+1 (verified with Hibernate statistics / datasource-proxy in tests); max page size enforced.

### ☐ P4-T04 — Reviews & ratings
- **How:** Only customers with a delivered order for that product can review (checked via order events → local `verified_purchase` table); one review per user/product (unique constraint); rating summary updated incrementally; moderation status; XSS-safe text (sanitise on input, encode on output).
- **Done when:** duplicate review → 409; summary correct under concurrent reviews.

### ☐ P4-T05 — Million-product data generator (Scenario #1)
- **How:** A CLI/profile that bulk-inserts 1M products using JDBC batch / `COPY`; measure list query times; add/adjust indexes; document results.
- **Done when:** paginated listing p95 measured and documented before/after indexing.

### ☐ P4-T06 — Redis caching (Scenario #16)
- **How:** Cache-aside for product detail & category tree with `@Cacheable` + Redis, TTL + jitter, explicit eviction on update events, cache stampede protection (lock or `sync=true`), metrics for hit rate.
- **Done when:** hit-rate metric visible; stale data window documented.

### ☐ P4-T07 — inventory-service with safe reservations (Scenario #11)
- **What:** Never oversell.
- **How:** Tables `stock_item(product_id, warehouse_id, available, reserved, version)`, `reservation(order_id, …, status, expires_at)`. Implement reservation two ways and compare: (a) optimistic locking + retry, (b) `SELECT … FOR UPDATE` (pessimistic) — and (c) atomic `UPDATE … SET available = available - :q WHERE available >= :q`. Reservation expiry job releases stale holds.
- **Done when:** a test fires 200 concurrent reservations for 50 units → exactly 50 succeed, for each strategy; ADR records the chosen default.

### ☐ P4-T08 — OpenAPI & API docs
- **How:** springdoc per service, aggregated in gateway Swagger UI; examples for each request/response; start `docs/API_DOCUMENTATION.md`.
- **Done when:** Swagger UI shows all services via gateway.

---

## Phase 5 — Shopping Cart & Order Management

### ☐ P5-T01 — cart-service
- **How:** Cart in Redis hash keyed by user (TTL 30 days) / guest token; add/update/remove items; merge guest cart on login; on checkout re-validate prices & availability from catalog/inventory (REST).
- **Done when:** guest → login merge works; quantity limits validated.

### ☐ P5-T02 — Resilient REST clients (Scenario #5)
- **How:** Spring `RestClient` / HTTP interface clients + Resilience4j: timeout, retry with exponential backoff + jitter (only idempotent calls), circuit breaker, bulkhead; fallback to cached price with "price may change" flag.
- **Done when:** WireMock/Testcontainers test: catalog down → cart still returns, CB opens, recovers.

### ☐ P5-T03 — Order domain & state machine
- **How:** `Order`, `OrderItem` (price snapshot!), `OrderStatus` (PENDING → INVENTORY_RESERVED → PAYMENT_PENDING → CONFIRMED → SHIPPED → DELIVERED; CANCELLED/FAILED), transitions enforced in domain methods; `order_status_history` table for tracking timeline.
- **Done when:** unit tests reject illegal transitions.

### ☐ P5-T04 — Idempotent order creation (Scenario #6)
- **How:** Client sends `Idempotency-Key` header; store key + request hash + response in `idempotency_record` (unique key); repeated request returns stored response; different payload with same key → 422.
- **Done when:** double-click test produces exactly one order.

### ☐ P5-T05 — First events via simple outbox
- **How:** In the same transaction as the order insert, write to `outbox_event`; a scheduled relay publishes to Kafka `order.events` with key = orderId; mark sent. (Upgraded to Debezium in Phase 7.)
- **Done when:** killing the app between commit and publish still publishes after restart.

### ☐ P5-T06 — Order query APIs
- **How:** My orders (paginated), order detail, tracking timeline; admin search by status/date with pagination.
- **Done when:** customer can't see others' orders; admin can.

---

## Phase 6 — Payment Service & Saga

### ☐ P6-T01 — Saga design doc
- **How:** Sequence diagrams for happy path, payment failure, stock unavailable, timeout; commands vs events naming; topic list; compensation table. Justify orchestration (ADR-0002).
- **Done when:** reviewed and approved.

### ☐ P6-T02 — Saga orchestrator in order-service
- **How:** `order_saga` table (saga_id, order_id, step, status, retries, deadline). On `OrderCreated` → send `ReserveInventoryCommand`; on `InventoryReserved` → `ProcessPaymentCommand`; on `PaymentCompleted` → confirm order; on failures → compensations (`ReleaseInventoryCommand`, cancel order). Scheduler handles timed-out sagas.
- **Done when:** saga state persisted at every step and resumable after restart.

### ☐ P6-T03 — Inventory saga participant
- **How:** Consume `ReserveInventoryCommand` → reserve (Phase 4 logic) → emit `InventoryReserved` / `InventoryReservationFailed`; consume `ReleaseInventoryCommand`.
- **Done when:** both outcomes tested with Testcontainers Kafka.

### ☐ P6-T04 — payment-service with Stripe test mode
- **How:** Create PaymentIntent with Stripe idempotency key = orderId; store `payment(order_id UNIQUE, status, provider_ref)`; webhook endpoint verifies `Stripe-Signature`, handles `payment_intent.succeeded/failed` idempotently (store processed event IDs); refunds API for compensation. Keys only from env/secret store.
- **Done when:** sandbox payment succeeds; replayed webhook doesn't double-update.

### ☐ P6-T05 — Duplicate payment prevention (Scenario #7)
- **How:** Three layers: unique constraint on `order_id`, provider idempotency key, state check (only PENDING can be charged). Document each layer.
- **Done when:** test sends the same command 5× concurrently → one charge.

### ☐ P6-T06 — Failure scenario test suite (Scenarios #12, #19)
- **How:** Integration tests: card declined → inventory released, order CANCELLED; out of stock → no payment attempted; payment service down → saga retries then times out → compensation; notification down → order still confirmed.
- **Done when:** all scenarios automated and green.

---

## Phase 7 — Kafka Hardening & Transactional Outbox

### ☐ P7-T01 — Avro contracts & Schema Registry
- **How:** `.avsc` files in `event-schemas` with namespace/versioning; `avro-maven-plugin` generates classes; `KafkaAvroSerializer` with `auto.register.schemas=false` in non-local envs; compatibility BACKWARD set per subject; event envelope fields: eventId, eventType, occurredAt, aggregateId, correlationId, version.
- **Done when:** incompatible change fails CI (schema compatibility check plugin).

### ☐ P7-T02 — Debezium outbox (CDC)
- **How:** Kafka Connect container + Debezium Postgres connector with Outbox Event Router SMT reading each service's `outbox_event` table; remove polling relay (keep as documented fallback); logical replication slot monitoring.
- **Done when:** events flow with no app-side publishing code; replication slot lag visible.

### ☐ P7-T03 — Producer & consumer configuration
- **How:** Producer: `acks=all`, `enable.idempotence=true`, compression, linger/batch tuning. Consumer: manual ack / `AckMode.RECORD` vs batch, `max.poll.records`, `isolation.level=read_committed`, concurrency = partitions; partition count per topic documented; key = aggregate ID for ordering.
- **Done when:** config documented in KAFKA_EVENTS.md with reasoning per setting.

### ☐ P7-T04 — Idempotent consumers (Scenario #8)
- **How:** `processed_event(consumer_group, event_id)` with unique constraint written in the same DB transaction as the business change; duplicates skipped and counted in a metric.
- **Done when:** replaying a topic from offset 0 produces no duplicate side effects.

### ☐ P7-T05 — Retries, DLT, replay (Scenario #9)
- **How:** `@RetryableTopic` with backoff for transient errors; non-retryable exceptions (validation, deserialization) go straight to DLT via `ErrorHandlingDeserializer`; admin endpoint/CLI to inspect and replay DLT messages; alert on DLT growth.
- **Done when:** poison message test lands in DLT; consumer keeps processing others; replay works after fix.

### ☐ P7-T06 — Consumer crash recovery demo
- **How:** Testcontainers test: consumer processes, crashes before commit, restarts → message redelivered → idempotency prevents double effect. Document rebalance behaviour and cooperative-sticky assignor.
- **Done when:** test green; runbook written.

### ☐ P7-T07 — notification-service
- **How:** Consumes `OrderConfirmed`, `OrderCancelled`, `PaymentFailed`, `CustomerRegistered`; Thymeleaf email templates; sends via Spring Mail (Mailpit locally); `notification_log` with status and retry; never blocks the producer.
- **Done when:** completing an order shows an email in Mailpit.

### ☐ P7-T08 — Schema evolution exercise
- **How:** Add optional field `couponCode` to `OrderCreated` v2; old consumers keep working; new consumer reads it. Document the rules (add optional with default, never rename/remove required).
- **Done when:** mixed-version test passes.

---

## Phase 8 — File Management Service & Large Uploads

### ☐ P8-T01 — Storage abstraction
- **How:** Interface `ObjectStorage` (presignUpload, initiateMultipart, presignPart, completeMultipart, abort, presignDownload, delete) with `S3ObjectStorage` (AWS SDK v2, works with MinIO) and `AzureBlobObjectStorage` (block blobs + SAS, works with Azurite); selected by config.
- **Done when:** same integration test suite passes against MinIO and Azurite.

### ☐ P8-T02 — Pre-signed direct upload flow (Scenario #3)
- **How:** `POST /files/uploads` (validate declared size/type, create `upload_session`) → returns pre-signed URLs per part → client uploads directly to storage → `POST /files/uploads/{id}/complete` → verify ETags/sizes → status `UPLOADED` → event `FileUploaded`. App server never streams the bytes.
- **Done when:** 2 GB file uploads with service heap flat (check metrics).

### ☐ P8-T03 — Resumable uploads & progress
- **How:** `GET /files/uploads/{id}` returns completed parts (from storage `ListParts` / DB) so the client resumes missing parts only; progress = uploaded bytes / total; short URL expiry with re-issue endpoint.
- **Done when:** interrupted upload resumes without re-sending finished parts.

### ☐ P8-T04 — Validation & virus scanning
- **How:** Size limits per file category; extension allow-list; magic-byte MIME detection (Apache Tika) after upload; quarantine prefix until ClamAV (clamd) scan passes via async consumer; infected → deleted + event.
- **Done when:** EICAR test file is rejected; renamed `.exe` → `.jpg` rejected.

### ☐ P8-T05 — Product image processing
- **How:** On clean `FileUploaded` with purpose `PRODUCT_IMAGE` generate thumbnails (Thumbnailator) and publish `ProductImageReady` for catalog.
- **Done when:** catalog shows thumbnail URLs (pre-signed download).

### ☐ P8-T06 — Bulk product CSV/Excel import
- **How:** Stream-parse (OpenCSV / Apache POI streaming `XSSF` event model or FastExcel); validate per row; upsert in batches of 500 via catalog bulk API or event; job table with counts; produce error report file.
- **Done when:** 1M-row import completes with bounded memory; error report downloadable.

### ☐ P8-T07 — Async large Excel reports (Scenario #4)
- **How:** `POST /reports` → job queued (Kafka) → worker streams data with keyset pagination into `SXSSFWorkbook` (window 100 rows) → multipart upload to storage → notify user with pre-signed download link; status polling endpoint.
- **Done when:** 1M-row sales report generated with constant memory.

### ☐ P8-T08 — Cleanup policies
- **How:** Scheduled job aborts incomplete multipart uploads older than 24 h; storage lifecycle rules (S3 lifecycle / Blob lifecycle management) for temp prefixes; soft-delete metadata retention.
- **Done when:** abandoned uploads disappear after policy window (test with shortened window).

---

## Phase 9 — Elasticsearch Indexing & Advanced Search

### ☐ P9-T01 — Index design
- **How:** Index template `products-v1`: `text` fields with custom analyzer (lowercase, asciifolding, stemming, synonyms), `keyword` for facets, `search_as_you_type`/edge-ngram for autocomplete, numeric price, nested attributes; alias `products` → `products-v1`.
- **Done when:** template versioned in `infrastructure/docker/elasticsearch/`.

### ☐ P9-T02 — Event-driven indexing
- **How:** search-service consumes catalog, price, inventory, review-summary events → builds denormalised product document → bulk index with refresh interval tuning; idempotent via document version (external versioning).
- **Done when:** product change reflected in search within seconds.

### ☐ P9-T03 — Search API
- **How:** `GET /search?q=&category=&brand=&minPrice=&maxPrice=&rating=&sort=&page=`; multi-match with field boosts, filters in `filter` context (cached), aggregations for facets, highlighting, `search_after` for deep paging; autocomplete endpoint.
- **Done when:** facets counts correct; p95 measured on 1M docs.

### ☐ P9-T04 — Zero-downtime reindex
- **How:** Create `products-v2`, full reindex from catalog (bulk export API or Kafka replay), switch alias atomically, delete old index.
- **Done when:** search keeps answering during reindex.

### ☐ P9-T05 — Sync failure handling (Scenario #13)
- **How:** Failed bulk items → retry → DLT; nightly reconciliation compares catalog `updated_at`/checksums with ES and re-indexes drift; metric `search_index_drift_count`.
- **Done when:** with ES stopped during updates, data converges after ES returns.

### ☐ P9-T06 — Relevance tuning
- **How:** Judgement list of ~30 queries; tune boosts, `function_score` (rating, popularity, in-stock); measure with ES Ranking Evaluation API (nDCG).
- **Done when:** documented before/after nDCG.

---

## Phase 10 — AI Integration with Spring AI

### ☐ P10-T01 — LLM fundamentals notes
- **How:** Write `docs/learning/llm-fundamentals.md`: tokens, context window, temperature/top-p, system vs user prompts, structured output, embeddings, cost model per 1M tokens.
- **Done when:** you can explain each in your own words (gap tracker updated).

### ☐ P10-T02 — Provider-agnostic Spring AI setup
- **How:** ai-assistant-service with Spring AI `ChatClient`; profiles `ollama` (local, e.g. llama/qwen + `nomic-embed-text`), `anthropic`, `openai`/`azure-openai`; API keys from env only; Ollama added to Compose profile `ai`.
- **Done when:** a test prompt works with two providers by switching profile.

### ☐ P10-T03 — Prompt templates & structured output
- **How:** Prompts in `resources/prompts/*.st` versioned; `.entity(Record.class)` structured output with validation and retry on parse failure.
- **Done when:** invalid model output is caught and handled, never leaks to API.

### ☐ P10-T04 — AI product description generator
- **How:** Catalog admin requests draft → ai service generates description + SEO title/meta from attributes → stored as `DRAFT` → admin approves/edits → `PUBLISHED`. Never auto-publish.
- **Done when:** workflow enforced by state + role checks.

### ☐ P10-T05 — Cost, limits, resilience
- **How:** Micrometer metrics for prompt/completion tokens per feature & user; per-user daily token budget (Redis); timeouts, retry on 429/5xx, fallback to smaller model; response caching for identical prompts.
- **Done when:** Grafana-ready metrics exist; budget exceeded → friendly 429.

---

## Phase 11 — RAG Customer Support & Semantic Search

### ☐ P11-T01 — Document ingestion pipeline
- **How:** Admin uploads manuals/FAQs/policies via file-service → `FileUploaded(purpose=KNOWLEDGE_DOC)` → ai-assistant consumes → Tika/PDF reader → chunking (`TokenTextSplitter`, try 300/800 tokens with overlap) → embeddings → `PgVectorStore` with metadata (docId, title, section, page, version).
- **Done when:** re-uploading a doc replaces old chunks (no duplicates).

### ☐ P11-T02 — Retrieval & answer with citations
- **How:** `QuestionAnswerAdvisor`/custom retrieval: top-k + similarity threshold + metadata filter; prompt instructs "answer only from context, cite [n]"; response includes sources (doc title, page, link).
- **Done when:** answers show citations that open the right document.

### ☐ P11-T03 — No-context & hallucination handling
- **How:** If no chunk passes threshold → return "I couldn't find this in our policies" + handoff to human support; post-check citations exist in retrieved set.
- **Done when:** out-of-domain questions get the fallback, not a guess.

### ☐ P11-T04 — Chunking & retrieval experiments
- **How:** Compare chunk sizes, overlap, and HNSW vs IVFFlat index on a golden Q&A set; record recall@k.
- **Done when:** results table in AI_ARCHITECTURE.md; chosen settings justified.

### ☐ P11-T05 — Product embeddings & hybrid search
- **How:** search-service generates embeddings for product text on index (via ai service or Spring AI embedding model), stores in ES `dense_vector`; hybrid query = BM25 + kNN combined with RRF; natural-language queries ("warm jacket for hiking under 5000").
- **Done when:** hybrid beats keyword-only on the judgement list.

### ☐ P11-T06 — RAG evaluation harness
- **How:** Golden set (50 Q&A); automated metrics: retrieval hit-rate, answer relevance and faithfulness (LLM-as-judge with Spring AI `RelevancyEvaluator`/`FactCheckingEvaluator`); run in CI on demand.
- **Done when:** evaluation report generated and thresholds defined.

---

## Phase 12 — AI Shopping Assistant, Agent & Recommendations

### ☐ P12-T01 — Tool definitions
- **How:** `@Tool` methods: `searchProducts`, `getProductDetails`, `compareProducts`, `checkStock`, `addToCart`, `getOrderStatus`, `placeOrder`. Each calls the real service through the gateway/clients with the **user's** token. Strict input schemas, output trimming to save tokens.
- **Done when:** each tool unit-tested with mocked clients.

### ☐ P12-T02 — Authorization boundaries & confirmation
- **How:** Read tools allowed freely; write tools (`addToCart`) allowed with audit; sensitive tools (`placeOrder`) return a pending action that the user must confirm via a separate API call with a one-time confirmation ID — the model can never confirm on its own.
- **Done when:** test proves the model cannot place an order without explicit user confirmation.

### ☐ P12-T03 — Conversation memory & context management
- **How:** `ChatMemory` persisted in Postgres per conversation; window/summary strategy to cap tokens; conversation ownership checks.
- **Done when:** long conversation stays under token budget.

### ☐ P12-T04 — Prompt-injection protection
- **How:** System prompt hardening; treat tool outputs/RAG text as data (delimited); input classifier/guard for jailbreak patterns; tool allow-list per user role; output filters (PII, URLs); red-team test suite (≥ 20 attacks).
- **Done when:** red-team suite passes in CI.

### ☐ P12-T05 — ai-recommendation-service
- **How:** Consume order/view events → co-purchase matrix (item-to-item); embedding similarity for "related products"; customer preference vector from history; blend scores; LLM generates short "why recommended" from signals (cached).
- **Done when:** `/recommendations/me` and `/products/{id}/related` return explained results.

### ☐ P12-T06 — AI observability
- **How:** Spring AI observation (Micrometer) → traces per chat call & tool call, token counts, latency, model name; log prompts only with redaction; dashboard per feature.
- **Done when:** one assistant conversation is traceable end-to-end.

---

## Phase 13 — Automated Testing, Security Hardening, Performance

### ☐ P13-T01 — Test pyramid audit
- **How:** Coverage per service (jacoco), list untested business rules, add unit/repository/MockMvc/Kafka tests; shared `common-test` Testcontainers singletons with reuse for speed.
- **Done when:** every major feature has automated tests; coverage gate raised to 75–80 %.

### ☐ P13-T02 — Consumer-driven contracts
- **How:** Spring Cloud Contract (or Pact): cart→catalog, order→inventory REST contracts; messaging contracts for key events; stubs published to local repo used by consumers' tests.
- **Done when:** breaking a provider response fails the provider build.

### ☐ P13-T03 — End-to-end tests
- **How:** `tests/e2e` (JUnit + RestAssured) runs against Compose stack: register → login → search → cart → order → pay (Stripe test card) → email received (Mailpit API) → review.
- **Done when:** E2E suite runs green from a script.

### ☐ P13-T04 — Performance tests (Scenarios #2, #15, #18)
- **How:** k6/Gatling scenarios: browse/search mix, checkout, spike (10× in 30 s), soak (1 h); capture p50/p95/p99, error rate, saturation; find and fix top 3 bottlenecks (profiling with JFR/async-profiler).
- **Done when:** baseline + improvements documented.

### ☐ P13-T05 — Traffic spike handling
- **How:** Gateway rate limits, bulkheads, queue-based load levelling for heavy work, graceful degradation (disable recommendations under load via feature flag), HPA plan.
- **Done when:** spike test produces controlled 429s, not 500s/timeouts.

### ☐ P13-T06 — OWASP Top 10 hardening
- **How:** Checklist per category: access control (IDOR tests), crypto (TLS, no secrets in logs), injection (parameterised queries only, no string-built JPQL), insecure design, misconfig (actuator lock-down, security headers, CORS), vulnerable deps (Dependency-Check), auth failures (lockout in Keycloak), integrity (signed images later), logging (audit log), SSRF (URL allow-lists in file/AI fetches).
- **Done when:** SECURITY.md has each item with evidence.

### ☐ P13-T07 — Secure service-to-service calls (Scenario #14)
- **How:** Client-credentials tokens with audience per service for internal calls; network policies (later in K8s); optional mTLS (discuss service mesh).
- **Done when:** internal endpoint rejects calls without a valid service token.

### ☐ P13-T08 — Audit logging
- **How:** Audit events (login, admin changes, payment, order state) to an append-only table/topic with who/what/when/where; never log secrets/PII.
- **Done when:** admin product change is audit-traceable.

---

## Phase 14 — Observability, Monitoring, Centralized Logging

### ☐ P14-T01 — Metrics stack
- **How:** Compose profile `obs`: Prometheus (scrape `/actuator/prometheus`), Grafana provisioned dashboards (JVM, HTTP RED, Hikari, Kafka consumer lag via kafka-exporter, Redis, ES).
- **Done when:** dashboards load automatically on startup.

### ☐ P14-T02 — Distributed tracing
- **How:** Micrometer Tracing + OpenTelemetry exporter → OTel Collector → Tempo or Jaeger; trace context propagated through Kafka headers; correlation ID = trace ID in logs.
- **Done when:** one order shows a single trace spanning gateway, order, inventory, payment, notification.

### ☐ P14-T03 — Centralized logging
- **How:** JSON structured logs (Spring Boot structured logging / logstash encoder) → Filebeat/Fluent Bit → Elasticsearch/OpenSearch → Kibana/Dashboards; fields: service, traceId, spanId, userId (hashed), orderId.
- **Done when:** searching an orderId shows logs from all services.

### ☐ P14-T04 — Business metrics
- **How:** Counters/timers: orders created/confirmed/failed, payment failure rate, cart abandonment, AI tokens & cost, search zero-result rate.
- **Done when:** business dashboard exists.

### ☐ P14-T05 — Alerts & SLOs
- **How:** SLOs (checkout availability, search latency); Prometheus alert rules (error rate, latency, consumer lag, DLT size, pool usage > 80 %).
- **Done when:** alerts fire in a simulation.

### ☐ P14-T06 — Pool exhaustion drill (Scenario #10)
- **How:** Simulate slow query / leaked connection; observe Hikari metrics & alerts; fix with timeouts, pool sizing formula, leak detection, moving long work outside transactions; runbook.
- **Done when:** runbook in `docs/runbooks/` with screenshots/metrics.

---

## Phase 15 — CI/CD & Kubernetes Deployment

### ☐ P15-T01 — CI pipeline
- **How:** GitHub Actions: path-filtered matrix per service; steps: checkout → JDK 21 + Maven cache → `./mvnw -Pci verify` (unit + Testcontainers ITs) → Checkstyle/SpotBugs → SonarCloud/SonarQube → OWASP Dependency-Check → upload reports.
- **Done when:** PR checks required for merge.

### ☐ P15-T02 — Image build & scanning
- **How:** Build multi-arch images, tag with semver + git SHA, push to GHCR; Trivy scan (fail on HIGH/CRITICAL); SBOM generation; image signing (cosign) optional.
- **Done when:** images in registry with scan reports.

### ☐ P15-T03 — Helm chart
- **How:** Generic `spring-service` chart: Deployment (probes on `/actuator/health/liveness|readiness`, resources, securityContext non-root, read-only FS), Service, ConfigMap, Secret refs, HPA, PDB, Ingress, ServiceAccount; umbrella chart with values per env.
- **Done when:** `helm lint` + `helm template` pass in CI.

### ☐ P15-T04 — Local Kubernetes (kind)
- **How:** kind cluster + ingress-nginx; infra via Helm (Bitnami/official charts or Strimzi for Kafka); deploy all services; switch discovery to K8s DNS.
- **Done when:** full checkout works in kind.

### ☐ P15-T05 — Zero-downtime deployment (Scenario #17)
- **How:** RollingUpdate (`maxUnavailable: 0`), readiness gates, graceful shutdown (`server.shutdown=graceful`, preStop sleep), backward-compatible DB migrations (expand/contract), run k6 during rollout.
- **Done when:** zero failed requests during a rollout.

### ☐ P15-T06 — CD pipeline & environments
- **How:** On merge to `main` → deploy to `dev`; tag `v*` → manual approval → `prod`; environment-specific values & secrets via GitHub Environments; release notes from conventional commits (release-please / semantic-release).
- **Done when:** a tag produces a versioned release and deployment.

---

## Phase 16 — AWS Deployment

### ☐ P16-T01 — Cost guardrails first
- **How:** AWS Budgets alert (e.g. $20), Cost Explorer tags, `scripts/aws-destroy.sh`; free-tier review; plan to run the cluster only during sessions.
- **Done when:** budget alert active before creating resources.

### ☐ P16-T02 — Terraform foundations
- **How:** Remote state (S3 + lock), VPC (2 AZs, public/private subnets, NAT cost note — consider single NAT or VPC endpoints), ECR, IAM.
- **Done when:** `terraform plan/apply/destroy` cycle works.

### ☐ P16-T03 — EKS & data services
- **How:** EKS (managed node group, spot instances), RDS PostgreSQL (db.t4g.micro, single-AZ for dev), S3 buckets (private, encryption, lifecycle), Kafka via Strimzi in-cluster (document MSK Serverless alternative and cost), ElastiCache or in-cluster Redis, Elasticsearch in-cluster or Elastic Cloud trial.
- **Done when:** services reach RDS/S3 from pods.

### ☐ P16-T04 — Identity, secrets, ingress
- **How:** IRSA / EKS Pod Identity for S3 access (no static keys), Secrets Manager + External Secrets Operator, AWS Load Balancer Controller + ALB Ingress with ACM TLS.
- **Done when:** no AWS keys in any manifest; HTTPS endpoint live.

### ☐ P16-T05 — Observability & serverless extras
- **How:** CloudWatch Container Insights / logs; optional: Lambda triggered by S3 for thumbnails (compare with in-service processing), SES for email, SQS/SNS vs Kafka comparison note, API Gateway vs in-cluster gateway note.
- **Done when:** AWS_DEPLOYMENT.md complete with diagrams and cleanup steps.

### ☐ P16-T06 — Cloud cost optimization (Scenario #20)
- **How:** Right-size requests/limits, spot nodes, scale to zero off-hours, Graviton, storage tiers, log retention, NAT avoidance; compute monthly estimate.
- **Done when:** cost breakdown + optimisations documented.

---

## Phase 17 — Azure Deployment

### ☐ P17-T01 — Service mapping table
- **How:** AWS ↔ Azure table: EKS↔AKS, ECR↔ACR, S3↔Blob, RDS↔PostgreSQL Flexible Server, Secrets Manager↔Key Vault, CloudWatch↔Monitor/App Insights, IAM roles↔Managed Identity/Workload Identity, ALB↔App Gateway/ingress, MSK↔Event Hubs (Kafka API), SQS/SNS↔Service Bus, Lambda↔Functions, SES↔Communication Services.
- **Done when:** table in AZURE_DEPLOYMENT.md.

### ☐ P17-T02 — Terraform for Azure
- **How:** Resource group, VNet/subnets, AKS (spot node pool), ACR, PostgreSQL Flexible Server (Burstable B1ms), Storage account, Key Vault; budgets.
- **Done when:** apply/destroy cycle works.

### ☐ P17-T03 — Deploy with the same Helm charts
- **How:** `overlays/azure` values: storage provider = azure, Workload Identity for Blob/Key Vault (CSI secrets driver), Azure OpenAI profile optional.
- **Done when:** full checkout works on AKS with only config changes.

### ☐ P17-T04 — Azure monitoring
- **How:** Application Insights Java agent or OTel exporter → Azure Monitor; Log Analytics queries for order traces.
- **Done when:** one order traced in App Insights.

### ☐ P17-T05 — Service Bus comparison exercise
- **How:** Implement notification consumer variant on Service Bus (or Event Hubs Kafka endpoint) behind a messaging port; document when you'd choose each.
- **Done when:** comparison documented.

---

## Phase 18 — Production Readiness, Architecture Review, Optimization

### ☐ P18-T01 — Production readiness checklist
- **How:** Per service: health probes, timeouts everywhere, retries bounded, idempotency, limits, dashboards, alerts, runbooks, backups, security scans, ownership.
- **Done when:** checklist all green or exceptions justified.

### ☐ P18-T02 — Architecture review
- **How:** Revisit each ADR: still valid? What would you change at 10× scale? Would a modular monolith have been better? Write ADR updates.
- **Done when:** review doc written.

### ☐ P18-T03 — Performance & JVM tuning
- **How:** GC choice (G1 vs ZGC), heap/container sizing, virtual threads where blocking I/O dominates, query plans, cache TTLs; compare before/after load tests.
- **Done when:** measurable improvements recorded.

### ☐ P18-T04 — Resilience & DR
- **How:** Chaos drills (kill pods, Kafka broker, DB failover), backup/restore test (PITR), RPO/RTO definitions; partial-failure behaviour matrix (Scenario #19).
- **Done when:** restore tested; matrix documented.

### ☐ P18-T05 — Final documentation
- **How:** Complete README, ARCHITECTURE, DATABASE_DESIGN, API_DOCUMENTATION, KAFKA_EVENTS, AI_ARCHITECTURE, SECURITY, LOCAL_SETUP, AWS_DEPLOYMENT, AZURE_DEPLOYMENT, TROUBLESHOOTING, all 20 scenario runbooks.
- **Done when:** a new engineer can run and understand the system from docs alone.

### ☐ P18-T06 — Interview readiness pack
- **How:** Per area: 10 conceptual + 5 scenario questions with your own answers referencing this codebase; system design walk-through (whiteboard version of this architecture); final gap tracker update.
- **Done when:** you can present the project in 15 min and handle deep-dive questions.

---

## Recurring tasks (every phase)

- ☐ Update `docs/learning/TECHNICAL_GAP_TRACKER.md` (covered, understood, needs practice, implemented, interview readiness, next learning).
- ☐ Add interview questions for the phase to `docs/learning/interview-questions.md`.
- ☐ Keep `./mvnw verify` green and the Compose stack runnable.
- ☐ Update relevant docs and ADRs.
- ☐ Tag the phase: `git tag phase-NN`.
