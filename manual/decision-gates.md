# Decision Gate Guide

This guide defines what to review at each approval checkpoint in the toolkit workflow, and explains why each review matters. Every gate must be explicitly completed before the next step begins.

Decision gates exist because the toolkit is a pipeline: the output of each step becomes an input to the next. An error that passes through a gate compounds downstream — it becomes harder and more expensive to fix at every subsequent step.

---

## DG-1 — Backend Service Card Review

**Triggered by:** Completion of Backend Service Card  
**Time investment:** 30–60 minutes for a new project

### What to check

**`SERVICE_CARD.md`**
- Service name, owner, and tier are correct
- The "what this service owns" section accurately describes the service's responsibility boundary — not what it will own someday, what it owns now
- The "fragile areas" section calls out real known risks, not generic ones

**`MODULE_INDEX.md`**
- Every capability entry maps to a real file at the stated path — verify a sample of paths actually exist
- No capabilities are missing (the AI may miss modules in non-standard directories)
- No capabilities are listed that no longer exist due to past refactors

**`CONTRACTS.md`**
- Every inbound HTTP endpoint is listed — compare against your route definitions
- Request/response shapes are accurate for the endpoints your features will use

**`CONNECTORS.md`** — highest priority
- Every outbound API call is listed with the correct base URL
- Auth mechanism is accurately described: how is the token obtained, how is it passed (header name, prefix), how is it refreshed?
- Timeout values match your actual configuration
- Retry behavior is accurate — retries on which status codes, how many times?
- Error posture is accurate — does your connector throw, return null, or return a typed error?

**`CODE_PATTERNS.md`** — highest priority
- Naming conventions match what you actually use (not what you aspire to use)
- Directory layout matches the real structure — verify 3-4 actual files to confirm paths
- DTO pattern is accurate — how do you define request and response types?
- Error handling pattern is accurate — how do you surface errors to callers?
- Validation pattern is accurate — which library, and at which layer?

**`DB_SCHEMA.md`**
- Table names are correct
- Column names and types are correct for tables the feature will touch
- Relationships are accurate

### Why this gate matters

Every piece of generated code will follow the patterns described in these cards. A wrong connector auth pattern means every generated API call will pass auth the wrong way. A wrong naming convention means every generated file will be named inconsistently with the rest of your project. These are not cosmetic issues — they will fail TypeScript checks, break authentication, and require manual correction across every generated file.

Catching an inaccuracy here costs 5 minutes. Catching it after code generation costs 30-60 minutes of manual file editing, plus the risk of missing an occurrence.

### What to do with findings

Edit the card files directly. Do not re-run the generator for small corrections. Commit the corrected cards before proceeding.

---

## DG-2 — UI Service Card Review

**Triggered by:** Completion of UI Service card  
**Time investment:** 30–60 minutes for a new project

### What to check

**`ROUTES.md`** — highest priority
- Every existing route is listed at the correct path
- Auth guards are accurate — which routes require authentication, and which store/context manages auth state?
- The stores required per route are listed correctly
- Navigation patterns are accurate — does your app use programmatic navigation, link components, or both?

**`DATA_LAYER.md`** — highest priority
- Fetch hook structure matches what you actually use — is it a custom hook, React Query, SWR, or something else?
- Auth injection is accurately described — where does the token come from, and how is it added to the request? This must match what `CONNECTORS.md` says on the backend side
- Error surface is accurate — what does a hook return when a request fails? Does it throw, return `{ data, error }`, or something else?
- Mutation pattern is accurate — how do you structure write operations vs. read operations?

**`STATE_MANAGEMENT.md`**
- All global stores are listed — verify against your actual store files
- Store initialization order is accurate — does the auth store initialize before the user profile store?
- Browser storage usage is accurate — does anything persist to localStorage or sessionStorage?

**`UI_CODE_PATTERNS.md`**
- Part A (general standards): naming conventions are accurate — PascalCase for components, camelCase for hooks, etc.
- Part B (project-specific): directory structure matches the real layout — verify a few actual file paths
- Design system map is accurate — which component library is used, and for which component types?

**`UI_PLATFORM.md`**
- Framework and version are correct
- Build scripts are accurate
- Environment variable names are correct

### Why this gate matters

An inaccurate `DATA_LAYER.md` is the single most common source of generated code bugs. If auth injection is described incorrectly, every generated fetch hook will send API requests without proper authentication. This will not be caught by TypeScript — it will be caught in runtime testing, after all the code has been written.

An inaccurate `ROUTES.md` means generated routes will have wrong auth guards or missing store dependencies. These are not always caught by TypeScript either — they manifest as runtime errors or security gaps.

### What to do with findings

Edit the card files directly and commit before proceeding.

---

## DG-3 — Backend LLD Review

**Triggered by:** Completion of Backend LLD
**Time investment:** 30–45 minutes

### What to check

**Data flow diagram** — most important section
- Read every step in the data flow. Would you build the backend this way if writing from scratch?
- Are all steps present? Is there a missing auth validation step, a missing transformation, a missing error branch?
- Does the data flow match the API spec? Are the right fields being passed to the right endpoints?
- Are error conditions from the API spec handled at the right step, in the right way?

**Backend changes table**
- Are the file paths correct for your project?
- Is anything missing — a route registration, a middleware hookup, a constant definition?
- Is anything unnecessary — a new utility that already exists, a new model for a type that is already defined?
- For each modified file, is the described change correct?

**Reuse summary** — verify every entry
- Does the referenced function, utility, or module actually exist at the stated path?
- Does it do what the LLD claims it does? (Check the implementation, not just the name)
- If the LLD says to reuse X and X is slightly different from what is needed, note that and decide: adapt X, or create a new function?

**Design decisions**
- Read each decision and form your own opinion before reading the AI's reasoning
- Do you agree with the choice? If not, change it in the LLD and update any downstream sections that reference it

**Acceptance criteria coverage**
- Map each criterion back to the experience card
- Verify that the described backend changes actually satisfy each criterion — not just "yes this is covered" but "this specific change in this specific file satisfies this criterion"

**Out of scope**
- Confirm that nothing in the experience card was accidentally excluded

### Why this gate matters

The LLD is the design. Code generation is execution. If you approve a wrong design, you get a correctly-executed wrong design — and changing the design after code generation means regenerating code.

Engineers who treat LLD review as a formality ("looks reasonable, ship it") consistently report higher rework rates and more validation failures. Engineers who treat it as a design review — reading every line, challenging every assumption, verifying every reuse claim — report generated code that passes review on the first pass.

The time you spend here is not overhead. It is time shifted forward from code review.

### What to do with findings

Edit the LLD file directly for small corrections (wrong path, wrong field name, minor design adjustment).

Re-run `/apply-api-spec` only if the LLD has structural problems (wrong data flow, wrong architecture). Before re-running, fix the source input that caused the problem — the API spec or the service cards.

---

## DG-4 — UI LLD Review

**Triggered by:** Completion of UI LLD   
**Time investment:** 30–45 minutes

### Review

**Fetch hook specs** — most critical section
- Every URL must match the backend LLD exactly — same path, same method
- Every request body field must match the backend LLD exactly — same field names, same types
- Every response field the hook uses must be in the backend LLD response shape — same field names
- Error handling in each hook must cover all error codes from the API spec

**Route changes**
- New route paths are correct
- Auth guards are applied correctly — compare against `ROUTES.md` patterns
- Required stores are listed correctly
- Route params match what the component actually needs

**Component specs**
- Each component has a clear, single responsibility
- Props are typed accurately — no `any`, no over-broad types
- If Figma was used, verify the visual specs reflect the design — component structure, not pixel values

**Reuse summary**
- Verify each referenced component and hook actually exists
- Verify they work the way the LLD describes

**State changes**
- New state is introduced at the right level — local component state vs. context vs. global store
- Changes to existing stores are minimal and do not affect other features

**Step-by-step journey match**
- Open the Adobe-provided experience card and the UI LLD side by side
- For each step in the experience card: does the LLD describe UI behavior that matches?
- Pay specific attention to: what the user sees on success, what happens on error, what the transition looks like (modal vs. page navigation vs. inline update)

**Error states**
- Every error state named in the experience card must appear as a named UI state in the LLD
- "Handle errors" is not a named UI state — the LLD should specify what the user sees, whether they can retry, and whether existing data is preserved
- If the experience card says "show inline error; keep previous preview visible," that exact behavior should be in the component spec

**Figma alignment**
- If Figma was used, verify the component specs in the LLD match the design — component structure and behavior, not pixel values
- If Figma was not used, flag any areas where the LLD describes a component vaguely that the experience card describes specifically

**Out of scope**
- Read the "Out of Scope" list. Does it exclude anything the experience card included? If so, raise it — do not silently accept scope reduction. Either add it back or get explicit confirmation from your Adobe partner contact that it can be deferred.

### Why this gate matters

It exists because the UI LLD is where Adobe's product intent (the experience card) is translated into an engineering plan — and that translation can fail silently. The LLD can look technically correct to an engineer while not matching what Adobe's experience card describes.

The most common failure mode: the experience card says "show a confirmation modal"; the AI interpreted it as "navigate to a confirmation page" because your project has existing page-navigation patterns. Both are valid technically. Only one matches the experience card. This discrepancy is invisible in the LLD if the PM does not review it.

Discovering this mismatch after code generation means writing new components and deleting generated ones. Discovering it here means a 2-line edit to the LLD.