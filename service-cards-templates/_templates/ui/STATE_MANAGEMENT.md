# State Management — {{SERVICE_NAME}}

> Complete shapes, actions, dependency order, and storage mappings for all global
> state owned by the UI. Works for React Context, Pinia, NgRx, Svelte stores,
> Zustand, or any other state library — the template captures the logical contract,
> not implementation syntax.
>
> **State library:** `{{STATE_LIBRARY}}`
>
> **Load during LLD / code generation for any change touching global state or browser storage.**
>
> **Last Updated:** {{DATE}}

---

## Overview

<!--
One paragraph:
- State management library in use
- How many stores/contexts exist (name them)
- The single most important thing about initialization order
- Whether any state is persisted to browser storage

Key Decision to state here: Is the initialization chain enforced reactively
(computed/derived state that auto-blocks downstream queries) or by manual guards
(components/hooks check isLoading / !!value before using)?
-->

| State library | `{{STATE_LIBRARY}}` |
|---------------|---------------------|
| Number of stores | `{{N}}` |
| Initialization chain | Reactive (auto-computed) / Manual guard pattern (pick one) |
| Persisted state | `{{sessionStorage: authToken; localStorage: cart — or "none"}}` |

---

<!--
Copy and fill the section below for each store/context in the application.
Use ## N. StoreName as the heading so MODULE_INDEX.md §5.3 can link here by anchor.
-->

## 1. {{StoreName}} ({{e.g., AuthStore / AuthContext}})

**File:** `{{contexts/AuthContext.tsx / stores/auth.ts / store/slices/auth.ts}}`
**Purpose:** {{One sentence — what problem this store solves.}}

### State Shape

<!--
TypeScript-flavored pseudocode. Even for non-TypeScript projects, express the shape
this way — it's the LLM's primary reference for what fields exist.
Mark fields as: required | optional | computed/derived
-->

```typescript
interface {{StoreName}}State {
  // Required fields
  {{fieldName}}: {{type}};          // {{brief description}}

  // Loading / error state
  isLoading: boolean;
  error: Error | null;
}
```

### Actions / Mutations

<!--
What can be called against this store. Use TypeScript function signatures even for
Vue/Svelte/Angular — mark as "action (Pinia)" or "dispatch" or "setter" as appropriate.
-->

```typescript
// Action signatures:
{{actionName}}(param: type): void | Promise<void>
{{anotherAction}}(): void
```

### Computed / Derived Values

<!--
Getters, selectors, computed properties. Values derived from state that components
can read without duplicating the derivation logic.
Omit this sub-section if there are no derived values.
-->

```typescript
// Derived values (read-only, auto-computed):
{{derivedValue}}: {{type}}   // derived from: {{which fields}}
```

### Initialization

<!--
Key Decision: What triggers this store to populate?
Where does its initial data come from?
Is initialization synchronous (from browser storage) or async (from an API call)?
-->

**Trigger:** `{{App mount / user action / reactive dependency on another store}}`
**Source:** `{{API call to /api/... / browser storage / hardcoded defaults / derived from {{OtherStore}}}}`
**Sync/async:** `{{Synchronous (reads localStorage on init) / Async (awaits API call)}}`

```typescript
{{Show the initialization code for THIS project — e.g.:

// React Context — reads sessionStorage on mount:
const [authToken, setAuthToken] = useState<string | null>(
  () => sessionStorage.getItem('authToken')
);

// Pinia — async action called on app mount:
const authStore = defineStore('auth', {
  actions: {
    async initialize() {
      this.token = sessionStorage.getItem('authToken');
      if (this.token) await this.fetchUserOrgs();
    }
  }
});

// Angular service — initialized via APP_INITIALIZER:
@Injectable({ providedIn: 'root' })
export class AuthService {
  private token$ = new BehaviorSubject<string | null>(sessionStorage.getItem('authToken'));
  // APP_INITIALIZER factory calls authService.init() before app bootstraps
  init(): Promise<void> { return this.token$.pipe(take(1)).toPromise().then(() => {}); }
}
}}
```

### Storage Persistence

<!--
If any part of this store is persisted to browser storage, document it here.
If nothing is persisted, write "None — this store is session-only (no browser storage)."

Key Decision: schema versioning strategy — if the stored shape changes, how are
old values migrated? Without a migration strategy, shape changes silently corrupt
persisted data.
-->

| Storage key | Type | Contents | TTL | Shape version | Migration strategy |
|-------------|------|----------|-----|---------------|-------------------|
| `{{key}}` | localStorage / sessionStorage / cookie | {{what is stored}} | session / permanent | `v{{N}}` | `{{none (session-only) / versioned field + migrator function / clear on version mismatch}}` |

_None — this store is session-only (no browser storage)._ ← (delete if persistence exists)

### Consumer Hooks / Selectors

<!--
How components read from this store. Show the exact import + call pattern.
-->

```typescript
{{Show the canonical consumer pattern — e.g.:

// React Context:
import { useAuth } from 'contexts/AuthContext';
const { authToken, isAuthenticated, isLoading } = useAuth();
const authToken = useAuthToken();   // shorthand selector hook

// Pinia:
import { useAuthStore } from 'stores/auth';
const authStore = useAuthStore();
const { token, isAuthenticated } = storeToRefs(authStore);

// NgRx:
import { Store } from '@ngrx/store';
import { selectAuthToken } from 'store/selectors/auth.selectors';
const authToken$ = this.store.select(selectAuthToken);
}}
```

### Rules

<!--
Invariants and forbidden patterns specific to this store.
-->

- Only `{{StoreName}}` reads/writes `{{storage key}}` — no other module touches it
- `{{isLoading}}` must be checked before using `{{fieldName}}` — it may be null during initialization
- `{{describe any write restrictions — e.g., "Only this store can set authToken"}}`

---

## 2. {{StoreName}} ({{e.g., SessionStore / SessionContext}})

<!-- Copy the full store template above and fill for each additional store -->

---

## Initialization Order & Boot Dependency Chain

<!--
CRITICAL SECTION. Document the exact sequence in which stores are populated
at application start. Every downstream store/hook that depends on a prior store
being initialized must be listed here.

Key Decision: Is this chain enforced reactively (computed/derived values that
auto-block downstream queries until prerequisites are met) or by manual guards
(components/hooks explicitly check isLoading / !!value)?

An LLM writing a new fetch unit that depends on authToken or userId must know this
to generate the correct enabled guard or reactivity chain.
-->

**Enforcement mechanism:** `{{Reactive — queries have enabled: !!dep computed automatically / Manual — components/hooks check guard expressions}}`

```
{{Dependency chain diagram. Example:

Auth provider initializes (external, browser SDK)
  └── authToken set → AuthStore.isAuthenticated = true
        └── GET /api/session fires (enabled: !!authToken)
              └── userId set → SessionStore populated
                    └── GET /api/user-profile fires (enabled: !!authToken && !!userId)
                          └── profile data available → ProfileStore populated
                                └── Dashboard page fully renders
}}
```

**Guard expressions used in hooks/queries:**

```typescript
{{Show the actual guard patterns used — e.g.:

// Each query's enabled flag mirrors the dependency:
enabled: !!authToken                                    // depends on: AuthStore
enabled: !!authToken && !!userId                        // depends on: AuthStore + SessionStore
enabled: !!authToken && !sessionLoading && !!userId     // depends on: auth complete + SessionStore
}}
```

---

## Cross-Store Dependencies

<!--
Store-to-store dependencies only. How the data fetch layer (hooks/services) depends on
stores is captured in DATA_LAYER.md §Auth Token Injection and §Standard Fetch Pattern
(the "dependency guard" / "enabled" pattern).

"Required field" = what must be non-null/non-undefined before this store can function.
"Store behavior when null" = what the store itself does while the dependency isn't ready
  (e.g. holds empty state, defers initialization, clears on change).
  Do NOT describe fetch-unit behavior here (that belongs in DATA_LAYER.md).
-->

| Store | Reads from | Required field | Store behavior when null |
|-------|------------|----------------|--------------------------|
| `{{ProfileStore}}` | `{{AuthStore, SessionStore}}` | `authToken`, `userId` | holds empty state until both are set |
| `{{CartStore}}` | `{{SessionStore}}` | `userId` | cart cleared on user change |

---

## Storage Key Registry

<!--
Every browser storage key used by this application, in one place.
This covers client-owned persisted state only (auth tokens, cart, user preferences).
In-memory server data cache settings (freshness/lifetime) are in §Cache Strategy below — not here.

"Persistence" = how long the browser keeps this value:
  session   → cleared when the tab/browser closes (sessionStorage)
  permanent → survives browser restart (localStorage)
  (not the same as in-memory server data cache lifetime in DATA_LAYER.md)

"Shape version" tracks the stored object's schema version (increment when shape changes).
"Migration strategy" documents what happens on version mismatch.

Common migration strategies:
- "none — session-only, never persists across sessions"
- "clear on mismatch — stored value is discarded, user gets fresh state"
- "versioned field + migrator — stored JSON has a 'version' field; on load, run migration fn if version < current"
-->

| Key | Storage type | Owner store | Contents | Persistence | Shape version | Migration strategy |
|-----|--------------|-------------|----------|-------------|---------------|--------------------|
| `{{authToken}}` | sessionStorage | `{{AuthStore}}` | Auth access token (string) | session | v1 | none (session-only) |
| `{{app_cart}}` | localStorage | `{{CartStore}}` | Cart items + session context | permanent | v{{N}} | `{{versioned field + migrator / clear on mismatch}}` |

**Rules:**
- Each key is owned by exactly one store — no key is written by two stores
- localStorage keys that store objects MUST include a `version` field in the serialized JSON
- On app load, each store checks `version` and runs the migrator if needed before using the stored value

---

## Cache Strategy

<!--
In-memory server data cache settings per domain. This is where the LLM finds freshness
and lifetime decisions — not in DATA_LAYER.md, which covers fetch contracts only.

Freshness = how long before data is re-fetched on next use.
Lifetime  = how long unused entries stay in memory before eviction.
Not all libraries have both concepts — leave N/A if not applicable.

Maps to library concepts:
  TanStack Query: staleTime (Freshness) / gcTime (Lifetime)
  SWR:            dedupingInterval (Freshness) / n/a (Lifetime)
  RTK Query:      refetchOnMountOrArgChange (Freshness) / keepUnusedDataFor (Lifetime)
  Angular HttpClient + RxJS: no built-in cache — document custom strategy or "N/A"

This section covers server-fetched data only.
Client-owned persisted state (auth tokens, cart) → §Storage Key Registry above.
-->

| Domain | Freshness | Lifetime | Invalidation Trigger | Notes |
|--------|-----------|----------|----------------------|-------|
| `{{resource list}}` | `{{always re-fetch}}` | `{{5min}}` | Mutation success | |
| `{{resource details}}` | `{{5min}}` | `{{30min}}` | Mutation success (specific item) | |
| `{{static / reference data}}` | `{{24h}}` | `{{7d}}` | Manual (never auto) | `{{e.g. config, lookup tables}}` |
