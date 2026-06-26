---
name: implement-feature
description: >
  Executes a pre-written LLD by reading the target repo's service cards for
  coding conventions, then writing all files specified in the LLD. Supports
  backend-only, UI-only, or full-stack generation via an optional mode argument.
  The LLD defines WHAT to build; the service cards define HOW to write it.
---

# implement-feature

Execute a feature LLD — generate all specified files using the target repo's own
coding conventions.

---

## Input

```
<featureName> <targetRepoPath> [--mode=both|backend|ui] [--use-reference-files]
```

| Argument | Required | Description |
|----------|----------|-------------|
| `featureName` | Yes | Kebab-case feature name (e.g. `flexible-discounts`). Used to locate LLD and service card files. |
| `targetRepoPath` | Yes | Absolute path to the target repo. Code is written here. |
| `--mode` | No | `--mode=both` (default) — backend + UI + wiring check. `--mode=backend` — backend only. `--mode=ui` — UI only. Named flag; can appear in any position after `targetRepoPath`. |
| `--use-reference-files` | No | When set, load LLDs and service cards from the kit repo's pre-approved reference location instead of the target repo. Use when the target repo does not yet have generated artifacts. |

**Derived paths:**

| Resource | Default path | Path when `--use-reference-files` |
|----------|-------------|-----------------------------------|
| Backend LLD | `<targetRepoPath>/docs/ai-kit/LLD/backend/<featureName>-lld.md` | `feature-specs/<featureName>/reference-files/LLD/backend/<featureName>-lld.md` |
| UI LLD | `<targetRepoPath>/docs/ai-kit/LLD/ui/<featureName>-lld.md` | `feature-specs/<featureName>/reference-files/LLD/ui/<featureName>-lld.md` |
| Backend service cards | `<targetRepoPath>/docs/ai-kit/service-cards/backend/` | `feature-specs/<featureName>/reference-files/service-cards/backend/` |
| UI service cards | `<targetRepoPath>/docs/ai-kit/service-cards/ui/` | `feature-specs/<featureName>/reference-files/service-cards/ui/` |

If a LLD filename doesn't exactly match `<featureName>-lld.md`, search the directory for `*<featureName>*-lld.md` and use the match.

When `--use-reference-files` is set, log: `Using pre-approved reference files from kit repo: feature-specs/<featureName>/reference-files/`

If a required LLD does not exist, stop and tell the user. Do not generate the LLD — run `apply-api-spec` and/or `apply-experience-card` first.

---

## Phase 1 — Load reference material

| File | Load when | Path |
|------|-----------|------|
| Experience card | always | `feature-specs/<featureName>/experience-card/` — search for `*.md` |
| Backend LLD | always — UI fetch hooks must match backend endpoint URLs, request shapes, and response schemas exactly | derived path (see Input section) |
| UI LLD | `--mode=both` or `--mode=ui` | derived path (see Input section) |
| Backend service cards (`CODE_PATTERNS.md`, `CONNECTORS.md`, `MODULE_INDEX.md`, `CONTRACTS.md`) | `--mode=both` or `--mode=backend` | derived path (see Input section) |
| UI service cards (`UI_CODE_PATTERNS.md`, `DATA_LAYER.md`, `STATE_MANAGEMENT.md`, `ROUTES.md`, `UI_MODULE_INDEX.md`) | `--mode=both` or `--mode=ui` | derived path (see Input section) |

From the experience card, extract: every user-facing screen and surface, all acceptance criteria, every user interaction and its expected outcome, and all loading / error / empty states described.

From each LLD, extract the complete file manifest from the **Change Summary (§3)** overview table — every row is one file to CREATE or MODIFY. The **named subsection** for each file (e.g. `### path/to/file — CREATE`) is the complete, self-contained implementation spec — it contains everything an engineer needs without reading any other section. Also read **§2 Data Flow** to understand how files connect end-to-end.

---

## Phase 2 — Generate backend code

**Skip if `--mode=ui`.**

Read the **Change Summary table (§3)** of the backend LLD — this is the authoritative file manifest. Every row is one file to CREATE or MODIFY.

**Execution order:** process files in dependency order — lower-level layers (data schemas, models) before the layers that import them (business logic, handlers, routes). Derive the correct order for this project's stack from `CODE_PATTERNS.md` and the Layer values in the Change Summary table.

For each file:
- The **named subsection** for this file in §3 is the complete implementation spec — read every bullet, table, and code block and implement exactly what it states
- Cross-reference **§2 Data Flow** to understand how this file fits into the end-to-end call chain
- Apply the patterns from backend service cards that match its layer type

Apply these rules for every file:

1. **Naming and location** — follow `CODE_PATTERNS.md` naming conventions and directory layout exactly.
2. **MODIFY files** — read the existing file first; add only what the LLD specifies; do not refactor existing code.
3. **CREATE files** — match the file structure pattern from `CODE_PATTERNS.md` for that layer type.
4. **Auth and outbound calls** — use the exact auth header pattern, HTTP client, and error handling posture from `CONNECTORS.md`.
5. **Error handling** — use the project's error type and throw pattern from `CODE_PATTERNS.md §Error Handling`.
6. **Validation** — apply the validation approach (library, invocation point) from `CODE_PATTERNS.md §Validation`.
7. **Constants** — follow the naming and object structure convention from `CODE_PATTERNS.md §Constants`.
8. **Reuse** — import existing schemas, utilities, or helpers identified in `MODULE_INDEX.md` rather than re-implementing them.

Write exactly what the LLD specifies — no extra fields, no extra error cases, no refactoring of surrounding code.

---

## Phase 3 — Generate UI code

**Skip if `--mode=backend`.**

Read the **Change Summary table (§3)** of the UI LLD — this is the authoritative file manifest. Every row is one file to CREATE or MODIFY.

**Execution order:** process files in dependency order — shared utilities and data-fetching layers before the components that consume them, state stores before the components that read them. Derive the correct order for this project's stack from `UI_CODE_PATTERNS.md` and the Layer values in the Change Summary table.

For each file:
- The **named subsection** for this file in §3 is the complete implementation spec — read every bullet, table, and code block and implement exactly what it states
- Cross-reference **§2 Data Flow** to understand how this file connects to other files and to the backend
- Apply the patterns from UI service cards that match its layer type

Apply these rules for every file:

1. **Naming and location** — follow `UI_CODE_PATTERNS.md` naming conventions and directory layout exactly.
2. **MODIFY files** — read the existing file first; add only what the LLD specifies; do not refactor existing code.
3. **CREATE files** — match the file structure pattern from `UI_CODE_PATTERNS.md` for that layer type.
4. **Data fetch files** — use the exact fetch pattern from `DATA_LAYER.md`: auth injection, cache key format, dependency guard, cache freshness, error surface. Endpoint, request shape, and response shape come verbatim from the **Change Required column** of the hook's §3 row and from the backend LLD — do not invent contracts.
5. **Auth and session** — read auth token, org ID, and other session values using the pattern from `STATE_MANAGEMENT.md` and `DATA_LAYER.md §Auth Token Injection`.
6. **Route / page files** — follow the page structure pattern from `UI_CODE_PATTERNS.md`. Auth guard, layout wrapper, and navigation patterns come from `ROUTES.md`.
7. **Component files** — implement the props interface, visual spec, all states (loading / error / empty / happy path), and interactions exactly as the **Change Required column** of the component's §3 row specifies. Design system imports follow the pattern in `UI_CODE_PATTERNS.md`.
8. **Style files** — semantic class names only; values from the LLD's visual spec; follow the file naming convention in `UI_CODE_PATTERNS.md`.
9. **Utility / type files** — implement exactly the functions, types, and constants listed in the **Change Required column** of the file's §3 row. Match the existing file's naming and export style.
10. **State store files** — follow the existing store pattern from `STATE_MANAGEMENT.md`; add only the fields specified in the **Change Required column** of the file's §3 row.
11. **Reuse** — import existing components, hooks, and utilities identified in `UI_MODULE_INDEX.md` rather than re-implementing them.

Write exactly what the LLD specifies — no extra props, no extra states, no refactoring of surrounding code.

---

## Phase 4 — Wiring check

**Skip if `--mode=backend` or `--mode=ui`.**

### 4a — Technical wiring

For each data flow in UI LLD §2 (Data Flow), verify:

- [ ] Data fetch unit endpoint URL matches the server handler path exactly
- [ ] Every query parameter the server handler requires — including any dispatch or routing params (e.g. `type`, `action`) — is present in the request sent from the UI; compare the server handler's full param list against what the UI actually sends
- [ ] Request fields sent from the UI match what the server handler accepts
- [ ] Response fields accessed in the UI match what the server handler returns
- [ ] All required auth/session params are forwarded on every call that needs them
- [ ] Navigation targets exist as page files and receive the correct params
- [ ] Every backend LLD operation that the UI calls has been implemented

### 4b — Experience card validation

For each acceptance criterion and user-facing surface in the experience card, verify the implementation satisfies it:

- [ ] Every screen or surface described in the experience card has a corresponding route or component in the implementation
- [ ] Every user interaction described (click, submit, navigate) has a corresponding event handler wired to the right action
- [ ] Every loading state described is rendered while the relevant fetch is in flight
- [ ] Every error state described is rendered when the relevant fetch or mutation fails
- [ ] Every empty state described is rendered when data is absent
- [ ] Every acceptance criterion from the experience card is satisfied by at least one implemented component, hook, or route — flag any that are not covered

Fix every mismatch before writing the report.

---

## Phase 5 — Report

```
✅ Feature implemented: <featureName>  [--mode=both | backend | ui]

Backend files:          (omit if --mode=ui)
  Created:  <list>
  Modified: <list>

UI files:               (omit if --mode=backend)
  Created:  <list>
  Modified: <list>

Reuse applied:
  <component / hook / schema> imported from <file>

Wiring verified:        (omit if --mode=backend or --mode=ui)
  ✓ Endpoint URLs
  ✓ All query params sent from UI to server (including dispatch/routing params)
  ✓ Auth/session param forwarding
  ✓ Request/response shapes
  ✓ Navigation params

Experience card coverage:  (omit if --mode=backend or --mode=ui)
  ✓ <N> acceptance criteria satisfied
  ⚠️  <list any criteria not fully covered, or "none">

⚠️  Design decisions applied:
  - <decision from LLD>: <how it was implemented>
```
