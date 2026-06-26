---
service: {{SERVICE_NAME}}
owner: {{OWNER_EMAIL}}
status: production
repo: {{REPO_URL}}
tier: {{1|2|3}}
last_updated: {{DATE}}
---

# Service Card — {{SERVICE_NAME}} ({{SHORT_NAME}})

> A structured snapshot of this service — what it does, its API surface, and its coding conventions — sufficient for planning and implementing new features without reading the source repository.
>
> **Last Updated:** {{DATE}}

---

## 1. What We Do

{{One paragraph: what this service is, who calls it, what problem it solves.}}

---

## 2. What We Explicitly Do Not Do

{{Bulleted list of hard boundaries. Each item explains what is NOT this service's responsibility
and who owns it instead. These are the guardrails Claude must respect during design.}}

- **Does NOT ...** — {{who does instead}}

---

## 3. Contracts

> **Cross-reference convention:** every row in the tables below links directly to the
> exact section of the companion file (GitHub-slug anchor). "See `CONTRACTS.md`" or
> "see `CONNECTORS.md`" pointers are NOT allowed — downstream skills (apply-api-spec, apply-experience-card)
> should be able to jump from a row to a single operation/connector/topic block without
> scanning the file.

### APIs We Expose

| Endpoint | Method | Purpose | Version | Stability |
|----------|--------|---------|---------|-----------|
| [`/path`](CONTRACTS.md#<method>-<path-slug>--<purpose-slug>) | | | | STABLE / IN FLUX |


### APIs We Consume

| Service | Connector Interface | Connector Impl | Auth Type | Purpose |
|---------|-------------------|----------------|-----------|---------|
| [Name](CONNECTORS.md#<n>-<connectorname>-<desc>) | | | `{{e.g., Bearer / API_KEY / mTLS}}` | |

### Events We Consume

| Event/Topic | Source Service | Consumer Class | Purpose |
|-------------|---------------|----------------|---------|
| [`topic_name`](CONTRACTS.md#topic_name) | | | |

### Events We Publish

| Event/Topic | Consumer Service(s) | Publisher Class | Purpose |
|-------------|-------------------|-----------------|---------|
| [`topic_name`](CONTRACTS.md#topic_name) | | | |

---

## 4. Module Index (Summary)

Package root: `{{ROOT_PACKAGE}}`

> **Authoritative capability resolver:** [`MODULE_INDEX.md`](MODULE_INDEX.md) — contains
> the full Capability Catalogue (§4), Key Decisions per capability, cross-reference
> indexes (§5), and type inventory (§7). This section is a summary only; Impact
> Analysis / SD / EP / code-gen should load `MODULE_INDEX.md` first.

| Domain | # Capabilities | MODULE_INDEX anchor |
|--------|----------------|---------------------|
| | | [link](MODULE_INDEX.md#<anchor>) |

---

## 5. Source Structure

| Layer | Path | Files |
|-------|------|-------|
| {{layer_name}} | `{{path/}}` | {{N}} files |

---

## 6. What You Need to Know Before Planning Work Here

### Fragile Areas

{{List areas where changes have caused incidents or have subtle coupling.
Each item: what the area is, why it's fragile, what to watch out for.}}


### Known Constraints

{{Resource limits, latency budgets, scaling boundaries, versioning rules.}}

### Test Coverage

{{Where coverage is strong, where it's weak, what's growing.}}

### LLD Hints

{{Implementation rules for new code. Where to put new endpoints, services, connectors.}}

- **Flag policy:** {{when-must-a-new-feature-be-flag-gated — e.g. "internal endpoints always flagged; external endpoints case-by-case". If no written policy: `Unknown — requires product-owner input` and note the observed convention from `PLATFORM.md §3`.}}

---

## Companion Files

| File | When to Load | Purpose |
|------|-------------|---------|
| `MODULE_INDEX.md` | Impact analysis, feature planning, code generation (load FIRST) | Capability → implementation mapping: which classes, connectors, events, and flags implement each business capability, with Key Decisions per capability. |
| `CONTRACTS.md` | Feature planning, code generation | The service's full API surface: HTTP operations it exposes and async events it consumes and publishes, with request/response shapes and auth. |
| `CONNECTORS.md` | Feature planning, code generation | Every outbound dependency: auth, transport config, resilience settings, and error-handling posture. |
| `BUILD_CONFIG.md` | Code generation | Runtime stack, dependencies, environment variables, and secrets. |
| `CODE_PATTERNS.md` | Code generation | Team coding conventions: naming, package layout, error handling, validation, test approach. |
| `PLATFORM.md` | Impact analysis, deployment planning | Deployment topology, monitoring, feature flags, caching, database, active migrations, runbooks. |
| `DB_SCHEMA.md` | Impact analysis, code generation (anything touching persisted state) | The database this service owns: entity mapping, tables, relationships, and migrations. |
