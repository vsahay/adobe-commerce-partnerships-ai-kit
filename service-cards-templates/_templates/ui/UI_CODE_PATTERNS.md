# Code Patterns — {{SERVICE_NAME}}

> Coding conventions for the UI layer. Part A is general TypeScript/JS + component
> design standards (stack-agnostic). Part B is project-specific and must be filled
> in before using these cards for code generation. Code review rejects Part B deviations.
>
> **When the general (Part A) and project-specific (Part B) rules disagree,
> Part B wins** — but the disagreement should be rare and deliberate.
>
> **Last Updated:** {{DATE}}

---

# Part A — General UI Standards

> **STANDARDS_VERSION:** 2026-04-21
>
> General TypeScript + UI component idioms that code review would expect in any
> modern UI application. Shared across teams; vendored verbatim from the UI template set.
>
> **Refresh procedure:** update `STANDARDS_VERSION` when adopting a newer version of
> this Part A from the shared `service-card-skills/_templates/ui/` template.

## A.1 Naming

- **Components / Pages:** `PascalCase`. No `Page` suffix required (use local convention).
- **Hooks / Composables / Services:** naming is stack-dependent — `use<Name>` (React/Vue), `<Name>Service` (Angular), `fetch<Name>` / `create<Name>Store` (Svelte/other). Exact convention declared in Part B.
- **Event handlers:** `handle<Action>` — `handleSubmit`, `handleAddToCart`, `handleClose`.
- **Store / Context selectors:** naming is stack-dependent — `use<StoreName>` / `use<Value>` (React), `storeToRefs(use<Name>Store())` (Pinia), `select<Value>` (NgRx), `$<storeName>` (Svelte). Exact convention declared in Part B.
- **Constants:** `UPPER_SNAKE_CASE` for module-level constants. Enum members: `PascalCase`.
- **Types / Interfaces:** `PascalCase`. No `I` prefix (not `IUser` — just `User`).
- **Booleans:** prefix `is`, `has`, `can`, `should` — `isLoading`, `hasError`, `canSubmit`.
- **Test files:** `{{ComponentName}}.test.tsx` or `{{hookName}}.test.ts` co-located with source, OR in a parallel `__tests__/` directory — pick one convention and be consistent.

## A.2 Component Design Principles

- **Functional components only** — no class components.
- **Single responsibility.** If the component name needs "And" or "Panel", consider splitting.
- **No business logic in components.** Orchestration and side effects belong in hooks/composables.
- **Props interface defined above the component** with a descriptive name: `interface ProductCardProps { ... }`.
- **No prop drilling past 2 levels.** If a value is needed 3+ levels deep, promote it to the store.
- **Default export for pages** (router requirement). Named exports for shared components.
- **Target ≤ 200 lines per component file.** Extract sub-components when exceeding.

## A.3 Async & Side Effects

- **All data fetching through the data fetch library** — no raw `fetch()` or `axios` calls in components.
- **Mutations through the library's mutation primitive** — no `useEffect` + `fetch` for writes.
- **`useEffect` (or equivalent) only for genuinely imperative side effects** — scroll position, focus management, third-party SDK initialization.
- **Async event handlers:** wrap with `async` and `try/catch`; never let unhandled rejections escape.

## A.4 TypeScript Discipline

- **No `any`.** Use `unknown` + narrowing, or the data library's inferred response types.
- **`as` casts require a comment** explaining why narrowing isn't possible.
- **`satisfies` over `as`** for literal type assertions.
- **Explicit return types on exported functions and hooks.** Infer internally, be explicit at boundaries.
- **Zod (or equivalent)** for validating external data (API responses, localStorage) at runtime.

## A.5 State Hygiene

- **Local state stays local** — only promote to store when ≥2 unrelated components need it.
- **Derived values as computed/selectors** — never duplicate a piece of state; derive it.
- **No side effects in reducers, setters, or Pinia actions** that modify state synchronously — side effects go in `async` actions or `useEffect`.
- **Reads from store are always through the designated selector/hook** — never read raw store object properties in components.

## A.6 Accessibility (a11y)

- **WCAG target:** `{{A11Y_WCAG_LEVEL}}` (AA recommended minimum).
- **All interactive elements reachable by keyboard** — `Tab`, `Enter`, `Space`, arrow keys as appropriate.
- **Focus management on modal open/close** — focus must move to the first interactive element inside a modal; return to the trigger on close.
- **`aria-label` or visible label** on every interactive element without visible text.
- **Never use color alone** to convey meaning — always pair with an icon or text label.
- **`role` attribute** on custom interactive elements that have no native HTML equivalent.
- **Don't remove focus outlines** — style them; don't hide them.

## A.7 Performance

- **Lazy-load routes not on the critical path** — use dynamic `import()` or the router's lazy-load syntax.
- **Memoize expensive computations only when profiled** — don't memo reflexively.
- **Stable function references in render** — avoid creating new function objects on every render in event-heavy lists.
- **Optimize images** — use the framework's image optimization component (`<Image>` in Next.js, etc.) rather than raw `<img>`.

## A.8 Error Handling

- **Never swallow silently.** Every `catch` must either log or surface an error to the user.
- **Errors from mutations remain visible until dismissed** — don't auto-clear mutation errors on the next unrelated action.
- **Meaningful error messages** — "Something went wrong" is not acceptable; show what failed and (if possible) what the user can do.

## A.9 Testing Principles

- **Arrange / Act / Assert** — visual blank line between sections.
- **Test behavior, not implementation.** Test what the user sees and does, not internal component methods.
- **Mock at the boundary** — mock the fetch library or network layer, not internal hook functions.
- **One behavior per test** — multiple assertions on the same behavior are fine; multiple behaviors in one test are not.
- **No random data without a seed** — deterministic tests only.

## A.10 CSS / Styling

- **The project's styling mechanism is declared in Part B.** Part A only sets: no inline styles for anything that isn't truly dynamic, no `!important`, `z-index` values declared as named constants (no magic numbers).
- **Consistent unit** — pick `rem` or `px` for spacing and stick to it (Part B declares the choice).

---

# Part B — Project-Specific (Strict) Conventions

> Every placeholder below MUST be filled. Every row in Part B is enforced in code review.
> Deviations require a written exception in the CHANGELOG.

## B.1 Naming Conventions

| Kind | Pattern | Example |
|------|---------|---------|
| Page | `{{pages/<route>.tsx}}` | `{{pages/resources.tsx}}` |
| Component (reusable) | `{{components/<domain>/<Name>.tsx}}` | `{{components/orders/OrderList.tsx}}` |
| Hook / Composable | `{{hooks/use<Resource><Action>.ts}}` | `{{hooks/usePaginatedResourceList.ts}}` |
| Store / Context | `{{contexts/<Name>Context.tsx}}` | `{{contexts/CartContext.tsx}}` |
| Store selector hook | `{{use<Value>()}}` | `{{useAuthToken(), useCurrentUserId()}}` |
| Util | `{{utils/<name>Utils.ts}}` | `{{utils/cartUtils.ts}}` |
| Form schema | `{{schemas/<Name>Schema.ts or co-located}}` | `{{schemas/CreateResourceSchema.ts}}` |
| Test file | `{{<Name>.test.tsx co-located / tests/<path>}}` | `{{ResourceCard.test.tsx}}` |

## B.2 File / Directory Structure

```
{{Paste the actual project directory tree here, annotated.
This is the authoritative reference for "where does X go?"
Example:

project-root/
├── pages/                    ← Next.js Pages Router — one file per route
│   ├── _app.tsx              ← Root layout + auth gate + context providers
│   ├── _document.tsx
│   ├── index.tsx             ← Redirects to /dashboard
│   └── dashboard.tsx
├── components/
│   ├── common/               ← Shared across multiple domains
│   ├── <domain>/             ← Domain-specific components
│   └── Layout/
├── contexts/                 ← Global state stores / contexts
├── hooks/                    ← Data-fetching hooks + mutations ({{DATA_FETCH_LIBRARY}})
├── models/                   ← Zod schemas + inferred TypeScript types
├── utils/                    ← Pure helpers
├── config/                   ← Static config (org type mapping, etc.)
└── types/                    ← Shared TypeScript types not tied to a schema
}}
```

## B.3 `{{DATA_FETCH_LIBRARY}}` Patterns

<!--
The canonical fetch and mutation patterns for THIS project, with the project's
actual auth injection, query key convention, staleTime defaults, and error handling.
Code gen copies these snippets verbatim — do not leave as generic examples.
-->

**Auth injection:** `{{AUTH_INJECTION_PATTERN}}`
**Query key convention:** `{{['noun', ...identifiers]}}`

```typescript
{{Paste the project's canonical useQuery pattern here}}
```

```typescript
{{Paste the project's canonical useMutation pattern here}}
```

## B.4 `{{STATE_LIBRARY}}` Patterns

<!--
The canonical pattern to read from and write to the global store in THIS project.
One pattern per store if they differ meaningfully; otherwise one generic example.
-->

```typescript
{{Paste the canonical store read pattern — e.g.:

// Reading from context (React):
import { useAuth } from 'contexts/AuthContext';
const { authToken, isAuthenticated } = useAuth();

// Reading from Pinia:
import { useAuthStore } from 'stores/auth';
const { token } = storeToRefs(useAuthStore());
}}
```

```typescript
{{Paste the canonical store write/action pattern}}
```

## B.5 Design System — `{{COMPONENT_LIBRARY}}`

<!--
Key Decision: Is using plain HTML ever acceptable?
If the answer is "no, always use the component library", say so explicitly.
If there are justified exceptions (accessible custom widgets, etc.), name them.
-->

**Adoption completeness:** `{{Always use {{COMPONENT_LIBRARY}} — plain HTML interactive elements are NOT acceptable / Exceptions: {{list cases where plain HTML is used and why}}}}`

**Import pattern:**
```typescript
{{e.g.:
import { Button, TextField, Table } from '@react-spectrum/s2';
import { VBtn, VTextField, VDataTable } from 'vuetify/components';
}}
```

**Scenario → Component mapping:**

| UI Scenario | Component to use | Notes |
|-------------|-----------------|-------|
| Primary action button | `{{Button variant="cta"}}` | |
| Secondary / ghost button | `{{Button variant="secondary"}}` | |
| Text input | `{{TextField}}` | |
| Dropdown / select | `{{Picker / Select}}` | |
| Data table | `{{TableView / DataTable}}` | |
| Modal / dialog | `{{DialogTrigger + Dialog}}` | |
| Toast / notification | `{{Toast / Snackbar}}` | |
| Loading state | `{{ProgressCircle / Skeleton}}` | |
| Form (container) | `{{Form}}` or plain `<form>` if library has no Form | |
| Checkbox | `{{Checkbox}}` | |
| Date picker | `{{DatePicker}}` | |

**Theming tokens:**
```typescript
{{Show how design tokens are consumed — e.g.:
// CSS custom properties (most libraries):
color: var(--spectrum-blue-800);
// Or theme object:
theme.palette.primary.main
}}
```

**a11y provided by the library:**
- `{{e.g., React Spectrum S2: ARIA roles, keyboard nav, focus management all built-in for listed components}}`
- `{{What the library does NOT cover that the project must add manually}}`

## B.6 Form Patterns

<!--
Key Decisions:
1. Which form library (or none)?
2. Where does validation schema live?
3. When does validation run (on blur / on change / on submit)?
4. How are errors displayed (inline / summary at top)?
5. Submission UX (disable button while pending / loading indicator)?
-->

**Form library:** `{{FORM_LIBRARY}}`
**Validation schema location:** `{{Co-located with form component / centralized in utils/schemas/ / co-located in models/}}`
**Validation runs on:** `{{blur + submit / change / submit only}}`
**Error display:** `{{Inline below each field / summary at top / toast}}`

```typescript
{{Paste the canonical form pattern for this project — e.g.:

// React Hook Form + Zod:
import { useForm } from 'react-hook-form';
import { zodResolver } from '@hookform/resolvers/zod';
import { CreateResourceSchema, CreateResourceRequest } from 'models/Resource';

const form = useForm<CreateResourceRequest>({
  resolver: zodResolver(CreateResourceSchema),
});

const handleSubmit = form.handleSubmit(async (data) => {
  await mutateAsync(data);
});
}}
```

**Submission UX pattern:**
```typescript
{{Show the disable-on-pending + error-display pattern:

<Button
  type="submit"
  isDisabled={form.formState.isSubmitting || isPending}
>
  {isPending ? 'Saving...' : 'Save'}
</Button>
{mutationError && <div role="alert">{(mutationError as Error).message}</div>}
}}
```

## B.7 Error Handling in Components

```typescript
{{Canonical query error display:}}
```

```typescript
{{Canonical mutation error display:}}
```

```typescript
{{Error boundary usage (if used):
// Wrap page or section:
<ErrorBoundary fallback={<ErrorDisplay />}>
  <SomePage />
</ErrorBoundary>
}}
```

## B.8 Test Patterns

**Framework stack:**

| Component | Library |
|-----------|---------|
| Test runner | `{{TEST_RUNNER}}` (Jest / Vitest) |
| Component testing | `{{COMPONENT_TEST_LIBRARY}}` |
| HTTP mocking | `{{MSW / jest.fn() on global.fetch / Axios mock adapter}}` |
| E2E | `{{E2E_FRAMEWORK}}` (Playwright / Cypress) |

**Unit test — Hook:**
```typescript
{{Paste the canonical hook unit test skeleton for this project}}
```

**Unit test — Component:**
```typescript
{{Paste the canonical component unit test skeleton}}
```

**E2E test — Page flow:**
```typescript
{{Paste the canonical Playwright/Cypress page test skeleton}}
```

**Test naming convention:**
```
{{e.g.:
<ComponentOrFetchUnit>_<scenario>_<expectedBehavior>
Examples:
  ProductCard_addToCart_updatesCartStore
  ProductPricingService_fetchError_returnsErrorState
  checkout_submitOrder_navigatesToConfirmation
}}
```

## B.9 Internationalization (i18n)

<!--
Omit this section entirely if the application has no i18n requirement (single language only).

If i18n is present, fill every field — an LLM generating a new component that
doesn't know about i18n will hardcode strings, breaking translations silently.

Key Decisions to capture:
1. Which i18n library is in use?
2. Where do translation files live and how are they named?
3. What is the key naming convention?
4. How does a component access translations (hook, HOC, context)?
5. Are pluralization and date/number formatting handled — and how?
-->

**i18n library:** `{{e.g., react-intl / i18next + react-i18next / vue-i18n / Angular i18n / none}}`
**Translation files:** `{{e.g., public/locales/{locale}/common.json / src/i18n/{locale}.ts}}`
**Supported locales:** `{{e.g., en-US (default), fr-FR, ja-JP}}`
**Locale source:** `{{e.g., user profile preference / browser Accept-Language header / URL path prefix /lang query param}}`

**Key naming convention:**
```
{{e.g.:
  <domain>.<component>.<element>
  Examples:
    order.summary.title         → "Order Summary"
    order.summary.submitButton  → "Place Order"
    common.errors.networkError  → "Something went wrong. Please try again."
}}
```

**How a component accesses translations:**
```typescript
{{Paste the canonical translation call pattern — e.g.:

// react-intl:
import { useIntl } from 'react-intl';
const intl = useIntl();
intl.formatMessage({ id: 'order.summary.title' })
// JSX shorthand:
<FormattedMessage id="order.summary.title" />

// i18next + react-i18next:
import { useTranslation } from 'react-i18next';
const { t } = useTranslation();
t('order.summary.title')

// vue-i18n:
const { t } = useI18n();
t('order.summary.title')

// Angular i18n (build-time extraction):
<span i18n="order summary title">Order Summary</span>
}}
```

**Pluralization pattern:**
```typescript
{{Show how pluralized strings are handled — e.g.:

// react-intl (ICU syntax in translation file):
// "cart.itemCount": "{count, plural, one {# item} other {# items}}"
intl.formatMessage({ id: 'cart.itemCount' }, { count: 3 })  // → "3 items"

// i18next:
t('cart.itemCount', { count: 3 })
// en.json: { "cart.itemCount_one": "{{count}} item", "cart.itemCount_other": "{{count}} items" }

// OR: "not used — all pluralization is handled server-side"
}}
```

**Date / number / currency formatting:**
```typescript
{{Show the canonical pattern — e.g.:

// react-intl built-in:
intl.formatNumber(1234.5, { style: 'currency', currency: 'USD' })  // → "$1,234.50"
intl.formatDate(new Date(), { dateStyle: 'medium' })

// OR: "not used — dates and numbers are pre-formatted by the API"
}}
```

**Rule for code generation:** `{{Never hardcode user-visible strings — always use the translation call pattern above. Add new keys to all locale files before generating the component.}}`
