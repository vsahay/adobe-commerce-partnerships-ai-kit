# Routes — {{SERVICE_NAME}}

> The routing contract for this UI application. Every navigable URL, its parameters,
> the state/context preconditions required to render it correctly, and the navigation
> patterns between routes.
>
> **Router:** `{{ROUTER}}`
> **Auth guard mechanism:** `{{AUTH_GUARD_MECHANISM}}`
>
> **Load during LLD / code generation for routing and navigation work.**
>
> **Last Updated:** {{DATE}}

---

## Overview

<!--
One paragraph:
- How many distinct routes exist
- What router is in use
- Whether auth is enforced globally (one place) or per-route (each route declares its own guard)
- What the root redirect is (e.g. "/" → "/dashboard")
- Any nested routing / layout hierarchy worth noting

Key Decisions to state explicitly here:
1. Auth enforced globally or per-route? (Code gen must know whether to add a guard to every new page)
2. What renders while auth initializes? (Loading screen, empty, redirects?)
3. Is the URL the source of truth for page state, or does state live in the store?
-->

| Router | `{{ROUTER}}` |
|--------|-------------|
| Auth guard | `{{AUTH_GUARD_MECHANISM}}` |
| Auth guard location | `{{file path of the guard, e.g. pages/_app.tsx or router/guards.ts}}` |
| Root redirect | `{{/}}` → `{{/first-page}}` |
| Unauthenticated redirect | `{{/login}}` or `{{LoginScreen component}}` |
| Layout hierarchy | `{{single root layout / nested layouts — describe}}` |

---

## Route Registry

<!--
One row per navigable URL. This is the authoritative routing contract.

Column definitions:
- Route Path: the URL pattern as declared in the router (e.g. /resources/:id, /resources/[id], /resources/{id})
- File / Component: the page file or component that renders this route
- Path Params: required dynamic segments (e.g. :resourceId)
- Query Params: accepted query string params — mark required vs optional
- Auth Required: yes / no / role-based
- Context Preconditions: which store/context values must be non-null before the page renders correctly
- Stability: STABLE / IN FLUX / DEPRECATED

Mark query params as: (required) or (optional, default: X)
-->

| Route Path | File / Component | Path Params | Query Params | Auth Required | Context Preconditions | Stability |
|------------|-----------------|-------------|-------------|---------------|----------------------|-----------|
| `{{/}}` | `{{pages/index.tsx}}` | — | — | no | — | STABLE |
| `{{/resource}}` | `{{pages/resource.tsx}}` | — | `offset` (optional, default: 0), `limit` (optional, default: 20) | yes | `{{userId}} set` | STABLE |
| `{{/resource/:id}}` | `{{pages/resourcedetails.tsx}}` | `id` | — | yes | `{{userId}} set` | STABLE |

---

## Auth Guard Pattern

<!--
Describe exactly how authentication is enforced. Be specific — LLM-generated pages
must know whether to add their own guard or rely on a global one.

Key Decisions:
- Is auth enforced globally (one wrapper component/layout/middleware) or per-route?
- What is the redirect target for unauthenticated users?
- What renders during auth initialization (before the check completes)?
- Is there a role/permission layer beyond basic auth?
-->

**Enforcement mechanism:** `{{AUTH_GUARD_MECHANISM}}`

```
{{Show the actual guard code snippet — e.g.:

// React / Next.js — global _app.tsx wrapper:
if (!isAuthenticated && !isLoading) return <LoginScreen />;
if (isLoading) return <LoadingSkeleton />;

// Vue Router — beforeEach guard:
router.beforeEach((to, from, next) => {
  if (to.meta.requiresAuth && !authStore.isAuthenticated) {
    next({ name: 'login' });
  } else {
    next();
  }
});

// Angular — CanActivate:
canActivate(): boolean {
  if (!this.authService.isAuthenticated) {
    this.router.navigate(['/login']);
    return false;
  }
  return true;
}
}}
```

**Rules for new pages:**
- `{{Global guard — new pages do NOT need their own auth check; the guard covers all routes.}}`
- `{{OR: Per-route guard — every new page that requires auth MUST declare meta.requiresAuth = true (Vue) / use AuthGuard (Angular) / etc.}}`

---

## Navigation Patterns

<!--
How the app navigates between routes programmatically. LLM-generated code that
navigates between pages must follow the project's established pattern.

Key Decisions:
1. Programmatic navigation — what function/method is used?
2. State passing — URL params vs router state vs store (this choice affects deep-linking)
3. Back navigation — is there a custom back button pattern?
4. Deep-link requirements — which query params must survive navigation?
-->

### Programmatic Navigation

```
{{Show the canonical navigation call for this project — e.g.:

// React / Next.js:
import { useRouter } from 'next/router';
const router = useRouter();
router.push(`/resources/${id}`);                                    // prefer URL params for bookmarkable state
router.push({ pathname: '/resources/[id]', query: { id } });        // typed form

// Vue:
import { useRouter } from 'vue-router';
const router = useRouter();
router.push({ name: 'resource-details', params: { id } });

// Angular:
constructor(private router: Router) {}
this.router.navigate(['/resources', id]);
}}
```

### State Passing Strategy

<!--
CRITICAL Key Decision: How is data passed between routes?
This affects deep-linking support, back-button behavior, and refresh behavior.
Pick ONE canonical answer for this project.
-->

| Approach | Used for | Example |
|----------|----------|---------|
| URL path params | {{e.g., resource identity (IDs)}} | `/resources/:resourceId` |
| URL query params | {{e.g., pagination, filters}} | `?offset=0&limit=20` |
| Router state | {{e.g., transient data that shouldn't be in URL}} | `router.push('/confirmation', { state: { orderId } })` |
| Store | {{e.g., data already fetched that is needed on destination}} | Read from store on destination page |

**Rule:** `{{State that needs to survive a page refresh MUST be in the URL. State that is only relevant for a single navigation step MAY use router state. Data already in the store MUST be read from the store, not passed via navigation.}}`

### Back Navigation

```
{{Show back navigation pattern if non-standard, or "Uses browser default back button — no custom pattern."}}
```

---

## Route Constants

<!--
Key Decision: Are route paths defined as constants (single source of truth) or
as inline strings scattered through the codebase?

If constants exist, document them here so code gen uses the constant, not a
literal string.
-->

**Pattern:** `{{Constants file at utils/routes.ts / inline strings / typed route object}}`

```
{{Show the constants or typed route pattern — e.g.:

// utils/routes.ts
export const ROUTES = {
  HOME: '/',
  RESOURCES: '/resources',
  RESOURCE_DETAILS: (resourceId: string) => `/resources/${resourceId}`,
  SETTINGS: '/settings',
} as const;

// Usage:
router.push(ROUTES.RESOURCE_DETAILS(id));
}}
```

---

## Error Routes

| Scenario | Route / Component | Behavior |
|----------|------------------|----------|
| Not found (404) | `{{pages/404.tsx}}` | {{Next.js auto-serves / custom error component}} |
| Auth failure / unauthenticated | `{{LoginScreen component}}` | {{Rendered inline / redirect to /login}} |
| Access denied (authorized but insufficient permissions) | `{{components/AccessDenied.tsx}}` | {{Rendered inline / redirect}} |
| Unhandled render error | `{{components/ErrorBoundary.tsx}}` | {{Catches and shows fallback UI / redirects}} |

---

## Notes for Code Generation

<!--
Summarize the rules that affect every new page or route a code gen step might produce.
-->

- `{{New pages do / do not need their own auth guard}}`
- `{{Route path constants live at X — always reference the constant, not a literal string}}`
- `{{Query params for resource identity are always named X — do not invent new param names}}`
- `{{Router state is / is not used for passing data between pages}}`
