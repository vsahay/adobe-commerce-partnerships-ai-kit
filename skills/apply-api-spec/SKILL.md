---
name: apply-api-spec
description: >
  Reads a feature spec, locates the matching API spec, reads the project's backend
  service cards, and produces a Low Level Design (LLD) document describing all the
  backend implementation changes needed. Does not write code — outputs a structured
  engineering plan.
---

# apply-api-spec

Produce a Low Level Design (LLD) for implementing a feature's backend by combining
a feature spec with its API spec and the project's backend service cards.

---

## Input

```
<targetRepo> <featureSpecPath>
```

| Argument | Required | Description |
|----------|----------|-------------|
| `targetRepo` | Yes | Absolute path to the target application repo (e.g. `/Users/me/bridge`). Used to locate service cards and to read the codebase when scanning for existing implementations. |
| `featureSpecPath` | Yes | Absolute or relative path to the feature spec file (e.g. `feature-specs/anytimeUpgrade/feature-spec/anytime-upgrade.md`). |

The skill derives everything else:
- **featureName** — the directory segment immediately after `feature-specs/` in the `featureSpecPath` (e.g. `feature-specs/anytimeUpgrade/...` → `anytimeUpgrade`). If the path does not contain `feature-specs/`, use the filename stem.
- **API spec** — `feature-specs/<featureName>/APISpecs/<feature-spec-filename>.md` (relative to AI-Kit working directory)
- **Backend service cards** — `<targetRepo>/docs/ai-kit/service-cards/backend/`
- **LLD output path** — `<targetRepo>/docs/ai-kit/LLD/backend/<feature-spec-filename>-lld.md`

> **Prerequisite:** Backend service cards for `<targetRepo>` must exist before running this skill. If they don't exist, run `/generate-backend-service-card <targetRepo>` first. See `manual/README.md` for the full workflow.

---

## How This Skill Reads Context

This skill always consults both service cards and source code. Real-world feature specs are rarely complete enough to produce a mergeable LLD from cards alone, and pretending otherwise buries assumptions that surface later as bugs.

| Principle | Detail |
|-----------|--------|
| **Service cards are the primary source of truth** | Cards resolve most questions directly: capability → class map, Key Decisions, Enforcement citations, call graphs, connector config, constants conventions, and shared standards. |
| **Source code fills gaps and validates card accuracy** | When cards are terse or silent, open the relevant source file directly. No audit trail is kept — cards are regenerated fresh each run. |
| **No fixed reading order** | If the feature describes a new connector and the card already answers the integration question, read the card and skip source. If the card's call graph for the relevant capability is terse, open the implementation file directly. |
| **Prefer reading** | Controllers, services, models/DTOs, connector implementations, route handlers, config files. |
| **Avoid reading** | Tests, generated code, unrelated modules. |

---

## Step 1 — Read the feature spec

Read `<featureSpecPath>`. Extract:
- Feature name and user-facing goal
- Which personas this feature serves
- Which backend operations are required (create / read / update / delete; which entities)
- Any explicit constraints or non-goals stated in the spec
- Acceptance criteria — used in Step 5 to validate the LLD is complete

---

## Step 2 — Locate and read the API spec

Derive the API spec path:

```
feature-specs/<featureName>/APISpecs/<feature-spec-filename>.md
```

Read the API spec. Extract per operation:
- HTTP method + endpoint path
- Required request headers (auth, org ID, API key, correlation ID)
- Request body shape (field names, types, required vs optional)
- Response body shape
- Query parameters
- Error responses and their semantics
- Any idempotency or ordering constraints

If the API spec file does not exist at the derived path, tell the user what path was looked for and ask them to provide the correct path before proceeding.

---

## Step 3 — Read the backend service cards

> These cards document **your own application's codebase** (the `targetRepo`), not Adobe's APIs.

Read the following files from `<targetRepo>/docs/ai-kit/service-cards/backend/`:

| File | What to extract |
|------|----------------|
| `SERVICE_CARD.md` | Application identity, fragile areas, and companion file index — note any fragile areas that the new feature's touch points intersect |
| `MODULE_INDEX.md` | Existing capabilities and their implementing files — detect reuse opportunities and avoid duplicating logic |
| `CONTRACTS.md` | All existing inbound HTTP routes and async events — verify no new operation overlaps an existing route |
| `CONNECTORS.md` | All existing outbound connectors — check whether new downstream calls can reuse an existing connector; note auth, transport, retry, and error posture for any connector to be reused |
| `BUILD_CONFIG.md` | Technology stack, framework version, key dependencies, and env var conventions — all generated code must match the project's language, framework, and library versions |
| `CODE_PATTERNS.md` | Naming conventions, file layout, controller pattern, route handler pattern, validation approach, error handling, constants conventions |
| `DB_SCHEMA.md` | Existing data models and schema — determines whether new entities are needed; check if stateless (if so, no DB changes are required) |
| `PLATFORM.md` | Runtime config, caching policy, env vars, health checks, deployment constraints — relevant if the feature adds new env vars, changes caching, or has deployment requirements |

---

## Step 4 — Write the LLD

Before writing each section, actively consult both sources:

| When to consult | What to do |
|-----------------|-----------|
| **Service cards have a clear answer** | Use the card directly — capability map, connector config, error codes, call graph, standards. Do not re-read source for information already captured. |
| **Card section is terse, missing, or suspect** | Open the relevant source file. Common triggers: call graph has no step count; connector shows `none configured` for timeouts/retries; a capability was recently added and the card may not reflect it. |
| **Card and source contradict** | Trust source. Note the discrepancy in LLD §5 (Design Decisions) and proceed with the source-accurate version. |

---

### LLD §1 — Summary

One paragraph: what the feature does, which API endpoints it introduces or modifies, and what the end-to-end data flow looks like at a high level.

---

### LLD §2 — Data Flow

For each operation the feature introduces or modifies, the complete call chain:

```
Operation: <what triggers this call — user action or upstream event>

  → <HTTP method> <inbound route handler path>
  → <business logic handler / controller>
  → <outbound connector> → <external API endpoint>
  ← response: <shape>
  ← handler maps to: <DTO / response schema>
  ← route returns: HTTP <status> + <body shape>
```

One block per operation. If an operation is purely read-through with no transformation, say so explicitly.

---

### LLD §3 — Change Summary

Before writing this section, cross-check `MODULE_INDEX.md` and `CONTRACTS.md` for existing files that could be extended rather than replaced. Prefer MODIFY over CREATE wherever the existing pattern supports it.

**Structure:** Write a slim overview table first, then a named subsection for every file.

**Overview table** — one row per file, no implementation detail:

| File | Action | Layer | Reason |
|------|--------|-------|--------|
| `<path/to/file>` | CREATE / MODIFY | Model / Controller / Route / Connector / Constants / Util / Config | `<one-line reason>` |

**Layer values:** Model, Controller, Route, Connector, Constants, Util, Config.  
**Action values:** CREATE (new file), MODIFY (existing file changed).

**Per-file subsections** — immediately after the table, one subsection per file:

```
### `<path/to/file>` — CREATE / MODIFY
```

> **MODIFY files — mandatory:** Read the existing source file implementation before writing its subsection.

Each subsection must contain everything an engineer needs to write the file without reading any other section. Use the formatting that best fits the content:

- **Bullet lists** for discrete implementation steps and behaviours
- **Tables** for request/response fields, params, or error mappings
- **Code blocks** for schema definitions, constant objects, and function signatures

Include as applicable per layer:

- **Route handlers / validators:** HTTP method and path; request fields (name, type, required/optional, validation rule, source — query param / body / header); response fields (name, type, source); error conditions → HTTP status mapping (e.g. missing required field → 400, upstream 4xx → 502, auth failure → 503)
- **Controllers / service handlers:** function signatures; business logic steps; which connector methods are called and with what params; response DTO shape
- **Connectors:** class name and base class; methods added with their upstream endpoint, auth pattern (Bearer / API key / mTLS), retry config, and timeout; base URL env var name
- **Constants files:** constant names and values in a code block
- **Models / schemas:** schema or type definitions in a code block using the project's validation library, with required vs optional annotated
- **Utilities:** function signatures and the logic they apply

---

### LLD §4 — DB Changes

If the service is stateless (no data store): write `No DB changes — service is stateless.`

Otherwise, for each schema or migration change:

| Change type | Entity / table | Field / index | Reason |
|-------------|---------------|--------------|--------|
| ADD COLUMN / CREATE TABLE / ADD MIGRATION | `<table>` | `<field or migration filename>` | `<reason>` |

Follow with field-level detail for any new table or significant schema change.

---

### LLD §5 — Design Decisions

For each non-obvious choice — deviations from existing patterns, new abstractions, or significant trade-offs:

| Decision | Why | Trade-off | Enforcement |
|----------|-----|-----------|-------------|
| `<what was decided>` | `<constraint or reason — cite service card section if applicable>` | `<what is gained vs lost>` | `<naming convention / card section / "not enforced — relies on convention">` |

---

### LLD §6 — Acceptance Criteria Coverage

| # | Acceptance criterion (from feature spec) | LLD section | Status |
|---|----------------------------------------|-------------|--------|
| 1 | `<verbatim criterion>` | §3.N `<filename>` | ✅ Covered |
| 2 | `<verbatim criterion>` | — | ⚠ Not covered — `<reason>` |

Flag every unmet criterion with ⚠ and a one-line reason.

---

### LLD §7 — Out of Scope

List anything the feature spec mentions that this LLD explicitly does not cover, and why.

If nothing is excluded, write "Nothing explicitly excluded."

---

## Step 5 — Self-validate the LLD

Before writing the output file, check:

**Completeness**
- [ ] Every API operation in the API spec has at least one corresponding row in §3 (Change Summary)
- [ ] Every file in the §3 overview table has a corresponding named subsection with enough detail to implement it without reading any other document
- [ ] Every acceptance criterion from the feature spec is mapped in §6 — unmet criteria are flagged with ⚠

**Correctness**
- [ ] No new route in §3 duplicates an existing route in `CONTRACTS.md`
- [ ] Every connector entry in §3 is consistent with `CONNECTORS.md` (auth pattern, retry posture, base URL source)
- [ ] All field types and validations in §3 match the API spec request/response shapes exactly
- [ ] All fragile areas from `SERVICE_CARD.md` that the feature touches are called out in §5 (Design Decisions)

**DB**
- [ ] If the service owns a data store, §4 (DB Changes) is populated; if stateless, §4 says so explicitly

Fix any gaps before writing the output file.

---

## Step 6 — Output the LLD

Write the LLD as a markdown document to:

```
<targetRepo>/docs/ai-kit/LLD/backend/<feature-spec-filename>-lld.md
```

Create the directory if it does not exist.

Then print a summary:

```
✅ LLD written to <targetRepo>/docs/ai-kit/LLD/backend/<feature-spec-filename>-lld.md

Backend changes:     <N> files (<M> create, <K> modify)
DB changes:          <summary or "none — stateless">
Design decisions:    <N>
AC coverage:         <N covered, M flagged ⚠>

⚠ Gaps / flags:
  - <any acceptance criteria not fully covered>
  - <any fragile area intersections to review>
  - <any card sections flagged for update in §8>
```
