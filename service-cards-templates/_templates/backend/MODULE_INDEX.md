# Module Index — {{SERVICE_NAME}}

> **Authoritative capability → code resolver.** Impact Analysis, LLD generation, and code
> generation all start here. Every class written for this service must trace back
> to a ## Capability Catalogue row.
>
> **Last Updated:** {{DATE}}

---

## 1. Workflow — How Each Pipeline Step Uses This File

### 1.1 Pipeline Entry Points

| Pipeline Step   | Sections Read | What it Extracts |
|-----------------|---------------|------------------|
| Impact Analysis | ## Capability Taxonomy, ## Capability Catalogue, ## Cross-Reference Indexes | Which capabilities are affected; blast radius |
| LLD generation  | ## Module Taxonomy, ## Capability Catalogue, ## Cross-Reference Indexes, ## Type Inventory | Package paths per new file; cross-ref anchors; type inventory |
| Code Generation | ## Module Taxonomy, ## Capability Catalogue, ## Type Inventory, ## Forbidden/Do-Not-Add Rules | Package + class naming; Key Decisions (idempotency, retry, SoR, fail-open/closed) |

### 1.2 Resolution Order — from "I need to add X" to "here is the package path"

1. Identify the capability from ## Capability Taxonomy
2. Jump to its ## Capability Catalogue row
3. Read the **Key Decisions** first — this is the architecture envelope (scan this first)
4. Read Controller / Service / Connector / Domain / Response mapper / Validation / Events / Flags
5. Follow the CONTRACTS.md / CONNECTORS.md / PLATFORM.md anchors for concrete shape
6. Only then look at code

---

## 2. Module Taxonomy — Layer Categories

<!--
Fill {{SOURCE_ROOT}} and location patterns with the actual paths for this project.

Examples by stack:
  Java/Spring:  source root = src/main/java/com/example/service/
                Controller → .controller[.internal]  Service → .services.impl  Connector → .connectors.impl
  Node/TS:      source root = src/
                Controller → routes/ or controllers/  Service → services/  Connector → clients/ or connectors/
  Python:       source root = app/
                Controller → routers/ or views/  Service → services/  Connector → clients/
  Go:           source root = internal/
                Controller → handler/  Service → service/  Connector → client/
-->

Source root: `{{SOURCE_ROOT}}`

| Layer | Location pattern | Responsibility | May depend on |
|-------|-----------------|---------------|---------------|
| Controller / Handler | `{{e.g., .controller / routes/ / routers/ / handler/}}` | HTTP entry, auth enforcement, request binding, correlation ID propagation | Service, Domain DTOs |
| Service | `{{e.g., .services.impl / services/ / service/}}` | Orchestration, business rules | Connector, Domain, Validator |
| Connector / Client | `{{e.g., .connectors.impl / clients/ / connectors/ / client/}}` | Downstream I/O, retry, timeout, error classification | Domain DTOs, config |
| Domain / Model | `{{e.g., .domain.* / models/ / types/ / model/}}` | Pure DTOs, value objects | (none) |
| Response Assembly | `{{e.g., .resource.* / dto/ / assembler/ — omit if done inline}}` | Response shaping / assembler targets | Domain |
| Validator | `{{e.g., .validator / validators/ / middleware/}}` | Request validation | Domain, constants |
| Config | `{{e.g., .config / config/}}` | Framework wiring, properties binding, dependency registration | (platform libraries) |
| Consumer / Subscriber | `{{e.g., .consumers / consumers/ / subscribers/}}` | Async event consumption (Kafka, SQS, RabbitMQ, etc.) | Service |

**Forbidden:** Controller → Controller, Service → Controller, Connector → Service,
Domain → any non-Domain layer.

See [SERVICE_CARD.md → ## What We Explicitly Do Not Do](SERVICE_CARD.md#2-what-we-explicitly-do-not-do) for service-level
boundaries.

---

## 3. Capability Taxonomy

Capabilities are grouped by domain. Each capability has a row in ## Capability Catalogue.

| Domain | Capabilities |
|--------|---------------------|
| {{Domain1}} | {{CapabilityName1}}, {{CapabilityName2}} |
| {{Domain2}} | {{CapabilityName3}}, {{CapabilityName4}} |

---

## 4. Capability Catalogue

> Every capability row MUST include **Key Decisions** — architectural choices (SoR
> boundaries, idempotency posture, retry/cache behaviour, fail-open vs fail-closed,
> stability) so readers don't need to re-derive them from code.

### 4.1 {{Capability Name}}

| Field | Value |
|-------|-------|
| Domain | {{Domain}} |
| Controller | `{{ControllerClass}}` → [CONTRACTS](CONTRACTS.md#<anchor>) |
| Service | `{{ServiceInterface}}` / `{{ServiceImpl}}` |
| Connector(s) | `{{ConnectorName}}` → [CONNECTORS](CONNECTORS.md#<anchor>) |
| Domain types | `{{pkg}}.{{DtoA}}`, `{{pkg}}.{{DtoB}}` |
| Response mapper | `{{pkg}}.{{MapperClass}}` *(omit if mapping done inline)* |
| Validation | `{{ValidatorClass}}` |
| Events | published: `{{topic}}` → [CONTRACTS (async)](CONTRACTS.md#<anchor>); consumed: — |
| Feature flags | `{{flag_name}}` *(omit if none)* |
| Notes | Any quirks, historical context |
| **Key Decisions** | {{SoR statement}}; {{idempotency posture}}; {{retry/cache behaviour}}; {{fail-open/closed}}; {{stability}} |
| **Call graph:** | {{NUMBERED_STEPS — 3–8 lines naming primary service method + downstreams, OR "single downstream — N/A"}} |
| **Enforcement:** | {{What enforces the Key Decision in code — e.g., Java: `@Idempotent(ttlDays=10) on FooController.bar`; Node/Python/Go: middleware function, guard expression, or decorator. OR honest "not enforced in code — relies on caller discipline / PR review"}} |
| **Scope:** | {{CONTROLLER_SCOPE — one-sentence single-responsibility statement for the controller row. What belongs here; what does NOT belong here. Example: "`OrderController` — order lifecycle operations (create, cancel, status). Not a catch-all; new order-adjacent features should land a purpose-specific controller if scope differs."}} |

<!-- Repeat for each capability -->

---

## 5. Cross-Reference Indexes

### 5.1 Endpoint → Capability

| Endpoint | Method | Capability |
|----------|--------|-----------|
| | | |

### 5.2 Connector → Capabilities

| Connector | Used by capabilities |
|-----------|---------------------|
| | |

### 5.3 Event Topic → Capabilities

| Topic | Role | Capabilities |
|-------|------|-------------|
| | consume / publish | |

### 5.4 Feature Flag → Capabilities

| Flag (source) | Capabilities gated |
|--------------|-------------------|
| | |

---

## 6. Cross-Cutting Modules

| Module | Purpose | Location |
|--------|---------|---------|
| Filters / interceptors | Auth, tracing, correlation ID, MDC | `{{pkg}}.filter` |
| Caches | In-process or distributed cache layer | `{{pkg}}.cache` |
| Analytics / metrics | Metrics emission, usage tracking | `{{pkg}}.analytics` |

---

## 7. Type Inventory

| Layer | Count | Notes |
|-------|-------|-------|
| Controllers | | |
| Connectors (custom) | | |
| Connectors (SDK-managed) | | |
| Domain classes | | |
| Response mappers / assemblers | | *(omit if 0)* |
| Validators | | |
| Consumers | | |
| Error codes | | |

---

## 8. Forbidden / Do-Not-Add Rules

- No new controller without a matching ## Capability Catalogue row
- No new connector without documenting timeout, pool size, error classification, and retry posture in [`CONNECTORS.md → ## 6. Adding a New Connector`](CONNECTORS.md#6-adding-a-new-connector)
- No capability without a Key Decisions entry — if it isn't decided, don't ship it
- No upward dependencies (Controller ← Service ← Connector ← Domain only)
- No cross-service DB access — always through a connector

---

## 9. Related Files

| File | Relationship |
|------|-------------|
| [SERVICE_CARD.md](SERVICE_CARD.md) | ## Module Index Summary points here; ## Data Contracts links to CONTRACTS.md / CONNECTORS.md / PLATFORM.md directly |
| [CONTRACTS.md](CONTRACTS.md) | Anchor targets for ## Capability Catalogue Controller rows + ## Cross-Reference Indexes endpoint index; also Async Contracts targets for ## Capability Catalogue Events rows + topic index |
| [CONNECTORS.md](CONNECTORS.md) | Anchor targets for ## Capability Catalogue Connector rows + ## Cross-Reference Indexes connector index |
| [PLATFORM.md](PLATFORM.md) | Profiles, scaling, feature flags, caches, deployment topology |
| [DB_SCHEMA.md](DB_SCHEMA.md) | Table-level detail for every entity referenced in ## Capability Catalogue rows; capabilities that read/write DB state should link to `DB_SCHEMA.md#<table_name>` |
| [CODE_PATTERNS.md](CODE_PATTERNS.md) | Naming conventions, error handling, and test patterns for anything added here |
| [BUILD_CONFIG.md](BUILD_CONFIG.md) | Dependencies & properties required by capabilities |
