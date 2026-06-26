---
service: {{SERVICE_NAME}}
owner: {{OWNER_EMAIL}}
status: production
repo: {{REPO_URL}}
tier: {{1|2|3}}
ui: true
stack: {{COMPONENT_FRAMEWORK}} / {{ROUTER}} / {{DATA_FETCH_LIBRARY}}
rendering: {{RENDERING_STRATEGY}}
last_updated: {{DATE}}
---

# Service Card — {{SERVICE_NAME}} ({{SHORT_NAME}})

> This card enables Claude to generate correct components, hooks, routes, and tests
> for this UI application — without reading the source repository.
>
> **Stack:** {{COMPONENT_FRAMEWORK}} · {{ROUTER}} · {{DATA_FETCH_LIBRARY}} · {{STATE_LIBRARY}} · {{COMPONENT_LIBRARY}}

---

## 1. What We Do

<!--
One paragraph covering:
- What the application is and who uses it
- What problem it solves
- Rendering strategy (CSR / SSR / SSG / hybrid) and why
- What data backends it talks to (BFF proxy, direct API calls, or both)
- What authentication provider it uses

Example:
"{{SERVICE_NAME}} is a {{CSR / SSR}} dashboard that lets {{who}} manage {{what}}.
It proxies all data through {{BFF name}} at {{/api/*}} — no direct browser-to-upstream-API
calls. Authentication is handled by {{IMS / Auth0 / Cognito}} via {{popup / redirect}} flow."
-->

---

## 2. What We Explicitly Do Not Do

<!--
Hard boundaries. Each bullet: what is NOT this UI's responsibility and who owns it instead.

Must answer:
- Does the UI own any business rule validation, or does it only enforce field shapes?
- Does it call upstream APIs directly from the browser, or only through a proxy?
- Does it own auth (login screen, token management) or does a wrapper/platform handle it?
- What is explicitly out of scope for this UI (payments, admin tools, mobile app, etc.)?
-->

- **Does NOT ...** — {{who does instead}}

---

## 3. Data Contracts

> **Cross-reference convention:** every row links directly to its anchor in the companion
> file. "See `DATA_LAYER.md`" pointers are NOT allowed — jump to the exact section.

### APIs We Consume

<!--
One row per logical data operation the UI performs. "Hook/Service/Composable" is the
browser-side function that wraps the call.
-->

| Hook / Composable / Service | Method | Endpoint | Purpose | Stability |
|-----------------------------|--------|----------|---------|-----------|
| [`{{fetchUnitName}}`](DATA_LAYER.md#{{anchor}}) | GET | `{{/api/path}}` | {{purpose}} | STABLE / IN FLUX |

### State We Own Client-Side

<!--
Browser storage used by this application. STATE_MANAGEMENT.md §Storage Key Registry is
the authoritative source; this table is the quick-reference summary.
-->

| Key | Storage | Owner Store/Context | Contents | TTL |
|-----|---------|---------------------|----------|-----|
| `{{key}}` | localStorage / sessionStorage | [`{{StoreName}}`](STATE_MANAGEMENT.md#{{anchor}}) | {{what is stored}} | session / permanent |


## 4. Source Structure

| Layer | Path | Files |
|-------|------|-------|
| {{layer_name}} | `{{path/}}` | {{N}} files |

---

## 5. Module Index (Summary)

File root: `{{PAGE_PATTERN}}` (pages) · `{{COMPONENT_PATTERN}}` (components) · `{{HOOK_PATTERN}}` (hooks) · `{{STORE_PATTERN}}` (stores)

> **Authoritative capability resolver:** [`UI_MODULE_INDEX.md`](UI_MODULE_INDEX.md) — contains the
> full Capability Catalogue (§4), Key Decisions per capability, cross-reference indexes (§5),
> and type inventory (§7). This section is a summary only.

| Domain | # Capabilities | UI_MODULE_INDEX anchor |
|--------|----------------|------------------------|
| {{Domain1}} | | [link](UI_MODULE_INDEX.md#{{anchor}}) |

---

## 6. What You Need to Know Before Planning Work Here

### Fragile Areas

{{List areas where changes have caused regressions or have subtle coupling.
Each item: what the area is, why it's fragile, what to watch out for.
Examples: shared context/store that many components depend on, complex routing guards, third-party component wrappers that are hard to update.}}

### Known Constraints

{{Browser support targets, bundle size budget, a11y requirements, design system guardrails.
Examples: "Must support Chrome 110+, Safari 15+", "Total JS budget: 250 kB gzipped", "WCAG 2.1 AA required on all interactive elements", "All UI components must come from @react-spectrum/s2 — no custom primitives".}}

### Test Coverage

{{Where component/hook test coverage is strong, where it's weak, what E2E scenarios exist.
Examples: "All data-fetching hooks have unit tests; page-level components are untested", "E2E covers happy-path checkout only — edge cases unverified".}}

### LLD Hints

{{Rules for placing new code. Where to add new pages, components, hooks, and stores.
What state management approach to use for different scenarios (local state vs context vs store).}}

- **Flag policy:** {{when must a new feature be flag-gated — e.g. "all new pages behind a feature flag until design sign-off". If no written policy: `Unknown — requires product-owner input` and note the observed convention.}}

---

## Companion Files

| File | When to Load | Purpose |
|------|--------------|---------|
| `UI_MODULE_INDEX.md` | Impact analysis, LLD, code gen (load FIRST) | Capability catalogue, Key Decisions, file paths |
| `ROUTES.md` | LLD, routing work, code gen | Route registry, auth guard, navigation contracts |
| `DATA_LAYER.md` | LLD, code gen | Fetch library, cache strategy, mutations, auth injection |
| `STATE_MANAGEMENT.md` | LLD, code gen (any state change) | All stores/contexts: shapes, actions, init order, storage keys, cache strategy |
| `UI_CODE_PATTERNS.md` | Code gen | Naming, component patterns, design system, forms, tests |
| `UI_PLATFORM.md` | Impact analysis, deployment planning, code gen | Runtime, build scripts, key dependencies, env vars, perf budget, a11y, monitoring |
