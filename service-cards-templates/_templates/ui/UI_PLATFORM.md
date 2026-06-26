# Platform — {{SERVICE_NAME}}

> Operational context: runtime, build, environment variables, routing table, feature
> flags, browser storage, performance budget, accessibility compliance, and monitoring.
>
> **Last Updated:** {{DATE}}

---

## 1. Runtime

| Property | Value |
|----------|-------|
| UI technology | `{{UI_FRAMEWORK_OR_TECHNOLOGY}}` + version |
| Router | `{{ROUTER}}` — or "server-side routing" / "not applicable" if server-rendered |
| Rendering strategy | `{{RENDERING_STRATEGY}}` — `{{CSR / SSR / SSG / ISR / server-rendered (no client fetch) }}` |
| Build tool | `{{BUILD_TOOL}}` — or "not applicable" if server-rendered without a separate build step |
| Language | `{{LANGUAGE}}` `{{version}}` |
| Runtime | `{{RUNTIME + version}}` — e.g. Node.js 20, JVM 17, .NET 8, Python 3.12, or "not applicable" |
| State library | `{{STATE_LIBRARY}}` — or "server-side session" / "not applicable" |
| Data fetch library | `{{DATA_FETCH_LIBRARY}}` — or "server-rendered, no client fetch library" |
| Component library | `{{COMPONENT_LIBRARY}}` — or "none" |
| Form library | `{{FORM_LIBRARY}}` — or "none" |

---

## 2. Build

```bash
{{Paste the canonical build and start commands for this stack — e.g.:

# JavaScript/TypeScript
npm run dev           # dev server
npm run build         # production build
npm test              # tests

# Java (Maven)
mvn spring-boot:run   # dev server
mvn package           # production build
mvn test              # tests

# C# (.NET)
dotnet run            # dev server
dotnet publish        # production build
dotnet test           # tests

# Python
flask run / python manage.py runserver   # dev server
gunicorn app:app      # production
pytest                # tests
}}
```

**Build output:** `{{output directory or artifact — e.g. dist/, target/*.jar, bin/Release/, or "server-rendered, no build artifact"}}`
**Bundle analysis:** `{{tool if applicable / none}}`
**Code splitting:** `{{Route-based (automatic) / manual dynamic import() at: list boundaries / none}}`

**Key Decision:** `{{Are there manual lazy-load boundaries (dynamic import())? If yes, list them here so code gen uses lazy loading consistently.}}`

### Key Dependencies

<!--
List packages where code gen must know the exact version or import path — version-specific
API changes, required peer packages, non-obvious import paths, or known constraints.
Omit packages already fully described by the §1 Runtime table with no extra caveats.
-->

| Package | Version | Notes |
|---------|---------|-------|
| `{{framework-package}}` | `{{version}}` | |
| `{{router-package}}` | `{{version}}` | |
| `{{data-fetch-library}}` | `{{version}}` | `{{e.g., v5 — breaking API changes from v4; see DATA_LAYER.md}}` |
| `{{component-library}}` | `{{version}}` | `{{e.g., import path: @scope/package; requires bundler plugin X}}` |
| `{{validation-library}}` | `{{version}}` | `{{e.g., used for API response validation + form schemas}}` |

**Known version constraints / peer dep notes:**
`{{e.g., component-library requires react ≥ 18.2; do not upgrade data-fetch-library past vX without testing cache invalidation / none}}`

**Build plugins required:**
`{{e.g., unplugin-parcel-macros for component-library macros — must be in vite.config / next.config / none}}`

---

## 3. Environment Variables

<!--
Key Decision: How does this stack handle environment-specific configuration?
  JavaScript (Next.js):    NEXT_PUBLIC_* prefix → exposed to browser
  JavaScript (Vite):       VITE_* prefix → exposed to browser
  JavaScript (Angular):    environment.ts / environment.prod.ts build-time config files
  Java (Spring):           application.properties / application.yml — server-side only
  C# (.NET):               appsettings.json / appsettings.{env}.json — server-side only
  Python:                  .env loaded via python-dotenv / settings.py — server-side only
  Server-rendered apps: env vars are server-side only; no browser exposure concept applies.
-->

**Convention:** `{{ENV_VAR_CONVENTION}}` — e.g. prefix for browser-accessible vars, or "server-side only — all config is server-side" for server-rendered apps.

| Variable | Browser-accessible | Purpose | Required | Example |
|----------|-------------------|---------|----------|---------|
| `{{ENV_VAR_PREFIX}}_AUTH_CLIENT_ID` | yes | `{{Auth provider client ID (browser SDK)}}` | yes | `{{abc123}}` |
| `{{ENV_VAR_PREFIX}}_API_BASE_URL` | yes | `{{API base URL (browser-facing)}}` | yes | `{{https://api-stage.example.com}}` |
| `{{SERVER_ONLY_VAR}}` | **no** | `{{Server-side secret or config}}` | yes | `{{(secret)}}` |

**Critical:** Variables without the `{{ENV_VAR_CONVENTION}}` prefix are server-side only and are NOT available in browser JavaScript.

---

## 4. Local Development

```bash
{{Describe the setup steps:
1. cp .env.sample .env (fill in values)
2. npm install
3. npm run dev (or node server.js for HTTPS)
}}
```

**HTTPS required for local dev:** `{{yes — auth provider requires HTTPS for redirect or cookie flows, run the HTTPS dev server / no — HTTP is sufficient}}`

**Auth in development:** `{{describe any special setup — mock auth, dev token endpoint, etc.}}`

---

## 5. Routing Table (Quick Reference)

> `ROUTES.md` is authoritative. This table is a quick reference — kept in sync but not
> the source of truth.

| URL | File / Component | Auth Required | Notes |
|-----|-----------------|---------------|-------|
| `{{/}}` | `{{pages/index.tsx}}` | no | Redirects to `{{/dashboard}}` |
| `{{/resource}}` | `{{pages/resource.tsx}}` | yes | |
| `{{/resource/:id}}` | `{{pages/resourcedetails.tsx}}` | yes | `id` in path param |

---

## 6. Feature Flags

<!--
Key Decision: Are feature flags evaluated in the UI layer directly, or are they
server-authoritative (the API response tells the UI what is allowed)?

Two models:
A. UI-evaluated: a flag client (LaunchDarkly, Statsig, etc.) is initialized in the
   browser; components check flag values directly. LLM must generate flag checks.
B. Server-authoritative: the backend/API encodes availability in response fields
   (e.g., allowedActions[], enabled: bool). The UI only reads and renders these fields.
   LLM should NOT generate client-side flag checks — just read the API response field.
-->

**Feature flag evaluation:** `{{UI-evaluated via {{flag service}} / Server-authoritative — availability encoded in API response fields (e.g., allowedActions[])}}`

| Flag / Response Field | Source | Default | Controls |
|-----------------------|--------|---------|----------|
| `{{flag_name}}` | `{{UI-evaluated / API response field}}` | `{{off / on}}` | `{{what feature}}` |

`{{If server-authoritative: "LLM must NOT generate client-side flag checks. Availability is determined by {{allowedActions[]}} in API responses. Only read and display."}}` 

---

## 7. Browser Storage (Quick Reference)

> `STATE_MANAGEMENT.md §Storage Key Registry` is authoritative.

| Key | Type | Owner | Contents | TTL |
|-----|------|-------|----------|-----|
| `{{authToken}}` | sessionStorage | `{{AuthStore}}` | Auth access token | session |
| `{{app_cart}}` | localStorage | `{{CartStore}}` | Cart items + context | permanent |

---

## 8. Accessibility Compliance

| Property | Value |
|----------|-------|
| WCAG target | `{{A11Y_WCAG_LEVEL}}` (A / AA / AAA) |
| Automated tooling | `{{axe-core in Jest / Lighthouse CI / none}}` |
| Manual testing cadence | `{{each release / quarterly / ad-hoc}}` |
| Screen reader tested with | `{{VoiceOver (macOS) / NVDA (Windows) / not tested}}` |

**Known exceptions:**
| Component | Exception | Rationale |
|-----------|-----------|-----------|
| `{{component}}` | `{{what WCAG criterion it doesn't meet}}` | `{{why it's accepted}}` |

`{{None — all components meet {{A11Y_WCAG_LEVEL}} with no known exceptions.}}` ← (delete if exceptions exist)

---

## 9. Monitoring & Error Tracking

| Tool | Purpose | Notes |
|------|---------|-------|
| `{{error tracking tool}}` | Error tracking | `{{config: env var or config file name}}` |
| `{{analytics tool}}` | User behavior analytics | `{{event schema location}}` |
| `{{monitoring tool}}` | Real User Monitoring / APM | `{{NOT CAPTURED}}` |

`{{NOT CAPTURED — no monitoring configured.}}` ← (delete if monitoring exists)

---


## 10. Analytics Event Schema

<!--
Omit this section if the application has no analytics instrumentation.

If analytics is present, fill every field — an LLM generating a new feature will
skip instrumentation entirely, or name events inconsistently, without this schema.

Key Decisions to capture:
1. Which analytics library / vendor is in use?
2. What is the event naming convention?
3. What properties are required on every event?
4. Where are events fired — component, hook, or store?
5. Is there consent gating (events must not fire until the user accepts cookies)?
-->

**Analytics library:** `{{e.g., Adobe Analytics / Segment / Mixpanel / GA4 / custom util at utils/analytics.ts / none}}`
**Consent gating:** `{{e.g., events must not fire until user accepts analytics cookies — check consentStore.analyticsAllowed / no consent gate}}`

**Event naming convention:**
```
{{e.g.:
  <domain>:<action>
  Examples:
    order:view          → user viewed the order summary page
    order:submit        → user clicked "Place Order"
    order:submit_error  → order submission failed
    cart:add_item       → user added an item to cart
}}
```

**Required properties on every event:**
| Property | Type | Description |
|----------|------|-------------|
| `{{userId}}` | string | Authenticated user identifier |
| `{{sessionId}}` | string | Current session identifier |
| `{{page}}` | string | Current route path |
| `{{timestamp}}` | ISO-8601 | Event fire time (UTC) |

**Where events are fired:**
`{{e.g., always fire from hook or store action — never directly in a component render or JSX event handler. Use the analytics util: import { track } from 'utils/analytics';}}`

**Canonical fire pattern:**
```typescript
{{Paste the canonical analytics call — e.g.:

import { track } from 'utils/analytics';

// In a hook / store action (preferred):
track('order:submit', {
  orderId,
  itemCount: cart.items.length,
  totalAmount: cart.total,
});

// In a component (only for pure UI interactions with no hook):
<Button onPress={() => track('order:view_details', { orderId })}>
  View Details
</Button>
}}
```

**Rule for code generation:** `{{Every new capability MUST include at least a "view" event on page mount and an "action" event on primary user interaction. Check consentStore.analyticsAllowed before firing if consent gating is enabled.}}`

---

## 11. Links & Runbooks

| Resource | URL / Location |
|----------|---------------|
| Staging URL | `{{NOT CAPTURED}}` |
| Production URL | `{{NOT CAPTURED}}` |
| CI / CD pipeline | `{{NOT CAPTURED}}` |
| Error dashboard | `{{NOT CAPTURED}}` |
| Design system / Figma | `{{NOT CAPTURED}}` |
