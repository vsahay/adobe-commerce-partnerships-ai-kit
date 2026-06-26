# Platform — {{SERVICE_NAME}}

> Runtime configuration that affects code generation: feature flags, caching strategy,
> database stack, and in-flight migrations. Load during impact analysis and feature planning.
>
> **Last Updated:** {{DATE}}

---

## 1. Feature Flags

<!--
List every flag consulted in code. For each: name, mechanism (env var / config property /
feature-flag service), default, the capability it gates (link to MODULE_INDEX.md), and purpose.
If a flag's purpose is unknowable from code, state `Unknown — requires product-owner input`.
-->

| Flag name | Mechanism | Default | Capability | Purpose |
|-----------|-----------|---------|-----------|---------|
| `{{flag_name}}` | `{{env var / config property / feature-flag service}}` | `{{ON / OFF / value}}` | [{{CapabilityName}}](MODULE_INDEX.md#<anchor>) | {{what this flag gates}} |

**Flag naming convention:** {{describe the pattern observed — e.g., "env vars are UPPER_SNAKE_CASE; config properties are dot.separated.lowercase; feature-flag service keys are kebab-case"}}.

**Flag policy:** {{when must a new feature be flag-gated — e.g., "all external-facing endpoints require a flag; internal endpoints case-by-case" — or `Unknown — requires product-owner input`}}.

---

## 2. Caching

<!--
One row per cache strategy in use. Omit if the service has no caching.
-->

| Strategy | Location | Cache key pattern | TTL | Invalidation trigger |
|----------|----------|-------------------|-----|---------------------|
| `{{e.g., Idempotency cache}}` | `{{path/to/cache/}}` | `{{prefix}}:{{id}}` | `{{N}}s` | `{{update path / event}}` |
| `{{e.g., Response cache}}` | `{{path/to/cache/}}` | `{{prefix}}:{{id}}` | `{{N}}s` | `{{update path / event}}` |

**Cache library / provider:** `{{e.g., Redis / Caffeine / in-memory / none}}`

> Cross-reference: [`DB_SCHEMA.md → ## 7. Cache Strategy`](DB_SCHEMA.md#7-cache-strategy) for per-table cache details.

---

## 3. Database

> Full schema detail is in [`DB_SCHEMA.md`](DB_SCHEMA.md). This section records only the stack choices that drive code patterns.

| Field | Value |
|-------|-------|
| Engine | `{{e.g., PostgreSQL / MySQL / DynamoDB / MongoDB}}` |
| ORM / data access | `{{e.g., Spring Data JPA / Prisma / SQLAlchemy / GORM / none}}` |
| Migration tool | `{{e.g., Flyway / Liquibase / Alembic / Prisma Migrate / golang-migrate}}` |
| DDL mode | `{{e.g., validate / none — never auto-create in production}}` |

---

## 4. Active Migrations

<!--
Only list migrations IN PROGRESS that affect code or behaviour now.
Completed migrations belong in a changelog, not here.
-->

| Migration | Status | Impact on code generation |
|-----------|--------|--------------------------|
| `{{e.g., v1 → v2 API}}` | IN PROGRESS | `{{what it means for new endpoints / models}}` |
