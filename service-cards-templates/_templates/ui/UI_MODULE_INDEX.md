# Module Index — {{SERVICE_NAME}}

> **Authoritative capability → code resolver** for the UI layer. Impact Analysis,
> LLD, and code generation start here. Every page, component, hook, and store
> traces back to a ## Capability Catalogue row.
>
> **Last Updated:** {{DATE}}

---

## 1. Workflow — How Each Pipeline Step Uses This File

| Pipeline Step | Sections Read | What it Extracts |
|---------------|---------------|------------------|
| Impact Analysis | ## Capability Taxonomy, ## Capability Catalogue, ## Cross-Reference Indexes | Which capabilities are affected; blast radius |
| LLD / Engineering Plan | ## Module Taxonomy, ## Capability Catalogue, ## Cross-Reference Indexes, ## Type Inventory | File paths for new code; cross-ref anchors; type inventory |
| Code Generation | ## Module Taxonomy, ## Capability Catalogue, ## Type Inventory, ## Forbidden/Do-Not-Add Rules | Layer naming; Key Decisions (state ownership, auth deps, rendering, optimistic update posture) |

### 1.2 Resolution Order — from "I need to add X" to "here is the file path"

1. Identify the capability from ## Capability Taxonomy
2. Jump to its ## Capability Catalogue row
3. Read **Key Decisions** first — rendering strategy, state ownership, auth dependency, fail-posture
4. Read Page / Component / Hook / Store / Util
5. Follow `ROUTES.md` / `DATA_LAYER.md` / `STATE_MANAGEMENT.md` anchors for concrete shapes
6. Only then write code

---

## 2. Module Taxonomy — Layer Categories

<!--
Fill {{PAGE_PATTERN}} etc. with the actual file path patterns for this project.
Examples:
  React/Next.js:   pages/*.tsx, components/**/*.tsx, hooks/use*.ts, contexts/*Context.tsx, utils/*.ts
  Vue/Nuxt:        src/views/*.vue, src/components/**/*.vue, composables/use*.ts, stores/*.ts, utils/*.ts
  Angular:         src/app/**/*.component.ts, src/app/shared/**/*.component.ts, src/app/services/*.service.ts, store/reducers/*.ts
  SvelteKit:       src/routes/*/+page.svelte, src/lib/**/*.svelte, src/lib/stores/*.ts, src/lib/utils/*.ts
-->

File root: `{{PROJECT_ROOT}}`

| Layer | Location pattern | Responsibility | May depend on |
|-------|-----------------|---------------|---------------|
| Page | `{{PAGE_PATTERN}}` | Route entry, layout composition, page-level data setup | Components, Hooks/Composables/Services, Stores |
| Component | `{{COMPONENT_PATTERN}}` | Rendering, local UI state, user interaction | Stores (read-only), Hooks/Composables/Services (local instantiation) |
| Hook / Composable / Service | `{{HOOK_PATTERN}}` | Data fetch, mutations, reactive side effects | Stores (for auth/session), utils |
| Store / Context | `{{STORE_PATTERN}}` | Global state + actions (auth, session, cart, org) | Hooks/Composables/Services (for initial data fetch on mount) |
| Util | `{{UTIL_PATTERN}}` | Pure helpers — no I/O, no side effects, no store access | (none) |

**Forbidden dependency directions:**
- Store must NOT import from Page or Component
- Component must NOT call `fetch()` or the data fetch library directly — all data through Hooks
- Hook must NOT import from Page
- Util must NOT import from any layer above it
- No cross-store writes — stores only mutate their own state

See [UI_SERVICE_CARD.md → ## What We Explicitly Do Not Do](UI_SERVICE_CARD.md#2-what-we-explicitly-do-not-do) for service-level boundaries.

---

## 3. Capability Taxonomy

<!--
Group capabilities by user-facing domain. Each capability gets a ## Capability Catalogue row.
A capability = one user-visible feature or workflow step (not a component or a hook).
-->

| Domain | Capabilities |
|--------|---------------------|
| {{Domain1}} | {{CapabilityName1}}, {{CapabilityName2}} |
| {{Domain2}} | {{CapabilityName3}}, {{CapabilityName4}} |

---

## 4. Capability Catalogue

> Every capability row MUST include **Key Decisions** — rendering strategy, state
> ownership, auth dependency, optimistic update posture, fail-posture, stability.
> These are the invariants an LLM cannot derive from source code.

<!--
Template for one capability row. Copy and fill for each capability.
Remove fields that don't apply (e.g., Feature flags = "none").
-->

### 4.1 {{Capability Name}}

| Field | Value |
|-------|-------|
| Domain | {{Domain}} |
| Page | `{{PageFile}}` → [ROUTES](ROUTES.md#{{route-anchor}}) |
| Component(s) | `{{ComponentName}}` |
| Component hierarchy | `{{PageComponent}} → {{PanelComponent}} → {{ItemComponent}}` — list parent → child nesting order; mark shared components with `(shared)` |
| Hook / Composable / Service | `{{fetchUnitName}}` → [DATA_LAYER](DATA_LAYER.md#{{hook-anchor}}) |
| Store / Context | `{{StoreName}}` → [STATE_MANAGEMENT](STATE_MANAGEMENT.md#{{store-anchor}}) |
| Util(s) | `{{utilFunctionName}}` in `{{utils/file.ts}}` |
| Form / Validation | `{{SchemaName}}` via `{{FORM_LIBRARY}}` (or "none") |
| Feature flags | `{{flag_name}}` — `{{on/off default}}` (or "none") |
| **Key Decisions** | Rendering: {{CSR client-fetch / SSR pre-fetch / static}}; State owner: {{which store}}; Auth dep: {{which store value must be set before this renders}}; Optimistic update: {{yes — rollback via X / no}}; Fail-posture: {{open — degraded UI / closed — hard error}}; Stability: {{STABLE / IN FLUX / DEPRECATED}} |
| **Call graph** | 1. {{Page mounts / user action}} → 2. `{{hookName(params)}}` → 3. `{{DATA_FETCH_LIBRARY}} fetches {{endpoint}}` → 4. {{result flows to component}} (OR: "single downstream — N/A") |
| **Enforcement** | {{What enforces the Key Decision in code — guard expression, enabled flag, middleware, or "not enforced — relies on code review"}} |
| **Scope** | `{{PageName}}` — {{one sentence: what belongs here; what does NOT belong here}} |

<!-- Repeat for each capability -->

---

## 5. Cross-Reference Indexes

### 5.1 Route → Capability

| Route Path | Method / Trigger | Capability |
|------------|------------------|-------------------|
| `{{/path}}` | GET (page load) | |
| `{{/path/:id}}` | user action | |

### 5.2 Hook / Composable / Service → Capabilities

| Hook / Composable / Service | Used by capabilities |
|-----------------------------|---------------------|
| `{{fetchUnitName}}` | {{CapabilityName1}}, {{CapabilityName2}} |

### 5.3 Store / Context → Capabilities It Gates

| Store / Context | Gates capabilities |
|-----------------|-------------------|
| `{{StoreName}}` | {{CapabilityName}} (must be initialized before render) |

### 5.4 Browser Storage Keys

| Key | Storage type | Owner store | Capabilities |
|-----|-------------|-------------|-------------|
| `{{key}}` | localStorage / sessionStorage | `{{StoreName}}` | {{CapabilityName}} |

### 5.5 Feature Flag → Capabilities

<!--
Omit this section if the application has no feature flags.
-->

| Flag | Source (UI-evaluated / server-authoritative) | Capabilities gated |
|------|---------------------------------------------|-------------------|
| `{{flag_name}}` | | |

---

## 6. Cross-Cutting Modules

<!--
Modules that are used across capabilities but don't belong to a single capability row.
Examples: Layout wrapper, error boundary, navigation panel, loading skeleton,
analytics event emitter, i18n provider, toast/notification system.
-->

| Module | Purpose | Location |
|--------|---------|---------|
| Layout wrapper | Page chrome, navigation, footer | `{{components/Layout/Layout.tsx}}` |
| Error boundary | Catch unhandled render errors | `{{components/ErrorBoundary.tsx}}` |
| Loading skeleton | Shared loading placeholder | `{{components/LoadingSkeleton.tsx}}` |
| Toast / notification | User feedback after mutations | `{{components/Toast.tsx}}` |
| Analytics | Event emission | `{{utils/analytics.ts}}` |
| i18n provider | Translations | `{{i18n/index.ts}}` |

---

## 7. Type Inventory

| Layer | Count | Notes |
|-------|-------|-------|
| Pages | | |
| Stores / Contexts | | |
| Hooks / Composables | | |
| Reusable components | | |
| Form schemas | | |
| Util files | | |

---

## 8. Forbidden / Do-Not-Add Rules

<!--
Explicit rules that code review enforces. Fill these in for the project.
The examples below are common defaults — edit to match actual decisions.
-->

- **No new page without a ## Capability Catalogue row and a ROUTES.md entry**
- **No direct `fetch()` in components or pages** — all data through the data fetch library
- **No `{{STATE_LIBRARY}}` write outside the owning store** — stores only mutate their own state
- **No `{{STATE_LIBRARY}}` read before checking initialization guards** (e.g. `!!authToken && !!userId`)
- **No localStorage write outside the owning store** — `STATE_MANAGEMENT.md §Storage Key Registry` is authoritative
- **No plain HTML `<{{interactive_element}}>` when `{{COMPONENT_LIBRARY}}` has an equivalent** — unless justified in CODE_PATTERNS.md §B.5
- **No new capability without an explicit Key Decisions entry** — if it isn't decided, don't ship it
- **No `{{any}}` in TypeScript** — use `unknown` + narrowing or the data library's inferred types
- **No `console.log` in production code** — use the project's logging/analytics utility

---

## 9. Related Files

| File | Relationship |
|------|-------------|
| [UI_SERVICE_CARD.md](UI_SERVICE_CARD.md) | ## Module Index Summary points here; ## Data Contracts links to DATA_LAYER.md directly |
| [ROUTES.md](ROUTES.md) | Anchor targets for ## Capability Catalogue Page rows + ## Cross-Reference Indexes route index |
| [DATA_LAYER.md](DATA_LAYER.md) | Anchor targets for ## Capability Catalogue Hook/Service rows + ## Cross-Reference Indexes fetch-unit index |
| [STATE_MANAGEMENT.md](STATE_MANAGEMENT.md) | Anchor targets for ## Capability Catalogue Store rows + ## Cross-Reference Indexes store index + storage keys; cache strategy |
| [UI_CODE_PATTERNS.md](UI_CODE_PATTERNS.md) | Class/file structure for anything added here |
| [UI_PLATFORM.md](UI_PLATFORM.md) | Runtime, build scripts, key dependencies, env vars, feature flags, performance budget, a11y target |
