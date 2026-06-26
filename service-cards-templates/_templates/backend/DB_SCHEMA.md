# Database Schema — {{SERVICE_NAME}}

> **Purpose:** Document the database this service owns — schema, entity mapping,
> relationships, migrations, and drift audit.
>
> **Last Updated:** {{DATE}}

<!--
  Schema source — pick ONE and delete the others:
    - "Schema source: migration files at {{path}}"
    - "Schema source: ORM entity scan at commit {{hash}}"
    - "Schema source: live DB connection to {{host}}:{{port}} on {{DATE}}"
    - "Schema source: no relational DB — see §3 for owned data stores"
-->

---

## 1. Overview

| Field | Value |
|-------|-------|
| Engine | `{{e.g., PostgreSQL / MySQL / DynamoDB / MongoDB}}` |
| Schema / database name | `{{SCHEMA_NAME}}` |
| System-of-record boundary | `{{which data does this service own vs reference from elsewhere}}` |
| Schema source | `{{migration files / ORM entity scan / live connection / no relational DB}}` |

---

## 2. Connection & Deployment

| Field | Value |
|-------|-------|
| Connection string / URL | `{{e.g., postgres://host:5432/dbname or jdbc:mysql://host:3306/db}}` |
| Connection pool | `{{library and key settings — e.g., HikariCP maxPoolSize=10 / pg pool max=10}}` |
| Credentials | see [`BUILD_CONFIG.md → ## 4. Secrets & Sensitive Data`](BUILD_CONFIG.md#4-secrets--sensitive-data) |
| HA / replica topology | `{{e.g., Aurora writer + 2 readers / single instance / multi-AZ}}` |

---

## 3. Entity → Table Mapping

*One row per owned table or collection. Adapt column headers for non-relational stores.*

| Entity / Model class | Table / Collection name | Primary key | Source file |
|----------------------|------------------------|-------------|-------------|
| `{{ModelClass}}` | `{{table_name}}` | `{{pk_field}}` | `{{path/to/model}}` |

---

## 4. Tables

<!-- One `### <table_name>` per table. Heading must match the physical name verbatim — anchors are used by MODULE_INDEX.md capability rows. -->

### {{TABLE_NAME}}

**Description:** <!-- one-line purpose -->

**Columns**

| Column | Type | Nullable | Default | Notes |
|--------|------|----------|---------|-------|
| `id` | `BIGINT` | NO | AUTO_INCREMENT | PK |

**Primary key:** `{{pk_column_or_composite}}`

**Indexes**

| Name | Columns | Unique |
|------|---------|--------|
| | | |

**Foreign keys**

| Column | References | On delete |
|--------|-----------|-----------|
| | | |

---

## 5. Relationships

```
{{EntityA}} 1 ───< N {{EntityB}}    (via {{fk_column}})
{{EntityB}} N >─── M {{EntityC}}    (via join table {{join_table_name}})
```

| Side A | Cardinality | Side B | Join / FK |
|--------|-------------|--------|-----------|
| `{{EntityA}}` | 1 : N | `{{EntityB}}` | `b.a_id → a.id` |

---

## 6. Migrations

| Field | Value |
|-------|-------|
| Tool | `{{e.g., Flyway / Liquibase / Alembic / Prisma Migrate / golang-migrate}}` |
| Changelog / migrations location | `{{e.g., db/migrations/ / classpath:db/changelog/}}` |
| DDL mode | `{{e.g., validate / none / auto}}` |
| Last applied (per env, if known) | `{{filename or version id}}` |

---

## 7. Cache Strategy

> Cross-reference: [`PLATFORM.md → ## 4. Caching`](PLATFORM.md#4-caching).

| Table / query | Cache layer | Key pattern | TTL | Invalidation trigger |
|---------------|-------------|-------------|-----|---------------------|
| `{{TABLE_NAME}}` | `{{e.g., Redis / in-memory / none}}` | `{{prefix}}:{{id}}` | `{{N}}s` | `{{update path / event}}` |

---

## 8. Drift Audit

> Populated when both an ORM entity scan AND a schema source are available.
> If either is missing, mark as `NOT RUN — <reason>`.

**Status:** `{{POPULATED / NOT RUN — no schema source / NOT RUN — no ORM entities}}`

| Kind | Entity / field | Table / column | Delta | Risk |
|------|---------------|----------------|-------|------|
| Entity-only | `{{field on model, no matching column}}` | — | | |
| Schema-only | — | `{{column in DB, no matching field}}` | | |
| Type mismatch | `{{entity field}}` | `{{table.column}}` | `{{entity type}} vs {{db type}}` | |

---

## 9. Secrets

*DB credentials only. All other service secrets are in [`BUILD_CONFIG.md → ## 4. Secrets & Sensitive Data`](BUILD_CONFIG.md#4-secrets--sensitive-data).*

| Credential | Env var | Purpose |
|------------|---------|---------|
| DB username | `DB_USERNAME` / `DB_USER` | Database authentication |
| DB password | `DB_PASSWORD` | Database authentication |

---

## 10. Source

- [ ] Migration files: `{{path}}`
- [ ] ORM entity scan at commit `{{hash}}` (branch `{{branch}}`)
- [ ] Live DB connection: `{{host}}:{{port}}` scanned on `{{YYYY-MM-DD}}`
