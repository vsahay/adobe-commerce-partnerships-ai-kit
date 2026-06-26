---
name: apply-experience-card
description: >
  Reads an experience card, optionally fetches the Figma design if a URL is
  defined in the card, reads the project's UI service cards, and produces a
  Low Level Design (LLD) document describing all UI implementation changes an
  engineer needs to make. Does not write code — outputs a structured UI engineering plan.
---

# apply-experience-card

Produce a UI Low Level Design (LLD) from an experience card. If the card defines
a Figma URL the design is fetched and used as the visual reference. If no URL is
present the LLD is generated from the experience card description alone.

---

## Input

```
<featureSpecPath> <targetRepo> <backendLLDPath>
```

| Argument | Required | Description |
|----------|----------|-------------|
| `featureSpecPath` | Yes | Path to the experience card file (e.g. `feature-specs/anytimeUpgrade/experience-card/anytime-upgrade.md`) |
| `targetRepo` | Yes | Absolute path to the target UI repo (e.g. `/Users/me/bridge`). Used to locate UI service cards and scan the codebase for existing components, data fetch units, and routes. |
| `backendLLDPath` | No | Absolute or relative path to the backend LLD for this feature (e.g. `/Users/me/bridge/docs/ai-kit/LLD/backend/anytime-upgrade-lld.md`). Provides the exact API endpoints, request/response shapes, and auth contracts the UI must call. Pass `none` if the feature makes no new backend calls — Step 4 is skipped and all data fetch units must be derived from existing service cards only. |

The skill derives everything else:
- **featureName** — the directory segment immediately after `feature-specs/` in the `featureSpecPath` (e.g. `feature-specs/anytimeUpgrade/...` → `anytimeUpgrade`). If the path does not contain `feature-specs/`, use the filename stem.
- **Figma URL** — extracted from the experience card if present (see Step 1)
- **UI service cards** — `<targetRepo>/docs/ai-kit/service-cards/ui/`
- **LLD output** — `<targetRepo>/docs/ai-kit/LLD/ui/<experience-card-filename>-lld.md`

> **Prerequisite:** UI service cards for `<targetRepo>` must exist before running this skill. If they don't exist, run `/generate-ui-service-card <targetRepo>` first. See `manual/README.md` for the full workflow.

> **Figma:** Step 2 requires the Figma MCP plugin to be configured in Claude Code. If it is not available, the Figma URL branch is skipped automatically — the LLD is generated from the experience card description alone and all visual specs are marked approximate.

---

## How This Skill Reads Context

This skill always consults both UI service cards and source code. Real-world experience cards are rarely complete enough to produce a mergeable LLD from cards alone, and pretending otherwise buries assumptions that surface later as bugs.

| Principle | Detail |
|-----------|--------|
| **Service cards are the primary source of truth** | Cards resolve most questions directly: capability → component/fetch unit map, Key Decisions, Enforcement citations, call graphs, `DATA_LAYER.md` fetch patterns, `STATE_MANAGEMENT.md` store shapes, `UI_PLATFORM.md` stack constraints and framework rules. |
| **Source code fills gaps and validates card accuracy** | When cards are terse or silent, open the relevant source file directly. No audit trail is kept — cards are regenerated fresh each run. |
| **No fixed reading order** | If the feature needs a new data fetch unit and `DATA_LAYER.md` already answers the pattern question, use the card and skip source. If the module index for the relevant capability is terse, open the component or fetch unit file directly. |
| **Prefer reading** | Page/view components, data fetch units, state store files, router config, stylesheet files. |
| **Avoid reading** | Tests, generated code, unrelated modules. |

---

## Step 1 — Read the experience card

Read `<experienceCard>`. Extract:

- Feature name and user-facing goal
- **Figma URL** — look for any field or line that contains a `figma.com` URL; record it if found
- Every UI screen or view described
- Component-level requirements: what elements appear, in what hierarchy
- Conditional rendering rules (state A shows X, state B shows Y)
- Business logic and calculations that drive the UI
- Error states, loading states, and empty states described
- API operations and data the UI depends on
- User interactions: clicks, form submissions, navigation triggers
- Acceptance criteria

Record whether a Figma URL was found. This determines the branch taken in Step 2.

---

## Step 2 — Fetch the Figma design (Figma URL branch)

**If a Figma URL was found in Step 1 → execute this step.**
**If no Figma URL was found → skip to Step 3.**

### 2a — Parse the URL

Extract:
- `fileKey`: alphanumeric segment after `/design/`
- `nodeId`: `node-id` query param, converting `-` to `:` (e.g. `1-10981` → `1:10981`)

### 2b — Fetch design context

Call `mcp__plugin_figma_figma__get_design_context` with `fileKey` and `nodeId`.

If `get_design_context` is unavailable, call in sequence:
1. `get_figma_node` — component tree and layout values
2. `get_figma_image` (scale: 2) — visual reference screenshot
3. `get_figma_styles` — design tokens and colors

### 2c — Extract visual spec

Before proceeding, compile a precision table for every major element:

| Element | Figma exact value | Notes |
|---------|-------------------|-------|
| (e.g.) Card padding | 31 16 16 16 px | asymmetric — check all 4 |
| (e.g.) Gap between items | 32px | — |
| (e.g.) Primary button fill | rgb(59,99,251) | accent variant |
| (e.g.) Body text color | rgb(41,41,41) | semantic token |

This table feeds directly into LLD Section 3 (Component Specs). If a value is not
determinable from Figma, mark it `⚠ NOT CAPTURED`.

---

## Step 3 — Read the UI service cards

> These cards document **your own application's codebase** (the `targetRepo`), not Adobe's UI.

Read all UI service cards from `<targetRepo>/docs/ai-kit/service-cards/ui/`:

| File | What to extract |
|------|----------------|
| `UI_SERVICE_CARD.md` | Application identity, what it does and does not do, key constraints for code generation (auth model, response type conventions, framework restrictions) |
| `UI_MODULE_INDEX.md` | Existing capabilities and their views/components/data fetch units — detect reuse and avoid duplication |
| `ROUTES.md` | Existing routes; auth guard mechanism; programmatic navigation pattern; how IDs are passed; navigation guard pattern |
| `DATA_LAYER.md` | Existing data fetch unit patterns — auth injection, cache key convention, cache freshness, dependency guard, error surface |
| `STATE_MANAGEMENT.md` | Existing state store shapes and their consumer patterns; browser storage keys |
| `UI_CODE_PATTERNS.md` | Naming conventions, directory structure, design system import pattern, component structure rules, test skeleton |
| `UI_PLATFORM.md` | Runtime stack, framework version, key dependencies, env var conventions, build scripts — use to ensure all generated code matches the project's stack |

> **Note:** Service cards may not reflect the latest codebase state. Before deciding CREATE vs MODIFY for any file, open the relevant source directory (components, hooks/composables/services, state stores) and confirm what actually exists. This is especially important if service cards were not regenerated recently.

---

## Step 4 — Read the backend LLD

**If `backendLLDPath` is `none` → skip this step.** All data fetch units in LLD Section 4 must be derived from existing `DATA_LAYER.md` entries only — do not invent new endpoints.

Read `<backendLLDPath>`. Extract per API operation:

| Field | What to extract |
|-------|----------------|
| **Endpoint** | HTTP method + path (e.g. `GET /api/midterm-upgrades?type=getUpgradePaths`) |
| **Auth requirement** | Bearer token required? orgId query param required? |
| **Request shape** | All query params and request body fields with types |
| **Response shape** | Fields the UI will consume — names, types, nesting |
| **Error responses** | HTTP status codes and their meaning for the UI (e.g. 401 → redirect to login, 404 → show empty state) |
| **New vs existing route** | Is this a new route or an extension of an existing one? |
| **Design decisions** | Any backend constraints that affect how the UI must call the API |

This becomes the **authoritative contract** for every data fetch unit written in LLD Section 4.
Do not invent endpoint URLs or request shapes — use only what the backend LLD defines.

---

## Step 5 — Write the LLD

Produce a structured LLD document with the following six sections.

---

### LLD §1 — Summary

One paragraph: what the feature's UI does, which pages or views are affected,
what data it depends on, and the overall user journey from entry to completion.

State clearly: **"Figma URL present — design reference used"** or
**"No Figma URL — LLD derived from experience card description only"**.

---

### LLD §2 — Data Flow

For each primary user action, the end-to-end call chain:

```
User action: <what the user does>
  → <ComponentName> event handler fires
  → <data fetch unit> mutation / query fires
  → <backend route> (handled by backend — see backend LLD)
  ← response: <shape>
  ← data fetch unit provides: <data / loading state / error>
  ← component renders: <outcome>
```

One block per primary interaction. Write this before the change table so the reader has the mental model first.

---

### LLD §3 — Change Summary

Every file that must be created or modified to ship this feature. Do not list files that are used unchanged.

**Structure:** Write a slim overview table first, then a named subsection for every file.

**Overview table** — one row per file, no implementation detail:

| File | Action | Layer | Reason |
|------|--------|-------|--------|
| `<filepath>` | CREATE / MODIFY | Route / Component / Data Fetch / State / Style / Util | `<why this file changes>` |

**Layer values:** Route, Component, Data Fetch, State, Style, Util.  
**Action values:** CREATE (new file), MODIFY (existing file changed).

**Per-file subsections** — immediately after the table, one subsection per file:

```
### `<filepath>` — CREATE / MODIFY
```

> **MODIFY files — mandatory:** Read the existing source file implementation before writing its subsection.

Each subsection must contain everything an engineer needs to write the file without reading any other section. Use the formatting that best fits the content:

- **Bullet lists** for discrete behaviours, interaction flows, and state descriptions
- **Tables** for state variables (name / type / default), props, request params, or field mappings
- **Code blocks** for type definitions, constant objects, style definitions, and inline snippets

Include as applicable per file type:

- **Page / route components:** entry point and auth guard note; state variables with type and default — distinguish which trigger a re-fetch vs which are local-only; derived values and their formula; pagination or infinite-scroll logic; loading / error / empty state UI; accessibility landmarks
- **Components:** props interface (name, type, required/optional); visual spec (layout, design system components used, custom style classes); interaction logic (event → state update / callback); loading / error / empty states; accessibility attributes
- **Data fetch units** (hooks, composables, services, queries — use the term from `DATA_LAYER.md`): cache or query key; freshness / TTL config; fetch guard condition; request params; pagination continuation logic for paginated queries; merge/dedup/filter logic; shape exposed to callers; graceful degradation on partial failure
- **Style files:** style class names and their layout/spacing/colour intent in a code block — use the project's styling approach from `UI_CODE_PATTERNS.md`
- **Types / utils / constants:** type or interface definitions and constant objects in code blocks; helper function signatures and logic

---

### LLD §4 — Design Decisions

For each non-obvious choice — deviations from existing patterns, new abstractions, or significant trade-offs:

| Decision | Why | Trade-off | Enforcement |
|----------|-----|-----------|-------------|
| `<what was decided>` | `<constraint or reason>` | `<what is gained vs lost>` | `<naming convention / linter rule / "not enforced — relies on convention">` |

---

### LLD §5 — Acceptance Criteria Coverage

Map each acceptance criterion from the experience card to the Change Summary row(s) that satisfy it. Flag any criterion not addressed.

| Goal (from experience card) | Covered by |
|-----------------------------|-----------|
| `<verbatim criterion>` | `<file(s) in §3>` |
| `<verbatim criterion>` | ⚠ Not covered — `<reason>` |

---

### LLD §6 — Out of Scope

List anything mentioned in the experience card that this LLD explicitly does not cover, and why.

If nothing is excluded, write "Nothing explicitly excluded."

---

## Step 6 — Self-validate the LLD

Before writing the output file, check:
- [ ] Every file in the §3 overview table has a corresponding named subsection with enough detail to implement it without reading any other document
- [ ] Every UI screen / view described in the experience card has at least one row in §3
- [ ] Every Figma component (Step 2c precision table) is mapped to either a design system component or a custom component in the Change Required cell — nothing unmapped
- [ ] No new route duplicates an existing route in `ROUTES.md`
- [ ] No new data fetch unit duplicates an existing one in `DATA_LAYER.md`
- [ ] Every data fetch unit row includes every query parameter the server handler requires — including any dispatch or routing params — cross-referenced against the backend LLD's request shape. Do not list only the upstream API params.
- [ ] Every acceptance criterion from the experience card is mapped in §5
- [ ] If Figma URL was absent, §1 clearly states "No Figma URL" and all visual specs in Change Required cells are marked approximate / TBD
- [ ] For every component whose layout or element order is described in the experience card, the §3 subsection visual spec matches that description exactly — do not substitute a different layout direction

Fix any gaps before writing the output file.

---

## Step 7 — Output the LLD

Write the LLD as a markdown document to:

```
<targetRepo>/docs/ai-kit/LLD/ui/<experience-card-filename>-lld.md
```

Create the directory if it does not exist.

Then print a summary:

```
✅ LLD written to <targetRepo>/docs/ai-kit/LLD/ui/<experience-card-filename>-lld.md

Design source:    Figma URL (<url>) / Experience card description only
Backend LLD:      <backendLLDPath> — <N> API endpoints consumed
Files to create:  <N>
Files to modify:  <N>
Design decisions: <N>
AC coverage:      <N covered, M flagged ⚠>

⚠ Gaps / flags:
  - <any acceptance criteria not fully covered>
  - <any visual specs marked approximate / TBD>
  - <any backend LLD endpoint referenced but not fully defined>
```