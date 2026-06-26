---
name: generate-ui-service-card
description: Generate a complete UI Service Card set for any application by reading its codebase. Produces all 7 card files (UI_SERVICE_CARD, UI_MODULE_INDEX, ROUTES, DATA_LAYER, STATE_MANAGEMENT, UI_CODE_PATTERNS, UI_PLATFORM) filled with real project values. Works with any UI technology — JavaScript/TypeScript SPAs, Java JSP/Thymeleaf, C# Blazor/Razor, Python Django/Flask templates, Flutter, or others. Triggers on phrases like "generate service card", "create service card", "document this UI", or when a target app directory is provided with intent to generate cards.
---

# Generate UI Service Card Skill — Generate Service Cards for a UI Application

> **Purpose:** Produce a complete service card set for any UI application,
> sufficient for an agentic coding system to generate correct, idiomatic UI code
> without reading the source repo again.
>
> **Scope:** Document the codebase as it exists today. Do not suggest improvements,
> critique architecture, or propose changes unless explicitly asked.

---

## 1. When to Use

| Trigger | Example |
|---------|---------|
| Onboarding a new UI service | App has no cards yet |
| Significant refactor | New router, new state library, new component library |
| Quarterly refresh | Cards have drifted from codebase |
| Post-incident gap | Incident exposed a missing or wrong section |

---

## 2. Inputs

```
<source_repo_path>
```

| Input | Required | Notes |
|-------|----------|-------|
| `source_repo_path` | Yes | Path to the UI repo root cloned locally — this is the codebase to scan |

**If `source_repo_path` is not provided, stop immediately and ask:**
> "Please provide the path to the source repo you want to scan (e.g. `/path/to/my-app`)."

Do not proceed until the user supplies an explicit path.

**Derived automatically:**
- Service name — `basename(source_repo_path)`
- Source branch — `git -C <source_repo_path> rev-parse --abbrev-ref HEAD`
- Output directory — always `<source_repo_path>/docs/ai-kit/service-cards/ui/` — created if it does not exist

---

## 3. Outputs

All files written under `<source_repo_path>/docs/ai-kit/service-cards/ui/`.

| File | What it captures |
|------|-----------------|
| `UI_SERVICE_CARD.md` | Identity, boundaries, data contracts summary, companion file index. The file an engineer reads first. |
| `UI_MODULE_INDEX.md` | Capability catalogue — every feature mapped to its Page/Component/Hook/Store with Key Decisions and call graphs. |
| `ROUTES.md` | Full route registry, auth guard, navigation patterns, route constants. |
| `DATA_LAYER.md` | Every fetch unit the app calls: endpoint, cache key, auth injection, dependency guard, return shape. Mutation pattern. Shared HTTP client. |
| `STATE_MANAGEMENT.md` | Store shapes, actions, init order, browser storage keys, server data cache strategy. |
| `UI_CODE_PATTERNS.md` | Naming conventions, directory structure, real code excerpts per pattern category, design system map, test skeletons. |
| `UI_PLATFORM.md` | Runtime stack, build scripts, key dependencies, env vars, feature flags, a11y, monitoring. |

---

## 4. Process

### Phase 1 — Locate & Scaffold (~2 min)

Before reading any implementation file, build a map of the codebase. Scans in Phase 2 work from this map — no browsing.

**Step 1: Stack detection**

LS the repo root first, then read whichever manifest file is present.

**Build tool** — identify by manifest filename found at root:

| Manifest file | Build tool | Language |
|---------------|------------|----------|
| `package.json` | npm / yarn / pnpm | JS / TypeScript |
| `pom.xml` or `build.gradle` | Maven / Gradle | Java |
| `*.csproj` | MSBuild | C# |
| `pubspec.yaml` | Flutter CLI | Dart |
| `pyproject.toml` / `requirements.txt` | pip / Poetry | Python |

If no manifest at root, LS one level deeper — monorepos may nest apps.

**UI framework** — read from dependencies in the manifest:

| Marker | Framework |
|--------|-----------|
| `"react"` + `"next"` in dependencies | Next.js (React, SSR-capable) |
| `"react"` + `"react-router-dom"` (no `next`) | React SPA |
| `"vue"` + `"nuxt"` | Nuxt (Vue, SSR-capable) |
| `"vue"` (no `nuxt`) + `"vue-router"` | Vue SPA |
| `"@angular/core"` | Angular |
| `"svelte"` + `"@sveltejs/kit"` | SvelteKit |
| `*.jsx` / `*.tsx` files without above | React SPA (infer) |
| `Pages/*.razor` + `*.csproj` | Blazor |
| `Views/*.cshtml` + `*.csproj` | ASP.NET MVC Razor |
| `templates/*.html` + `requirements.txt` | Django / Flask |
| `lib/*.dart` + `pubspec.yaml` | Flutter |

**Router** — read from dependencies:

| Package | Router |
|---------|--------|
| `next` | Next.js file-based (pages or app router) |
| `react-router-dom` | React Router |
| `vue-router` | Vue Router |
| `@angular/router` | Angular Router |
| `@sveltejs/kit` | SvelteKit file-based |

**State management** — read from dependencies:

| Package | State management |
|---------|-----------------|
| `zustand` | Zustand |
| `@reduxjs/toolkit` | Redux Toolkit |
| `jotai` | Jotai atoms |
| `recoil` | Recoil |
| `pinia` | Pinia (Vue) |
| `@ngrx/store` | NgRx (Angular) |
| None detected | React Context / built-in primitives |

**Data fetching** — read from dependencies:

| Package | Pattern |
|---------|---------|
| `@tanstack/react-query` or `react-query` | TanStack Query hooks |
| `swr` | SWR hooks |
| `axios` (no query lib) | Axios instance + manual state |
| `@apollo/client` | Apollo GraphQL |
| None of the above | Native `fetch` / server-rendered |

**Test framework** — read from dev dependencies. If absent, note `none detected`.

**Language + runtime version** — check `.nvmrc`, `.node-version`, `engines` in `package.json`, or `tsconfig.json → compilerOptions.target`.

---

**Resolution — classify each field before proceeding:**

| Status | Meaning | Action |
|--------|---------|--------|
| `CONFIRMED` | Detected unambiguously from manifest or config | Record and continue |
| `INFERRED` | Best guess from indirect evidence | Record with note — e.g. `React Router (inferred from file patterns)` |
| `AMBIGUOUS` | Two or more equally likely values | Stop and ask |
| `NOT FOUND` | No evidence found after exhausting all sources | Stop and ask |

**If any field is `AMBIGUOUS` or `NOT FOUND`, stop immediately and ask before reading any more files.** Batch all unresolved fields into a single message.

Record all detected fields in `UI_PLATFORM.md → §1 Runtime`. Stack drives layer names, file patterns, and grep idioms for every scan that follows.

---

**Step 2: Codebase map**

Run a structured search before reading any implementation file. Goal: know where things live before reading what they say.

```
Search order:

1. LS top-level source directories — identify the layer structure

2. Grep for routing entry points using the framework detected in Step 1:

   File-based routes (Next.js pages, Nuxt, SvelteKit):
     List all files in pages/, app/, or routes/ excluding:
     - Next.js: _app*, _document*, _error*, api/, layout.*, loading.*, error.*
     - SvelteKit: +layout.*, +error.*

   Config-based routes (React Router, Vue Router, Angular):
     grep: createBrowserRouter|createHashRouter|Routes>|<Route  (React Router)
           createRouter|routes:\s*\[  (Vue Router)
           RouterModule.forRoot|Routes  (Angular)

   Auth guard / middleware:
     grep: middleware\.ts|middleware\.js  (Next.js)
           beforeEach|navigationGuard  (Vue Router)
           CanActivate|AuthGuard  (Angular)
           withAuthenticationRequired|useAuth  (React)

3. Grep for data fetch units:
   TanStack Query:   useQuery\|useMutation\|useInfiniteQuery
   SWR:              useSWR\b\|useSWRMutation
   Apollo:           useQuery\|useLazyQuery\|useMutation  (from @apollo)
   Axios / fetch:    axios\.create\|useAxios\|useFetch\|async function.*fetch
   Angular HttpClient: HttpClient\|this\.http\.

4. Grep for state / store files:
   Zustand:          create<\|create(  from zustand
   Redux:            createSlice\|configureStore\|createAsyncThunk
   Jotai:            atom(\|useAtom\|atomFamily
   Pinia:            defineStore
   Context:          createContext\|useContext\|\.Provider>
   NgRx:             createReducer\|createAction\|createSelector

5. Glob for naming patterns (framework-agnostic):
   *page*, *Page*, *view*, *View*, *screen*, *Screen*    → UI pages / routes
   *component*, *Component*, *widget*, *Widget*          → UI components
   *hook*, *Hook*, use[A-Z]*, composable*                → data / behaviour hooks
   *store*, *Store*, *slice*, *Slice*, *context*, *Context* → state management
   *service*, *Service*, *api*, *Api*, *client*, *Client* → API layer
   *util*, *Utils*, *helper*, *Helper*                   → pure utilities
   *.test.*, *.spec.*                                    → test files

6. Grep for optional feature signals (drives scans 2.14–2.16):
   WebSocket / SSE:  new WebSocket\|useWebSocket\|EventSource\|socket\.connect\|\.on\('message'\|io(
   i18n:             i18next\|useTranslation\b\|formatMessage\|defineMessages\|react-intl\|vue-i18n\|@angular/localize
   Analytics:        analytics\.track\|gtag(\|amplitude\.track\|mixpanel\.track\|posthog\.capture\|segment\.track

   Note which signals were found — these determine whether scans 2.14–2.16 run.

7. Note clusters — directories with 3+ related files are a layer.
```

Produce a **codebase map** grouping files by purpose, with full paths and file counts.

---

**Step 3: Scaffold output files**

Create the output directory and all 7 empty card files listed in §3.

---

### Phase 2 — Code Scan (~5–10 min)

Work from the codebase map. Read only the files the map identified.

**Posture:** Document what exists. If auth is not enforced on a route, write `none detected` — not "missing auth". Do not flag gaps as problems.

---

#### 2.1 Source structure
**Files:** Top-level directories (already in codebase map — no new search).
**Extract:** Layer names, paths, file counts.
**Populates:** `UI_SERVICE_CARD.md → § Source Structure`, `UI_MODULE_INDEX.md → § Module Taxonomy`.

---

#### 2.2 Route registry
**Files:** Pages / views cluster.
**Refine with grep** using the router detected in Step 1:

| Router | Signal to extract route paths |
|--------|------------------------------|
| Next.js (pages) | Filename relative to `pages/` → normalise `[param]` to `:param` |
| Next.js (app) | Filename relative to `app/` — collect all `page.tsx` paths |
| React Router | `path=` or `path:` values in router config |
| Vue Router | `path:` values in `routes` array |
| Angular | `path:` values in `Routes` array |
| SvelteKit | Filename relative to `routes/` → normalise `[param]` to `:param` |

**Extract per route:** URL path, rendering file/component, path params, query params accepted (scan for `useSearchParams`/`useParams`/`$route.query`/`ActivatedRoute.queryParams`), auth required (yes/no, evidence), pre-load requirements.

**Populates:** `ROUTES.md → Route Registry`, `UI_SERVICE_CARD.md → § Routes (Quick Reference)`, `UI_MODULE_INDEX.md § Capability Catalogue` (Page row per capability).

---

#### 2.3 Auth guard mechanism
**Files:** Root wrapper, middleware, router config, route wrapper components.
**Refine with grep:**

| Framework | Grep pattern |
|-----------|-------------|
| Next.js | `middleware\.ts\|middleware\.js\|getServerSideProps.*redirect\|useEffect.*router\.push.*login\|withAuth` |
| React | `<PrivateRoute\|<AuthGuard\|<RequireAuth\|useNavigate.*login\|Redirect.*login` |
| Vue | `router\.beforeEach\|navigationGuard\|meta.*requiresAuth` |
| Angular | `CanActivate\|AuthGuard\|canActivate` |
| SvelteKit | `load.*redirect\|hooks\.server` |
| Fallback | `isAuthenticated\|isLoggedIn\|accessToken\|authToken` near route/redirect patterns |

**Extract:** Mechanism type (global wrapper / middleware / navigation guard / route wrapper), file path, guard expression copied verbatim, what an unauthenticated request does (redirect to `/login`, 401, etc.).

**Populates:** `ROUTES.md → Auth Guard Pattern`.

---

#### 2.4 Rendering strategy and env convention
**Files:** Framework config file (`next.config.js`, `vite.config.ts`, `nuxt.config.ts`, etc.); representative page/route files.
**Extract:**
- Rendering strategy: look for `output: 'export'` / `ssr: false` (SSG), server-side data-loading functions (`getServerSideProps`, `load({fetch})`, `asyncData`, `server.ts` loaders) (SSR/hybrid), or neither (CSR).
- Env var convention: read `.env.sample`, `.env.example`, or `.env.local.example`. Identify the browser-accessible prefix (e.g. `NEXT_PUBLIC_`, `VITE_`, `REACT_APP_`).

**Populates:** `UI_PLATFORM.md → §1 Runtime (rendering strategy)`, `UI_PLATFORM.md → §3 Environment Variables`.

---

#### 2.5 Data fetch layer ⭐
**Files:** Hooks / composables / services / store actions cluster.
**Refine with grep** using the data fetch approach detected in Step 1.

**Extract per fetch unit:**

| Field | What to capture |
|-------|----------------|
| Name | Function / hook / method name |
| HTTP method | GET / POST / PUT / PATCH / DELETE |
| Endpoint | URL template, e.g. `/api/v3/customers/${id}` |
| Cache key | TanStack Query key tuple, SWR key string, or `n/a` |
| Auth injection | Where the token comes from: parameter / store-internal / central interceptor / cookie (HTTP-only) |
| Dependency guard | Condition that must be true before the fetch fires (e.g. `enabled: !!customerId`) |
| Return shape | Fields the consumer receives (top-level keys; nest if > 3 levels deep) |

**Also find:** The shared HTTP client or interceptor (a file that creates a configured `axios` instance, wraps `fetch`, or adds auth headers centrally). If found, auth injection pattern = "central interceptor / instance".

**Also find:** The query client / global fetch library config — where default `staleTime`, `gcTime` (formerly `cacheTime`), `retry` are set.

**Populates:** `DATA_LAYER.md → Fetch Layer Summary` (one row per fetch unit), `DATA_LAYER.md → Standard Fetch Pattern`, `DATA_LAYER.md → Auth Token Injection`, `UI_MODULE_INDEX.md → Capability Catalogue` (Hook/Service row), `UI_SERVICE_CARD.md → §3 Data Contracts`.

> **This is the most important scan.** An AI generating a new data-fetching hook that has to guess at the cache key shape, dependency guard pattern, or auth injection method will produce incorrect or non-idiomatic code.

---

#### 2.6 Mutation pattern
**Files:** Any fetch unit files with write operations (POST/PUT/PATCH/DELETE).
**Extract:** How cache is invalidated after a mutation (key invalidation / navigate away / manual update), whether optimistic updates are used (update UI before server confirms, rollback on failure), how mutation errors are surfaced (inline state / toast / thrown), the mutation call shape.

**Populates:** `DATA_LAYER.md → Standard Mutation Pattern`, `DATA_LAYER.md → Error Handling Posture`.

---

#### 2.7 State management
**Files:** Stores / context / slice files cluster.
**Refine with grep** using the state library detected in Step 1.

**Extract per store / context:**

| Field | What to capture |
|-------|----------------|
| Name | Store / context / slice name |
| File path | Relative path |
| State shape | Every field with its type (infer from `interface`, `type`, or initial value) |
| Actions / mutations | Every function that modifies state (with signature) |
| Computed / selectors | Getters, selectors, computed properties |
| Initialization | Trigger (mount / explicit call / lazy), source (API / storage / props), sync or async |
| Browser storage | `localStorage` / `sessionStorage` / cookie keys used — note key name and shape |
| Dependencies | Which other stores / contexts this one reads from |

**Also find:** The root component or app shell — trace the store initialization order.

**Populates:** `STATE_MANAGEMENT.md` (one `## N. StoreName` per store), `UI_MODULE_INDEX.md → Capability Catalogue` (Store row), `UI_SERVICE_CARD.md → §3 Data Contracts (Storage Keys)`.

---

#### 2.8 Browser storage keys
**Files:** All store files (already identified in 2.7), plus grep across the codebase.
**Refine with grep:** `localStorage\.setItem\|localStorage\.getItem\|sessionStorage\.\|Cookies\.set\|cookie\.set`

**Extract per key:** Key string, type of storage, value shape, set-by (which store/component), read-by (which store/component), persistence duration.

**Populates:** `STATE_MANAGEMENT.md → Browser Storage Key Registry`, `UI_PLATFORM.md → §7 Browser Storage`.

---

#### 2.9 Capabilities (page → component → hook → store)
**Files:** Each page/route file identified in 2.2, plus the components it imports.
**Extract per capability:**

| Field | What to capture |
|-------|----------------|
| Name | User-facing purpose (not file name) |
| Domain | Feature group (e.g. Auth, Dashboard, Reseller Management) |
| Page | File path |
| Component hierarchy | Top-down nesting: `PageComponent` → `PanelComponent` → `ItemComponent` |
| Fetch units | All hooks / services called (directly or via sub-components) |
| Stores / contexts | All stores and contexts read |
| Feature flags | Conditional renders based on flag SDK or API response field |
| Key Decisions | See below |

**Key Decisions per capability:**
- **Rendering:** CSR / SSR / SSG / hybrid
- **Auth dependency:** Which store field must be non-null before this page renders
- **Optimistic update:** Used or not
- **Fail-posture:** Open (degraded UI shown) or closed (hard error / redirect)
- **Stability:** STABLE / IN FLUX

For each decision emit an `**Enforcement:**` citation naming the guard / HOC / middleware that enforces it. If none exists, write `not enforced — relies on code review`.

**Populates:** `UI_MODULE_INDEX.md → Capability Catalogue` (one `### CapabilityName` per capability), `UI_MODULE_INDEX.md → Cross-Reference Indexes`.

---

#### 2.10 Form patterns
**Files:** 3–5 representative form components.
**Refine with grep:** `useForm\|<form\|handleSubmit\|FormProvider\|<Form\|onSubmit\|validationSchema\|zodResolver\|yupResolver`

**Extract:** Form library (or none), where schemas live, when validation runs (on blur / on submit / on change), how errors display, submission UX (button disable / spinner / optimistic).

**Populates:** `UI_CODE_PATTERNS.md → B.6 Form Patterns`.

---

#### 2.11 Cross-cutting concerns
**Files:** Root / app shell component, shared components directory.
**Refine with grep:** `ErrorBoundary\|Suspense\|ToastProvider\|NotificationProvider\|Layout\|<Layout\|useToast\|useNotification`

**Extract per component:**

| Field | What to capture |
|-------|----------------|
| Name | Component name |
| Purpose | One line |
| Scope | Global or route-specific |
| Reads | Props, context, or store it reads |
| Provides | What it injects or renders into children |

**Populates:** `UI_MODULE_INDEX.md → §6 Cross-Cutting Modules`, `UI_CODE_PATTERNS.md → B.7 Error Handling in Components`.

---

#### 2.12 Test patterns
**Files:** 2–4 representative test files (one fetch unit test, one component test if present).
**Extract:** Framework, component testing utility, mocking approach for fetch/API, test naming convention.

**Populates:** `UI_CODE_PATTERNS.md → B.8 Test Patterns`.

---

#### 2.13 Design system and component library usage
**Files:** 3–5 representative component files (scan for UI library imports).
**Refine with grep** for the library detected in Step 1 (e.g. `@react-spectrum`, `@mui`, `antd`, `@chakra-ui`, `shadcn`, `@adobe/react-spectrum`).

**Extract:** Import path pattern, which components are used where, what a11y the library provides automatically, what must be added manually.

**Populates:** `UI_CODE_PATTERNS.md → B.5 Design System`, `UI_PLATFORM.md → §1 Runtime`.

---

#### 2.14 Real-time / WebSocket
**Skip if:** No WebSocket/SSE signal found in codebase map step 6 — omit `DATA_LAYER.md → Real-time / WebSocket Pattern` entirely.

**Files:** Files containing WebSocket or SSE usage (from codebase map).
**Refine with grep:** `new WebSocket\|useWebSocket\|EventSource\|socket\.on\|socket\.emit\|\.subscribe\|io(`

**Extract:**
- Transport type: WebSocket / SSE / long-poll
- Connection owner: which component or hook creates and owns the connection
- Auth injection: how credentials are attached (query param / cookie / post-connect message)
- Subscribe/unsubscribe pattern (verbatim code snippet)
- Cache integration: does incoming data flow into the query cache or a store?
- Reconnection strategy: automatic / manual / none
- Error and disconnect handling

**Populates:** `DATA_LAYER.md → Real-time / WebSocket Pattern`.

---

#### 2.15 Internationalization (i18n)
**Skip if:** No i18n signal found in codebase map step 6 — omit `UI_CODE_PATTERNS.md → B.9 Internationalization` entirely.

**Files:** i18n config file, one representative translation file, 2–3 files that call the translation function.
**Refine with grep:** `i18next\|useTranslation\|formatMessage\|defineMessages\|t(\|<Trans\|locale`

**Extract:**
- Library name and version
- Translation file location and format (JSON / YAML / PO)
- Supported locales and locale source (URL param / browser / user store)
- Key naming convention (e.g. `domain.component.key` or flat)
- Translation access pattern (verbatim call)
- Pluralization approach
- Date / number / currency formatting approach

**Populates:** `UI_CODE_PATTERNS.md → B.9 Internationalization`.

---

#### 2.16 Analytics events
**Skip if:** No analytics signal found in codebase map step 6 — omit `UI_PLATFORM.md → §10 Analytics Event Schema` entirely.

**Files:** Analytics initialisation file, 2–3 files that fire events.
**Refine with grep:** `analytics\.track\|gtag(\|amplitude\.track\|mixpanel\.track\|posthog\.capture\|segment\.track`

**Extract:**
- Library and initialisation location
- Consent gating: is tracking conditional on cookie consent?
- Event naming convention (e.g. `NOUN_VERB` or `category:action`)
- Required properties on every event (user ID, session ID, etc.)
- Where events are fired (page mount / user action / both)
- Verbatim canonical fire pattern

**Populates:** `UI_PLATFORM.md → §10 Analytics Event Schema`.

---

### Phase 3 — Config Scan (~2–3 min)

Work from the configuration files identified in the codebase map.

| Scan | Read | Populates |
|------|------|-----------|
| 3.1 Package manifest | `package.json` (already read in Phase 1) | `UI_PLATFORM.md → §2 Build` (all scripts + key deps) |
| 3.2 Framework / build config | `next.config.*` / `vite.config.*` / `nuxt.config.*` / `angular.json` / etc. | `UI_PLATFORM.md → §1 Runtime`, `UI_PLATFORM.md → §2 Build (plugins)` |
| 3.3 Env vars | `.env.sample`, `.env.example`, `.env.local.example` — never `.env` | `UI_PLATFORM.md → §3 Environment Variables` |
| 3.4 README / setup docs | `README.md` | `UI_PLATFORM.md → §4 Local Development` |
| 3.5 CI config (optional) | `.github/workflows/*.yml`, `Jenkinsfile`, `.gitlab-ci.yml` — look for test/lint/deploy steps | `UI_PLATFORM.md → §8 Accessibility Compliance`, `§9 Monitoring` |

---

### Phase 4 — Assembly (~5–8 min)

1. Compile all scan outputs into the 7 card files.

2. **Formatting rules:**

| Rule | Detail |
|------|--------|
| Tables over prose | Use tables wherever content is list-like — cards are machine-read as well as human-read |
| Real code excerpts | Copy actual code from the codebase verbatim — do not invent synthetic examples |
| Short snippets | 3–8 lines max — show the idiom, not a full implementation |
| Missing data | Write `NOT CAPTURED — requires [what]` — no placeholder text |
| Cross-references | Anchor links only — no "see X.md" pointers |

3. **Heading patterns** (keeps anchors stable and machine-resolvable):

| Card | Heading pattern |
|------|----------------|
| `ROUTES.md` | `### /<path> — <Purpose>` per route |
| `DATA_LAYER.md` | `### <N>. <FetchUnitName> — <one-line desc>` per fetch unit |
| `STATE_MANAGEMENT.md` | `### <N>. <StoreName>` per store |

4. **`UI_SERVICE_CARD.md` required sections** — generate in this order:

| Section | Content | Source scan |
|---------|---------|-------------|
| Frontmatter | `service`, `owner`, `stack`, `rendering`, `last_updated` | Phase 1 |
| `## 1. What We Do` | One paragraph: purpose, users, rendering strategy, backends, auth provider | README + Phase 1 |
| `## 2. What We Explicitly Do Not Do` | 3–5 hard-boundary bullets | 2.5, 2.9 |
| `## 3. Data Contracts → ### APIs We Consume` | Fetch unit, Method, Endpoint, Purpose, Stability — one row per fetch unit | 2.5 |
| `## 3. Data Contracts → ### State We Own Client-Side` | Key, Storage, Owner, Contents, TTL — **omit if no browser storage** | 2.8 |
| `## 4. Source Structure` | Layer, Path, Files | 2.1 |
| `## 5. Module Index (Summary)` | File root patterns; domain → capability-count table with `UI_MODULE_INDEX.md` anchors | 2.9 |
| `## 6. What You Need to Know` | Fragile Areas, Known Constraints, Test Coverage, LLD Hints | 2.11, 2.12 |
| `## Companion Files` | Fixed table below — always the last section | — |

**Companion Files table — fixed for every UI service:**

| File | When to Load | Purpose |
|------|--------------|---------|
| `UI_MODULE_INDEX.md` | Impact analysis, LLD, code gen (load FIRST) | Capability catalogue, Key Decisions, file paths |
| `ROUTES.md` | LLD, routing work, code gen | Route registry, auth guard, navigation contracts |
| `DATA_LAYER.md` | LLD, code gen | Fetch library, cache strategy, mutations, auth injection |
| `STATE_MANAGEMENT.md` | LLD, code gen (any state change) | All stores/contexts: shapes, actions, init order, storage keys, cache strategy |
| `UI_CODE_PATTERNS.md` | Code gen | Naming, component patterns, design system, forms, tests |
| `UI_PLATFORM.md` | Impact analysis, deployment planning, code gen | Runtime, build scripts, key dependencies, env vars, perf budget, a11y, monitoring |

---

5. **`UI_MODULE_INDEX.md` required sections** — generate in this order:

| Section | Content |
|---------|---------|
| `## Overview` | One paragraph: what this app owns, its role, who uses it |
| `## Module Taxonomy` | Table: Layer \| File pattern \| Count |
| `## Capability Taxonomy` | Table: Domain \| Capabilities (comma-separated) |
| `## Capability Catalogue` | One `### <CapabilityName>` per capability — Page, Components, Hierarchy, Hook/Service, Store, Form, Feature flags, Key Decisions (with Enforcement), Call graph |
| `## Cross-Reference: Route → Capability` | Table: Route \| Capability |
| `## Cross-Reference: Fetch Unit → Capabilities` | Table: Fetch unit \| Capabilities that use it |
| `## Cross-Reference: Store → Capabilities It Gates` | Table: Store \| Capabilities — omit if no stores |
| `## Cross-Reference: Browser Storage Keys` | Table: Key \| Storage type \| Set by \| Read by |
| `## Cross-Reference: Feature Flag → Capabilities` | Table: Flag \| Source \| Default \| Capabilities it gates — omit if no flags found in 2.9 |
| `## Cross-Cutting Modules` | Table: Module \| Purpose \| Used by |
| `## Type Inventory` | Counts: pages, stores, fetch units, reusable components, form schemas, util files |
| `## Forbidden / Do-Not-Add Rules` | Project-specific rules derived from 2.7, 2.9 (e.g. "no direct API calls from page components") |

---

6. **`ROUTES.md` required sections** — generate in this order:

| Section | Content | Source scan |
|---------|---------|-------------|
| Overview table | Router, auth guard mechanism, auth guard file, root redirect, unauthenticated redirect, layout hierarchy | 2.2, 2.3 |
| Route Registry | One row per route: path, file/component, path params, query params, auth required, pre-load requirements | 2.2 |
| Auth Guard Pattern | Mechanism type, file path, verbatim guard code snippet, rules for new pages | 2.3 |
| Navigation Patterns | Verbatim programmatic navigation call; state-passing strategy table; back-navigation pattern | 2.2 |
| Route Constants | Verbatim constants object — or note `inline strings — no constants file detected` | 2.2 |
| Error Routes | 404, auth failure, access denied, unhandled render error: component + behavior | 2.3, 2.11 |
| Notes for Code Generation | Rules affecting every new page: auth guard required?, canonical param names, how IDs are passed | 2.2, 2.3 |

---

7. **`DATA_LAYER.md` required sections** — generate in this order:

| Section | Content | Source scan |
|---------|---------|-------------|
| Overview table | Data fetch library, auth injection pattern, default staleTime/gcTime, error surface, optimistic update posture, SSR pre-fetching | 2.5 |
| Fetch Layer Summary | One row per domain: fetch unit name, endpoint, cache key pattern, auth injection | 2.5 |
| Standard Fetch Pattern | Verbatim fetch unit — annotate (1) auth injection, (2) dependency guard, (3) return shape | 2.5 |
| Standard Mutation Pattern | Verbatim mutation unit — annotate (1) payload → call, (2) cache update after success, (3) error surface | 2.6 |
| Auth Token Injection | Verbatim pattern labelled Option A (parameter) / B (store-internal) / C (interceptor) | 2.5 |
| Error Handling Posture | Verbatim fetch-error and mutation-error display patterns | 2.6, 2.11 |
| Loading States | Verbatim loading pattern (skeleton / spinner / async boundary) | 2.11 |
| Real-time / WebSocket Pattern | Transport, auth, subscribe/unsubscribe, reconnection — **omit entire section if scan 2.14 found nothing** | 2.14 |
| Adding a New Read Unit — Checklist | 8-step checklist: identify capability → name unit → define cache key → set dependency guard → set freshness in STATE_MANAGEMENT.md → validate response shape → add to Fetch Layer Summary → add to UI_MODULE_INDEX.md | — |

---

8. **`STATE_MANAGEMENT.md` required sections** — generate in this order:

| Section | Content | Source scan |
|---------|---------|-------------|
| Overview table | Library, store count, init chain enforcement (reactive / manual), persisted stores | 2.7 |
| `## N. StoreName` per store | State shape (typed), all actions with signatures, computed values, init trigger + verbatim code, storage persistence table, verbatim consumer pattern, rules | 2.7 |
| Initialization Order & Boot Dependency Chain | Dependency diagram; verbatim guard expressions copied from fetch units | 2.7 |
| Cross-Store Dependencies | Table: Store \| Reads from \| Required field \| Behavior while waiting | 2.7 |
| Storage Key Registry | Every browser storage key: key, type, owner, contents, persistence, shape version, migration strategy | 2.8 |
| Cache Strategy | Per domain: staleTime, gcTime, invalidation trigger — from global query client config | 2.5 |

---

9. **`UI_CODE_PATTERNS.md` required sections:**

**Part A — copy verbatim from the card template.** Part A (sections A.1–A.10) contains the versioned general UI standards. Do not regenerate it — copy the template content exactly as written. If the scanned codebase demonstrably violates a rule (e.g. uses `any` throughout, relies on inline styles), add a `> ⚠ Project exception: <note>` blockquote immediately below the relevant rule. Do not delete or weaken rules.

**Part B — generate from scans:**

| Section | Content | Source scan |
|---------|---------|-------------|
| B.1 Naming Conventions | Table: Kind \| Pattern \| Example — use real file names from codebase | 2.1, 2.9 |
| B.2 File / Directory Structure | Annotated directory tree (`find` output) | 2.1 |
| B.3 Data Fetch Library Patterns | Verbatim query call + verbatim mutation call | 2.5, 2.6 |
| B.4 State Library Patterns | Verbatim store read + verbatim store write/action call | 2.7 |
| B.5 Design System | Import path pattern; scenario → component table from actual imports | 2.13 |
| B.6 Form Patterns | Verbatim canonical form; validation, error display, submission UX | 2.10 |
| B.7 Error Handling in Components | Verbatim fetch-error and mutation-error display patterns | 2.11 |
| B.8 Test Patterns | Framework stack table; unit test skeleton (hook); unit test skeleton (component) | 2.12 |
| B.9 Internationalization | i18n library, locale source, key convention, access pattern — **omit entire section if scan 2.15 found nothing** | 2.15 |

---

10. **`UI_PLATFORM.md` required sections** — generate in this order:

| Section | Content | Source scan |
|---------|---------|-------------|
| §1 Runtime | Framework, router, rendering strategy, build tool, language, runtime, state library, data fetch library, component library, form library | Phase 1, 2.4 |
| §2 Build | Scripts block (from `package.json`), key dependencies table, build plugins, bundle analysis | 3.1, 3.2 |
| §3 Environment Variables | One row per var: name, browser-accessible (yes/no), purpose, required, example value | 3.3, 2.4 |
| §4 Local Development | Setup steps from README, HTTPS config, auth in dev | 3.4 |
| §5 Routing Table (Quick Reference) | URL, file/component, auth required — brief; `ROUTES.md` is authoritative | 2.2 |
| §6 Feature Flags | UI-evaluated vs server-authoritative; known flags table | 2.9 |
| §7 Browser Storage (Quick Reference) | Key, type, owner, contents, TTL | 2.8 |
| §8 Accessibility Compliance | WCAG target, automated tooling, known exceptions — from CI config or a11y deps | 3.5, 2.13 |
| §9 Monitoring & Error Tracking | Tool, purpose, config env var — from deps/CI | 3.5 |
| §10 Analytics Event Schema | Library, naming convention, required properties, fire pattern — **omit entire section if scan 2.16 found nothing** | 2.16 |
| §11 Links & Runbooks | Resource, URL/location — from README | 3.4 |

---

11. **Before writing files** — run the [§5 Quality Gates](#5-quality-gates) checklist.

12. **Write `CHANGELOG.md` entry:**
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
- [ ] Every capability in `UI_MODULE_INDEX ## Capability Catalogue` has: page, component(s), fetch unit(s), store(s), Key Decisions with Enforcement citation
- [ ] Every capability with >1 data dependency has a **Call graph** step list

**Completeness — data layer**
- [ ] Every fetch unit documented: endpoint, cache key, auth injection, dependency guard, return shape
- [ ] Shared HTTP client / interceptor documented (or marked `none — fetch calls are direct`)
- [ ] Mutation pattern documented: cache invalidation method, error surface

**Completeness — state**
- [ ] Every store documented: state shape, all actions, initialization trigger, browser storage keys
- [ ] Initialization order documented (even if just one store)

**Completeness — routes**
- [ ] Every route has auth-required flag with evidence
- [ ] Auth guard pattern documented with actual code excerpt

**Completeness — code patterns**
- [ ] `UI_CODE_PATTERNS.md` Part A is copied verbatim from the template (not regenerated); any project exceptions are noted with `> ⚠ Project exception:`
- [ ] `UI_CODE_PATTERNS.md` Part B (B.1–B.8) has real code excerpts — no synthetic examples
- [ ] Design system component map populated from actual imports (not assumed)

**Completeness — optional sections**
- [ ] `DATA_LAYER.md → Real-time / WebSocket Pattern` is either populated (scan 2.14 found code) or the section is absent entirely — never a blank placeholder
- [ ] `UI_CODE_PATTERNS.md → B.9 Internationalization` is either populated (scan 2.15 found code) or the section is absent entirely
- [ ] `UI_PLATFORM.md → §10 Analytics Event Schema` is either populated (scan 2.16 found code) or the section is absent entirely

**Cross-references**
- [ ] Every `UI_SERVICE_CARD.md → §3 Fetch Contracts` row has a working anchor link into `DATA_LAYER.md`
- [ ] `UI_SERVICE_CARD.md → ## Companion Files` table is present as the last section

---

## 6. Report

```
✅ Service card written to: <source_repo_path>/docs/ai-kit/service-cards/ui/

Files generated:
  UI_SERVICE_CARD.md     — identity, boundaries, data contracts
  UI_MODULE_INDEX.md     — <N> capabilities across <M> domains
  ROUTES.md              — <N> routes documented
  DATA_LAYER.md          — <N> fetch units, auth injection: <pattern>
  STATE_MANAGEMENT.md    — <N> stores, <N> browser storage keys
  UI_CODE_PATTERNS.md    — naming conventions + real code excerpts
  UI_PLATFORM.md         — runtime, build, <N> env vars, dependencies

⚠ Gaps (NOT CAPTURED):
  - <file>: <field> — <reason>

Next steps:
  1. Fill the ⚠ gaps manually — they require human knowledge not visible in source
  2. Review Key Decisions in UI_MODULE_INDEX.md — these require human judgment
  3. Verification pass: pick one capability and try to describe a code change
     using only the cards. Any step that requires reading source = a gap to fill.
```
