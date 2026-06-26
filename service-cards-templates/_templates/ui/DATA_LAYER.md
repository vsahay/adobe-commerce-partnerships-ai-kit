# Data Layer — {{SERVICE_NAME}}

> The **READ layer** (data fetching, caching, loading states) and **WRITE layer**
> (mutations, cache invalidation) for the UI. Documents how data moves between
> the server and the component layer — without reading source code.
>
> **Data layer library:** `{{DATA_FETCH_LIBRARY}}`
> **Auth injection pattern:** `{{AUTH_INJECTION_PATTERN}}`
>
> **Load during LLD / code generation — not during Impact Analysis.**
>
> **Last Updated:** {{DATE}}

---

## Overview

<!--
One paragraph:
- What data layer library is in use (TanStack Query / SWR / Apollo / Pinia + useFetch / Angular HttpClient + RxJS / SvelteKit load fns)
- Whether SSR pre-fetching is used (affects where read operations run)
- Global caching defaults (staleTime, gcTime — or "no global defaults")
- How auth tokens are injected (per-unit parameter / Axios interceptor / HTTP interceptor / store-aware wrapper)
- What the error surface is (thrown errors caught by error boundary / returned error values / toast)

Key Decisions to state explicitly here:
1. Central auth interceptor vs per-unit token passing (determines every read unit's signature)
2. Global staleTime vs per-unit staleTime
3. Error surface: thrown vs returned, global boundary vs component-level
4. Optimistic update posture: used or not; rollback mechanism
-->

| Data layer library | `{{DATA_FETCH_LIBRARY}}` |
|--------------------|--------------------------|
| Auth injection | `{{AUTH_INJECTION_PATTERN}}` |
| Data freshness policy | `{{e.g., 'always re-fetch on access / 5 min global default / per-read-unit only'; maps to: TanStack staleTime / SWR dedupingInterval / RTK Query keepUnusedDataFor / n/a for Angular HttpClient}}` |
| Cache lifetime policy | `{{e.g., 'evict immediately / 5 min after last use / 7 days'; maps to: TanStack gcTime / SWR / RTK Query / n/a}}` |
| Error surface | `{{thrown — caught by error boundary / returned as error field / toast on write failure}}` |
| Optimistic updates | `{{used — rollback via X / not used}}` |
| SSR pre-fetching | `{{yes — data fetched on server before hydration / no — all client-side}}` |

---

## Read Layer — Registry

<!--
One row per logical data domain the UI reads. This is the quick-reference for code gen.
For full cache key and error handling details, see the Standard Read Pattern section.

"Auth Injection" column: "param" = token passed as parameter to the read unit;
"interceptor" = Axios/fetch interceptor reads token from store;
"store" = the read unit reads the store directly inside its body.

Freshness/lifetime (staleTime, gcTime, dedupingInterval) per domain → STATE_MANAGEMENT.md §Cache Strategy.
This table covers read contracts only: what is fetched, how the key is formed, how auth is injected.
-->

| Domain | Read Unit | Endpoint | Cache Key Pattern | Auth Injection |
|--------|-----------|----------|--------------------|----------------|
| {{Domain1}} | `{{fetchResource}}` | `GET {{/api/resource}}` | `['resource', scopeId]` | `{{param / interceptor / store}}` |
| {{Domain2}} | `{{fetchResourceDetails}}` | `GET {{/api/resource/:id}}` | `['resource', id]` | `{{param}}` |

---

## Standard Read Pattern

<!--
The canonical read unit pattern for THIS project.
Fill in the actual pattern used — do not leave as a generic example.
This is what code gen will copy for every new read operation.

Key Decision documented here: how the auth token reaches the read unit.
The three common patterns:
  A. Token as parameter:               readResource(id, authToken)
  B. Token from store inside read unit: const authToken = getAuthToken(); // inside the unit body
  C. Central interceptor:              token injected automatically; read units never see it
-->

**Auth injection:** `{{AUTH_INJECTION_PATTERN}}`

```typescript
{{
Paste the canonical read unit pattern for THIS project.
The three structural decisions that vary by stack:

  1. AUTH INJECTION
     A. Token as parameter:            useResource(id, authToken)        // React/Vue
     B. Token from store inside unit:  const token = useAuth();          // inside hook/composable/service
     C. Central interceptor:           token injected automatically; read unit never sees it

  2. DEPENDENCY GUARD
     Run the read only when required inputs are available.
     Concept: "enabled" / "immediate" / "conditional subscribe" depending on your library.
     Examples:
       TanStack Query:   enabled: !!authToken && !!resourceId
       SWR:              const { data } = useSWR(authToken && resourceId ? key : null, fetcher)
       Pinia + useFetch: watchEffect(() => { if (authToken.value && resourceId.value) fetch() })
       Angular resolver: canActivate guard populates token before component mounts

  3. RETURN SHAPE
     What does the read unit return to the consumer?
       TanStack / SWR:  { data, isLoading, error }
       Angular service: Observable<Resource>
       Composable:      { data: Ref<Resource>, pending: Ref<boolean>, error: Ref<Error | null> }

Replace this comment block with the actual pattern used in this project.
}}
```

**Cache key convention:** `{{describe how cache keys are constructed — e.g. ['noun', ...identifiers, scopeId] / URL string / object with operation name}}`

**Dependency guard pattern:** `{{describe when the read should run — e.g. only when authToken and required params are set; leave unset/null to block the read}}`

---

## Standard Write Pattern

<!--
The canonical write unit pattern for THIS project.
Fill in the actual pattern — this is what code gen copies for every new write operation.

Key Decisions documented here:
1. Optimistic update posture: does this project use them?
2. Cache invalidation: what triggers a re-read after a write succeeds?
3. Error handling: is the write error shown inline, as a toast, or thrown?
-->

**Optimistic update posture:** `{{used — rollback via onError / not used in this project}}`

```typescript
{{
Paste the canonical write unit pattern for THIS project.
The key structural decisions to capture:

  1. PAYLOAD → ASYNC CALL
     How the write operation is initiated and what it sends.
     Examples:
       TanStack useMutation:  mutationFn: async (payload) => fetch('/api/resource', { method: 'POST', body: JSON.stringify(payload) })
       Pinia action:          async createResource(payload) { await api.post('/resource', payload) }
       Angular service:       createResource(payload: Dto): Observable<Resource> { return this.http.post('/api/resource', payload) }

  2. CACHE UPDATE AFTER SUCCESS
     How the local cache reflects the change after the server confirms.
     Options:
       A. Invalidate + re-read:  queryClient.invalidateQueries({ queryKey: ['resources'] })
       B. Optimistic update:     set cache before call, rollback on error
       C. Navigate away:         no invalidation — list re-reads naturally on next visit
       D. Manual cache write:    update cache entry directly with the returned value

  3. ERROR SURFACE
     How the consumer learns about failure.
       Thrown error caught by caller / error field on write state / toast dispatched inside the write unit / returned Result type

Replace this comment block with the actual pattern used in this project.
}}
```

**Cache invalidation convention:** `{{invalidate the parent list cache key on write success}}`

---

## Auth Token Injection

<!--
Key Decision: how does the auth token reach outbound calls?
This is the single most impactful choice for code generation — it determines the
signature of every read unit the LLM generates.

Document the canonical pattern so the LLM never has to infer it.
-->

**Pattern:** `{{AUTH_INJECTION_PATTERN}}`

```typescript
{{Show the exact pattern used in this project. Three options:

// Option A — Token as parameter (easy to test; read unit receives token explicitly):
function useResource(id: string, authToken: string) { ... }
// Consumer reads token from store and passes it in:
// const { authToken } = useAuth();

// Option B — Read unit reads from store internally (less parameter passing, harder to test):
function useResource(id: string) {
  const { authToken } = useAuth();  // reads store inside the read unit
  ...
}

// Option C — Central interceptor (Axios / Angular HttpClient):
// All outbound calls go through an interceptor that reads the token from the store.
// Read units do NOT accept or handle the token — it is injected automatically.
axiosInstance.interceptors.request.use(config => {
  config.headers.Authorization = `Bearer ${store.getState().auth.token}`;
  return config;
});
function useResource(id: string) {
  return useQuery({ queryFn: () => api.get(`/resource/${id}`) }); // no token param needed
}
}}
```

**Rule for code gen:** `{{Always pass authToken as parameter to read units / Never pass token — use the interceptor / etc.}}`

---

## Error Handling Posture

<!--
How errors from read and write operations are surfaced to the UI.
Key Decision: this must be one consistent pattern — LLM must not mix approaches.
-->

**Read errors:** `{{thrown and caught by error boundary / returned as error field — component renders inline error}}`

**Write errors:** `{{shown as inline error below the form / shown as toast notification / thrown and caught by boundary}}`

**Retry behavior:** `{{library default / no retry / retry on 5xx only / custom retry logic with backoff}}`

```typescript
{{Show the canonical error display patterns used in this project:

// Read error — inline display:
const { data, error, isLoading } = useResource(id, authToken);
if (error) return <div role="alert">{(error as Error).message}</div>;

// Write error — inline display:
const { mutateAsync, error: writeError } = useCreateResource();
return (
  <>
    {writeError && <div role="alert">{(writeError as Error).message}</div>}
    <Button onPress={() => mutateAsync(payload)}>Submit</Button>
  </>
);
}}
```

---

## Real-time / WebSocket Pattern

<!--
Omit this section entirely if the application has no real-time data requirements
(no WebSockets, no Server-Sent Events, no long-polling).

If real-time is present, fill every field — an LLM generating a feature that
touches live data will not know how to wire subscriptions, handle reconnections,
or integrate with the state layer without this section.

Key Decisions to capture:
1. What transport is used — WebSocket, SSE, long-polling?
2. Where is the connection managed — a dedicated store, a read unit, a singleton service?
3. How does the real-time layer interact with the server-data cache (TanStack/SWR/etc.)?
4. What is the reconnection / error-recovery strategy?
5. What triggers subscribe / unsubscribe?
-->

**Transport:** `{{WebSocket / Server-Sent Events (SSE) / long-polling / none}}`
**Connection owner:** `{{e.g., singleton WebSocketService / dedicated store / read unit that manages its own connection}}`
**Auth injection:** `{{e.g., token sent as query param on connect / Authorization header on upgrade / cookie-based}}`

**Subscribe / unsubscribe pattern:**
```typescript
{{Paste the canonical subscribe pattern — e.g.:

// Read unit (hook / service / composable) that owns a WebSocket connection:
function useOrderUpdates(orderId: string) {
  const { authToken } = useAuth();
  useEffect(() => {
    const ws = new WebSocket(`wss://api.example.com/orders/${orderId}?token=${authToken}`);
    ws.onmessage = (event) => {
      const update = JSON.parse(event.data);
      queryClient.setQueryData(['orders', orderId], update);  // push into cache
    };
    return () => ws.close();  // unsubscribe on unmount
  }, [orderId, authToken]);
}

// SSE:
useEffect(() => {
  const source = new EventSource(`/api/orders/${orderId}/stream`);
  source.onmessage = (event) => { ... };
  return () => source.close();
}, [orderId]);
}}
```

**Cache integration:** `{{e.g., real-time updates are pushed directly into the data layer cache — no separate re-read triggered / real-time updates are stored in a dedicated store slice separate from the read cache}}`

**Reconnection strategy:** `{{e.g., exponential backoff via reconnecting-websocket library / browser SSE auto-reconnects natively / manual retry with 3s delay / none — connection loss shows an error banner}}`

**Error / disconnect handling:**
```typescript
{{Show how connection errors surface to the user — e.g.:

ws.onerror = () => setConnectionStatus('error');
ws.onclose = () => setConnectionStatus('disconnected');
// Component renders a "Reconnecting..." banner when status !== 'connected'
}}
```

**Rule for code generation:** `{{Real-time subscriptions must be managed in X — never open a WebSocket or SSE connection directly in a component. Always clean up (close / unsubscribe) in the effect's return function.}}`

---

## Adding a New Read Unit — Checklist

When adding a new read unit, follow these steps in order:

1. **Identify the capability** in `UI_MODULE_INDEX.md → ## Capability Catalogue` — the read unit should already be listed there
2. **Name the read unit** following the project's naming convention — stack-dependent (e.g. `use<Resource>` for React/Vue, `<Resource>Service` for Angular, `fetch<Resource>` for others)
3. **Define the cache key** following the convention: `{{['noun', ...identifiers]}}`
4. **Set the dependency guard**: `{{condition under which the read should run — only when required params are available; leave unset/null to block the read}}`
5. **Set freshness and lifetime** in `STATE_MANAGEMENT.md §Cache Strategy` for this domain
6. **Validate the response shape** with Zod (or the project's validation library) before returning
7. **Add the read unit** to `DATA_LAYER.md → ## Read Layer — Registry` table
8. **Add the read unit** to `UI_MODULE_INDEX.md → ## Cross-Reference Indexes` Hook / Composable / Service → Capabilities table

---
