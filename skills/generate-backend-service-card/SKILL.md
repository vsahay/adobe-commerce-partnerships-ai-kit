# Generate Backend Service Card Skill — Generate Service Cards for a Backend Service

> **Purpose:** Produce a complete service card set for any backend microservice,
> sufficient for an agentic coding system to produce a good-enough LLD without
> reading the source repo.
>
> **Scope:** Document the codebase as it exists today. Do not suggest improvements,
> critique architecture, or propose changes unless explicitly asked.

---

## 1. When to Use

| Trigger | Example |
|---------|---------|
| Onboarding a new service | Service has no cards yet |
| Significant refactor | New auth framework, new connector layer |
| Quarterly refresh | Cards have drifted from codebase |
| Post-incident gap | Incident exposed a missing or wrong section |

---

## 2. Inputs

```
<source_repo_path>
```

| Input | Required | Notes |
|-------|----------|-------|
| `source_repo_path` | Yes | Path to the service repo cloned locally — this is the codebase to scan |

**If `source_repo_path` is not provided, stop immediately and ask:**
> "Please provide the path to the source repo you want to scan (e.g. `/path/to/my-service`)."

Do not proceed until the user supplies an explicit path.

**Derived automatically:**
- Service name — `basename(source_repo_path)`
- Source branch — `git -C <source_repo_path> rev-parse --abbrev-ref HEAD`
- Output directory — always `<source_repo_path>/docs/ai-kit/service-cards/backend/` — created if it does not exist

---

## 3. Outputs

All files written under `<source_repo_path>/docs/ai-kit/service-cards/backend/`.

### 3.1 Generated per-service

| File | What it captures                                                                                                                                                                                        |
|------|---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| `SERVICE_CARD.md` | Identity: what the service owns, who runs it, its external surface at a glance, fragile areas, recent changes. The file an engineer reads first.                                                        |
| `MODULE_INDEX.md` | Capability → implementation mapping: which classes, connectors, events, and flags implement each business capability. Includes Key Decisions per capability.                                            |
| `CONTRACTS.md` | The service's full API surface: HTTP operations it exposes and async events it consumes and publishes, with request/response shapes and auth. |
| `CONNECTORS.md` | Every outbound dependency the service calls: auth, transport config, resilience settings, error-handling posture. |
| `BUILD_CONFIG.md` | Technology choices and major dependency categories (e.g. HTTP client library, ORM, auth framework). Exact coordinates, property keys, and secret paths are captured here. |
| `CODE_PATTERNS.md` | Team coding conventions: naming, package layout, DTO patterns, error handling, validation, test approach. **Use during code generation**                                                                |
| `PLATFORM.md` | Deployment topology, monitoring, feature flags, caching, database connection, active migrations, runbooks.                                                                                              |
| `DB_SCHEMA.md` | Database owned by the service: entity-to-table mapping, columns, relationships, migrations. Omit if stateless.                                                                                          |

---

## 4. Process

### Phase 1 — Locate & Scaffold (~2 min)

Before reading any file, build a map of the codebase. Scans in Phase 2 work from this map — no browsing.

**Step 1: Stack detection**

LS the repo root first, then read whichever manifest file is present.

**Build tool** — identify by manifest filename found at root:

| Manifest file | Build tool | Language |
|---------------|------------|----------|
| `pom.xml` | Maven | Java / Kotlin |
| `build.gradle` / `build.gradle.kts` | Gradle | Java / Kotlin |
| `build.xml` | Ant | Java |
| `package.json` | npm / yarn / pnpm | Node / TypeScript |
| `pyproject.toml` | Poetry / Hatch / Flit | Python |
| `setup.py` / `setup.cfg` | setuptools | Python |
| `go.mod` | Go modules | Go |

If no manifest is found at root, LS one level deeper — monorepos may nest manifests per module.

**Language + runtime version** — where to look per language:

| Language | Version source |
|----------|---------------|
| Java / Kotlin | `<java.version>` or `sourceCompatibility` in manifest; `.java-version`; `.tool-versions` |
| Node / TS | `engines.node` in `package.json`; `.nvmrc`; `.node-version` |
| Python | `python_requires` in `pyproject.toml` / `setup.cfg`; `.python-version`; `.tool-versions` |
| Go | `go` directive in `go.mod` |
| Any | CI config files (`.github/workflows/`, `Jenkinsfile`, `.gitlab-ci.yml`) — runtime version is often pinned there |

**Web framework** — read from dependencies in the manifest:

| Language | Common frameworks |
|----------|------------------|
| Java / Kotlin | Spring Boot, Quarkus, Micronaut, Vert.x, Javalin |
| Node / TS | Express, Fastify, NestJS, Koa, Hono |
| Python | FastAPI, Flask, Django, Starlette |
| Go | Gin, Echo, Chi, standard `net/http` |

**ORM / persistence** — read from dependencies:

| Case | What to record |
|------|---------------|
| ORM present | Library name and version (e.g. Hibernate 6.x, SQLAlchemy 2.x, Prisma, GORM) |
| Query builder present (no full ORM) | Library name (e.g. jOOQ, Knex, Dapper, sqlx) — note "query builder, no entity mapping" |
| Raw SQL / no library | Note "raw SQL — no ORM or query builder detected"; Phase 2.10 will derive schema from migration files |

**Test framework** — read from dev/test dependencies. If absent, note `none detected`.

---

**Resolution — classify each field before proceeding:**

| Status | Meaning | Action |
|--------|---------|--------|
| `CONFIRMED` | Detected unambiguously from a manifest or config file | Record and continue |
| `INFERRED` | Best guess from indirect evidence (e.g. framework inferred from CI config, version inferred from source compatibility) | Record with a note — e.g. `Spring Boot (inferred from CI)` |
| `AMBIGUOUS` | Two or more equally likely values detected | Stop and ask (see prompts below) |
| `NOT FOUND` | No evidence found after exhausting all sources | Stop and ask (see prompts below) |

**If any field is `AMBIGUOUS` or `NOT FOUND`, stop immediately and ask the user before reading any more files.** Batch all unresolved fields into a single message:

> "I couldn't confidently detect the following from the repo. Please confirm:
>
> - **Build tool / language:** [what was found or nothing] — e.g. Maven, Gradle, npm, go.mod?
> - **Runtime version:** [what was found or nothing] — e.g. Java 17, Node 20, Python 3.11?
> - **Web framework:** [what was found or nothing] — e.g. Spring Boot, Express, FastAPI?
> - **ORM / persistence:** [what was found or nothing] — e.g. Hibernate, Prisma, raw SQL, none?
>
> *(Only list the fields that are unresolved — omit confirmed fields.)*"

Do not proceed to Step 2 until every field is resolved.

---

Record all five fields in `BUILD_CONFIG.md → ## 1. Runtime Specifications`. Stack drives layer names, file patterns, and grep idioms for every scan that follows.

**Step 2: Codebase map**

Run a structured search before reading any implementation file. Goal: know where things live before reading what they say.

```
Search order:

1. LS top-level source directories — identify the layer structure

2. Grep for entry-point markers using the framework detected in Step 1:

   HTTP handlers —
   Java/Kotlin:   @RestController, @RequestMapping, @GetMapping, @FeignClient, @Entity
   Node/TS:       router\., app\.(get|post|put|delete|patch), axios\.create, prisma\.
   Python:        @app\.route, @router\., httpx\.Client, requests\.Session, Base\s*=
   Go:            http\.HandleFunc, http\.NewServeMux, r\.GET, grpc\.NewClient
   Fallback:      ("GET"|"POST"|"PUT"|"DELETE") combined with ("/api/"|"/v[0-9]")

   Async consumers/publishers —
   Java/Kotlin:   @KafkaListener, @RabbitListener, @SqsListener, @JmsListener, kafkaTemplate\.send, rabbitTemplate\.send
   Node/TS:       consumer\.subscribe, channel\.consume, sqs\.receiveMessage, producer\.send, sns\.publish
   Python:        consumer\.subscribe, channel\.basic_consume, boto3.*sqs, producer\.send, boto3.*sns.*publish
   Go:            consumer\.ConsumePartition, ch\.Consume, subscription\.Receive, producer\.Produce, sns\.Publish

3. Glob for naming patterns (language-agnostic):
   *controller*, *handler*, *router*               → inbound surface
   *client*, *connector*, *adapter*                → outbound connectors
   *service*, *usecase*, *manager*                 → business logic
   *repository*, *dao*, *store*                    → persistence access
   *entity*, *model*, *schema*                     → persistence / domain
   *dto*, *request*, *response*, *payload*         → data transfer objects
   *listener*, *consumer*, *subscriber*, *event*   → async events
   *middleware*, *interceptor*, *filter*            → cross-cutting concerns
   *test*, *spec*                                  → test files

4. Note clusters — directories with 3+ related files are a layer, not stragglers.
   Exception: async consumer/publisher files count as a cluster even if only 1–2 files are found — a single listener is still a meaningful layer.
```

Produce a **codebase map** grouping files by purpose, with full paths and file counts. Example:

```
## Codebase Map — <service-name>

### Entry-point handlers
- `src/controllers/OrderController.java`
- `src/controllers/SubscriptionController.java`

### Outbound connectors
- `src/clients/PaymentClient.java`
- `src/clients/InventoryClient.java`
- `src/clients/NotificationClient.java`

### Business logic
- `src/services/` — 6 files

### Persistence access
- `src/repository/OrderRepository.java`
- `src/repository/SubscriptionRepository.java`

### Domain / DTOs
- `src/model/` — 8 files
- `src/dto/` — 6 files (request/response shapes)

### Async events
- `src/listeners/OrderEventListener.java`

### Cross-cutting
- `src/interceptors/AuthInterceptor.java`

### Persistence entities
- `src/entity/` — 4 files

### Configuration
- `src/config/` — 3 files
- `src/resources/application.yml`
- `src/resources/application-prod.yml`

### Tests
- `src/test/` — 18 files (unit: 14, integration: 4)

### Entry points (module registration)
- `src/Application.java` — main class, component scan root
- `src/config/BeanConfig.java` — client beans registered here
```

**Step 3: Scaffold output files**

Create empty card files for the 8 generated cards listed in [§3.1 Generated per-service](#31-generated-per-service).

---

### Phase 2 — Code Scan (~5–10 min)

Work from the codebase map. Read only the files the map identified.

**Posture:** Document what exists. If a connector has no retry config, write `none configured` — not "missing retry logic". Do not flag gaps as problems.

---

#### 2.1 Source structure
**Files:** Top-level source directories (already in codebase map — no new search).  
**Extract:** Layer taxonomy with file counts per layer.  
**Populates:** `SERVICE_CARD.md → ## Source Structure`.

---

#### 2.2 Inbound HTTP surface
**Files:** Entry-point handlers cluster.  
**Refine with grep** — use the pattern matching the detected framework:

| Framework | Grep pattern |
|-----------|-------------|
| Spring (Java/Kotlin) | `@GetMapping\|@PostMapping\|@PutMapping\|@DeleteMapping\|@PatchMapping\|@RequestMapping` |
| Node/TS (Express, Fastify, NestJS) | `router\.(get\|post\|put\|delete\|patch)\|app\.(get\|post\|put\|delete\|patch)\|@Get\|@Post\|@Put\|@Delete` |
| Python (FastAPI, Flask, Django) | `@app\.route\|@router\.(get\|post\|put\|delete)\|path\(\|re_path\(` |
| Go | `http\.HandleFunc\|\.HandleFunc\|r\.(GET\|POST\|PUT\|DELETE)\|\.Methods(` |
| Fallback | `"GET"\|"POST"\|"PUT"\|"DELETE"` near path patterns `"/api\|"/v[0-9]` |
**Extract per operation:** HTTP method, path, auth requirement, parameter bindings, delegate target (which service/usecase it calls).  
**Populates:** `CONTRACTS.md` (one `### <METHOD> /<path> — <Purpose>` per operation), `SERVICE_CARD.md → ## 3. Contracts → ### APIs We Expose` (one row per operation: Endpoint linked to `CONTRACTS.md#<anchor>`, Method, Purpose, Version, Stability), `MODULE_INDEX.md → ## Capability Catalogue`.

---

#### 2.3 Outbound connectors ⭐
**Files:** Outbound connectors cluster.  
**Refine with grep** — use the pattern matching the detected framework:

| Framework | Grep pattern |
|-----------|-------------|
| Spring (Java/Kotlin) | `@FeignClient\|RestTemplate\|WebClient\|RestClient\|new.*Client(` |
| Node/TS | `axios\.create\|new.*Client\|fetch(\|got\.\|needle\.\|superagent\.` |
| Python | `httpx\.Client\|requests\.Session\|aiohttp\.ClientSession\|urllib\.request` |
| Go | `http\.NewRequest\|http\.DefaultClient\|grpc\.NewClient\|resty\.New` |
| Fallback | Grep for base URL assignment patterns: `base_url\|BASE_URL\|baseUrl\|base_uri` |
**Extract per connector:**

| Field | What to capture |
|-------|----------------|
| Auth mechanism | API key / OAuth client credentials / mTLS / bearer token — and where the credential is sourced |
| Base URL source | Config property name or env var that holds it |
| Connect timeout | Value and unit — or `none configured` |
| Read/write timeout | Value and unit — or `none configured` |
| Retry config | Max attempts, backoff strategy — or `none configured` |
| Error handling | Which HTTP status codes are retried, which throw, which return a fallback |
| Unavailability posture | What the service does when this connector is down: fail-fast, fallback value, circuit breaker |
| SDK / library | e.g. Feign, Axios, HTTPX, net/http |

**Also find:** Where each connector bean/instance is registered (entry points cluster → config files).  
**Populates:** `CONNECTORS.md` (one `### <N>. <ConnectorName> (<one-line desc>)` per connector), `SERVICE_CARD.md → ## 3. Contracts → ### APIs We Consume` (one row per connector: Service linked to `CONNECTORS.md#<anchor>`, Connector Interface, Connector Impl, Auth Type, Purpose), `MODULE_INDEX.md → ## Capability Catalogue`.

**Connector Interface / Connector Impl — how to fill by stack:**

| Stack | Connector Interface | Connector Impl |
|-------|--------------------|--------------------|
| Java/Kotlin (Feign) | Feign `@FeignClient` interface name (e.g. `PaymentClient`) | Spring bean class that implements it (e.g. `PaymentClientImpl`) |
| Java/Kotlin (RestTemplate / WebClient) | Class name of the wrapper (e.g. `PaymentConnector`) | Same class — write `n/a (no interface)` |
| Node/TS | Module file that owns the calls (e.g. `paymentConnector.ts`) | Exported function(s) that make the HTTP call (e.g. `getInvoices, createCharge`) |
| Python | Module file (e.g. `payment_client.py`) | Function or class method (e.g. `PaymentClient.get_invoices`) |
| Go | Go interface type (e.g. `PaymentClient`) | Struct that implements it (e.g. `httpPaymentClient`) — write `n/a (no interface)` if none |

> **This is the most important scan.** A backend service that exists to call external services is fully described by its connector layer. Every connector must be completely documented — a code-gen run that has to guess at a connector's auth or error posture will produce incorrect code.

---

#### 2.4 Async consumers and publishers
**Files:** Async consumer and publisher cluster (if any).

**Consumers — refine with grep:**

| Framework | Grep pattern |
|-----------|-------------|
| Spring (Java/Kotlin) | `@KafkaListener\|@RabbitListener\|@SqsListener\|@JmsListener\|@EventListener` |
| Node/TS | `consumer\.subscribe\|channel\.consume\|sqs\.receiveMessage\|pubsub\.subscribe` |
| Python | `@app\.task\|consumer\.subscribe\|channel\.basic_consume\|boto3.*sqs` |
| Go | `consumer\.ConsumePartition\|ch\.Consume\|subscription\.Receive\|queue\.Consumer` |
| Fallback | Grep for queue/topic name patterns: `queue_name\|topic_name\|QUEUE_URL\|TOPIC_ARN` |

**Extract (consumers):** Topic/queue name, deserialization approach, ack posture, error handling, DLQ config.  
**Populates:** `CONTRACTS.md` Events Consumed section (one `#### <topic_name>` per topic), `SERVICE_CARD.md → ## 3. Contracts → ### Events We Consume` (one row per topic: Topic linked to `CONTRACTS.md#<anchor>`, Source Service, Consumer Class, Purpose), `MODULE_INDEX.md → ## Capability Catalogue` (Events row).

**Publishers — refine with grep:**

| Framework | Grep pattern |
|-----------|-------------|
| Spring (Java/Kotlin) | `kafkaTemplate\.send\|rabbitTemplate\.send\|snsClient\.publish\|sqsTemplate\.send\|applicationEventPublisher\.publish` |
| Node/TS | `producer\.send\|channel\.publish\|sns\.publish\|sqs\.sendMessage\|pubsub\.topic` |
| Python | `producer\.send\|boto3.*sns.*publish\|boto3.*sqs.*send_message\|celery.*apply_async` |
| Go | `producer\.Produce\|channel\.Publish\|sns\.Publish\|sqs\.SendMessage` |
| Fallback | Grep for publish/emit patterns: `publish\|emit\|send.*topic\|send.*queue` near topic/queue name strings |

**Extract (publishers):** Topic/queue name, serialization approach, triggering condition, publisher class/function.  
**Populates:** `CONTRACTS.md` Events Published section (one `#### <topic_name>` per topic), `SERVICE_CARD.md → ## 3. Contracts → ### Events We Publish` (one row per topic: Topic linked to `CONTRACTS.md#<anchor>`, Consumer Service(s), Publisher Class, Purpose), `MODULE_INDEX.md → ## Capability Catalogue` (Events row).

**Skip if:** No async files found — omit `### Events We Consume` and `### Events We Publish` from `SERVICE_CARD.md → ## 3. Contracts` and the Async section from `CONTRACTS.md` entirely.

---

#### 2.5 Domain and DTO layer
**Files:** Domain/DTOs cluster.  
**Extract:** Request/response shapes, value objects, domain types not mapped to persistence.  
**Populates:** `MODULE_INDEX.md → ## Capability Catalogue` (Domain row per capability), `MODULE_INDEX.md → ## Type Inventory`.

---

#### 2.6 Error handling
**Files:** Exception hierarchy, error code enumerations, global exception handlers.  
**Refine with grep** — use the pattern matching the detected framework:

| Framework | Grep pattern |
|-----------|-------------|
| Spring (Java/Kotlin) | `@ControllerAdvice\|@ExceptionHandler\|extends.*Exception\|enum.*Error` |
| Node/TS | `class.*Error extends\|errorHandler\|app\.use.*err\|middleware.*error` |
| Python | `class.*Exception\|class.*Error\|exception_handler\|@app\.exception_handler` |
| Go | `type.*Error struct\|errors\.New\|fmt\.Errorf\|func.*error\b` |
| Fallback | Grep for error code patterns: `ERROR_CODE\|error_code\|HttpStatus\|StatusCode` |
**Extract:** Error type hierarchy, error codes, how errors are serialised in responses.  
**Populates:** `CODE_PATTERNS.md`.

---

#### 2.7 Validation
**Files:** Validator classes, schema validators, input guards.  
**Refine with grep** — use the pattern matching the detected framework:

| Framework | Grep pattern |
|-----------|-------------|
| Spring (Java/Kotlin) | `@Valid\|@Validated\|@NotNull\|@NotBlank\|@Size\|ConstraintValidator` |
| Node/TS | `Joi\.object\|zod\.\|yup\.\|class-validator\|@IsString\|@IsNotEmpty` |
| Python | `pydantic\.\|BaseModel\|validator\|@field_validator\|marshmallow` |
| Go | `validate\.Struct\|binding:"required"\|govalidator\|ozzo-validation` |
| Fallback | Grep for validation boundary patterns: `validate\|sanitize\|guard\|assert` near handler entry points |
**Extract:** Validation library, where validation is invoked (handler layer vs service layer).  
**Populates:** `CODE_PATTERNS.md`, `MODULE_INDEX.md → ## Capability Catalogue` (Validation row per capability).

---

#### 2.8 Cross-cutting concerns
**Files:** Cross-cutting cluster (middleware, interceptors, filters identified in Step 2).  
**Skip if:** No cross-cutting cluster found — omit this section entirely.  
**Refine with grep** — use the pattern matching the detected framework:

| Framework | Grep pattern |
|-----------|-------------|
| Spring (Java/Kotlin) | `implements.*Filter\|extends.*HandlerInterceptor\|@Aspect\|@Around\|@Before\|@After\|OncePerRequestFilter` |
| Node/TS | `app\.use(\|router\.use(\|express\..*middleware\|fastify\.addHook` |
| Python | `@app\.middleware\|BaseHTTPMiddleware\|dispatch.*request\|process_request\|process_response` |
| Go | `func.*middleware\|mux\.Use\|chi\.Use\|gin\.Use\|r\.Use(` |
| Fallback | Grep for `middleware\|interceptor\|filter` near `request\|response\|context` |

**Extract per component:**

| Field | What to capture |
|-------|----------------|
| Name | Class / function name |
| Purpose | One line — what it does (e.g. auth check, correlation ID injection, request logging) |
| Scope | Global (all routes) or route-specific — note which routes if scoped |
| Reads | What it reads from the request (headers, tokens, session) |
| Writes | What it adds or modifies (e.g. sets auth context, injects MDC fields) |
| Order | Execution position if deterministic (e.g. runs before auth, after logging) |

**Populates:** `CODE_PATTERNS.md → ## Cross-cutting Concerns` (one subsection per component), `MODULE_INDEX.md → ## Capability Catalogue` (note which interceptors apply to each capability's handler).

---

#### 2.9 Test patterns
**Files:** 3–4 representative test files from the test cluster (one unit, one integration if present).  
**Extract:** Framework, mocking library, naming convention, how connectors are mocked.  
**Populates:** `CODE_PATTERNS.md` Test section.

---

#### 2.10 Persistence entities
**Files:** Persistence entities cluster.  
**Refine with grep** — use the ORM/persistence type detected in Step 1:

| ORM / persistence type | Grep pattern |
|------------------------|-------------|
| JPA / Hibernate (Java/Kotlin) | `@Entity\|@Table\|@Column\|@ManyToOne\|@OneToMany` |
| SQLAlchemy (Python) | `Base\s*=\|DeclarativeBase\|__tablename__\|Column(` |
| Prisma (Node/TS) | `model\s+\w+\s*{\|@@map\|@map\|@relation` |
| TypeORM (Node/TS) | `@Entity\|@Column\|@ManyToOne\|@Table\(` |
| GORM (Go) | `gorm\.Model\|type.*struct.*{\|gorm:"` |
| Migration DDL only | Parse migration files — grep for `CREATE TABLE\|ALTER TABLE\|add_column` |
| Fallback | Grep for table/collection name patterns: `table_name\|collection_name\|TABLE_NAME` |
**Extract:** Entity name → table name, columns, relationships, embedded types.  
**Populates:** `DB_SCHEMA.md → ## 3. Entity → Table Mapping`, `## 4. Tables`, `## 5. Relationships`.  
**Skip if:** No persistence cluster — write "this service owns no data store" in `DB_SCHEMA.md → ## 1. Overview` and stop.

---

#### 2.11 Key decisions (per capability)
**Files:** Primary logic methods for each capability identified in 2.2.  
**Extract per capability:**
- System-of-record claim (does this service own this data, or is it a pass-through?)
- Idempotency posture (is the operation safe to retry?)
- Fail-open vs fail-closed (when a connector is down, does the operation proceed or halt?)
- Stability posture (STABLE / IN FLUX)

For each decision, emit an `**Enforcement:**` citation naming the guard/annotation/middleware that enforces it. If none exists, write `not enforced in code — relies on caller discipline`.  
**Populates:** `MODULE_INDEX.md → ## Capability Catalogue` (Key Decisions row per capability).

---

#### 2.12 Handler scope and call graphs
**Files:** Each route-handler class (identified in 2.2).  
**Extract per handler:**
- `**Scope:**` — one sentence: what routes live here and what does not.
- `**Call graph:**` — for capabilities with >1 downstream call: 3–8 numbered steps naming method and downstream at each step.

**Populates:** `MODULE_INDEX.md → ## Capability Catalogue`.

---

### Phase 3 — Config Scan (~2–3 min)

Work from the configuration files identified in the codebase map. No new searching needed.

| Scan | Read | Populates |
|------|------|-----------|
| 3.1 Dependency manifest | Already identified in Step 1 — `pom.xml` / `build.gradle` / `package.json` / `pyproject.toml` / `go.mod`. Fallback: whichever manifest was found at root | `BUILD_CONFIG.md` — coordinates, deps, versions |
| 3.2 Runtime config | Java/Kotlin: `application.yml` / `application.properties`. Node/TS: `config/` directory / `.env`. Python: `config.py` / `settings.py`. Go: `config.yml` / `.env`. Fallback: glob root for `*.yml`, `*.yaml`, `*.toml`, `*.json`, `*.properties`, `*.env*` excluding test and build directories | `BUILD_CONFIG.md` — all keys, env-var bindings, stage overrides |
| 3.3 Deployment spec | LS `deploy/`, `infra/`, `ops/`, `k8s/` for deployment-related files. Common formats: k8s manifests (`*.yaml`), Terraform (`*.tf`), CDK / Pulumi, Helm charts (`Chart.yaml`). If none found, note `NOT CAPTURED — no deployment spec located` | `PLATFORM.md` — topology, scaling, secrets, team |
| 3.4 Persistence config | Grep for DB connection settings: `DB_URL\|DATABASE_URL\|jdbc:\|datasource\|db_host`. Migration tool config: `liquibase.properties` / `flyway.conf` / `alembic.ini` / `db/migrate/` / `prisma/migrations/`. Fallback: grep for `migrate\|migration` filenames | `DB_SCHEMA.md → ## 2. Connection & Deployment`, `BUILD_CONFIG.md → ## 4. Secrets & Sensitive Data` |

---

### Phase 4 — Assembly (~5–8 min)

1. Compile all scan outputs into the 8 generated card files.

2. **Formatting rules:**

| Rule | Detail |
|------|--------|
| Tables over prose | Use tables wherever content is list-like — cards are machine-read as well as human-read |
| Code snippets | 3–5 lines max — show the idiom, not a full implementation |
| Missing data | Write `NOT CAPTURED — requires [what]` — no placeholder text |
| Cross-references | Anchor links only — no "see X.md" pointers |

3. **Heading patterns** (keeps anchors stable and machine-resolvable):

| Card | Heading pattern |
|------|----------------|
| `CONTRACTS.md` | `### <METHOD> /<path> — <Purpose>` per REST operation; `#### <topic_name>` per async event |
| `CONNECTORS.md` | `### <N>. <ConnectorName> — <one-line desc>` per connector |
| `DB_SCHEMA.md` | `### <table_name>` per table — must match physical table name verbatim |

4. **`SERVICE_CARD.md → ## 3. Contracts` structure** — four fixed subsections, never merged into a single table:

| Subsection | Columns | Source scan | Omit if |
|-----------|---------|-------------|---------|
| `### APIs We Expose` | Endpoint, Method, Purpose, Version, Stability | 2.2 | Never |
| `### APIs We Consume` | Service, Connector Interface, Connector Impl, Auth Type, Purpose | 2.3 | Never |
| `### Events We Consume` | Event/Topic, Source Service, Consumer Class, Purpose | 2.4 | No async consumers |
| `### Events We Publish` | Event/Topic, Consumer Service(s), Publisher Class, Purpose | 2.4 | No async publishers |

5. **`SERVICE_CARD.md → ## Companion Files`** — always the last section. Fixed table — same for every backend service. Omit the `DB_SCHEMA.md` row if the service owns no data store.

| File | When to Load | Purpose |
|------|-------------|---------|
| `MODULE_INDEX.md` | Impact analysis, feature planning, code generation (load FIRST) | Capability → implementation mapping: which classes, connectors, events, and flags implement each business capability, with Key Decisions per capability. |
| `CONTRACTS.md` | Feature planning, code generation | The service's full API surface: HTTP operations it exposes and async events it consumes and publishes, with request/response shapes and auth. |
| `CONNECTORS.md` | Feature planning, code generation | Every outbound dependency: auth, transport config, resilience settings, and error-handling posture. |
| `BUILD_CONFIG.md` | Code generation | Runtime stack, dependencies, environment variables, and secrets. |
| `CODE_PATTERNS.md` | Code generation | Team coding conventions: naming, package layout, error handling, validation, test approach. |
| `PLATFORM.md` | Impact analysis, deployment planning | Deployment topology, monitoring, feature flags, caching, database, active migrations, runbooks. |
| `DB_SCHEMA.md` | Impact analysis, code generation (anything touching persisted state) | The database this service owns: entity mapping, tables, relationships, and migrations. |

6. **`MODULE_INDEX.md` required sections** — generate in this order:

| Section | Content |
|---------|---------|
| `## Overview` | One paragraph: what this service owns, its role in the system |
| `## Source Structure` | Table: Layer \| Path \| File count (from scan 2.1) |
| `## Capability List` | Table: Capability \| Primary Handler \| Stability |
| `## Capability Catalogue` | One `### <CapabilityName>` per capability — Handler, Services, Connector(s), Domain, Validation, Events, Key Decisions (with Enforcement), Call graph |
| `## Cross-Reference: Connector → Capabilities` | Table: Connector \| Capabilities that use it |
| `## Cross-Reference: Event → Capabilities` | Table: Topic \| Direction \| Capabilities — omit if no async events |
| `## Type Inventory` | Table: Type name \| Kind (Request/Response/Domain/Entity) \| Used by capability (from scan 2.5) |

7. **Before writing files** — run the [§5 Quality Gates](#5-quality-gates) checklist.

8. **Write `CHANGELOG.md` entry:**
   ```markdown
   ## YYYY-MM-DD — Initial card creation
   - Source: <repo> at commit <hash>
   - Scans completed: [list]
   ```

---

## 5. Quality Gates

Before closing the run, verify:

**Coverage**
- [ ] No section left as "TBD" — `NOT CAPTURED — requires [what]` is the minimum

**Completeness — capabilities**
- [ ] Every capability in `MODULE_INDEX ## Capability Catalogue` has: handler, service(s), connector(s), domain, validation, Key Decisions with Enforcement citation
- [ ] Every capability with >1 downstream call has a **Call graph** bullet

**Completeness — connectors**
- [ ] Every connector documented: auth mechanism, base URL source, timeout config, retry config, error posture, unavailability posture

**Completeness — cards**
- [ ] `BUILD_CONFIG.md` has all deps + properties needed to understand the service's build and runtime context
- [ ] `CODE_PATTERNS.md` has test framework, mocking approach, naming convention
- [ ] `DB_SCHEMA.md` complete, or explicitly marked out-of-scope with the no-data-store banner

**Cross-references**
- [ ] Every `SERVICE_CARD.md → ## 3. Contracts` row has a working anchor link

---
