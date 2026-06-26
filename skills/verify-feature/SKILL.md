---
name: verify-feature
description: >
  Audits a generated feature implementation against its LLD — checking every
  file in the Change Summary table, every bullet in every Change Required cell,
  wiring between frontend and backend, and experience card acceptance criteria.
  Read-only. Reports pass / partial / fail per file with specific gaps and
  fix guidance. Run after implement-feature or after manual edits.
---

# verify-feature

Audit a feature implementation against its LLD. Read every generated file,
check it against the spec, and produce a gap report the partner can act on.

---

## Input

```
<featureName> <targetRepoPath> [--mode=both|backend|ui] [--use-reference-files]
```

| Argument | Required | Description |
|----------|----------|-------------|
| `featureName` | Yes | Kebab-case feature name (e.g. `flexible-discounts`). Locates LLDs and experience card. |
| `targetRepoPath` | Yes | Absolute path to the target repo. All generated files are read from here. |
| `--mode` | No | `--mode=both` (default) — audit backend + UI + wiring. `--mode=backend` — backend files only. `--mode=ui` — UI files only. |
| `--use-reference-files` | No | Load LLDs from the kit repo's pre-approved reference location instead of the target repo. |

**Derived paths** (identical to `implement-feature`):

| Resource | Default path | Path when `--use-reference-files` |
|----------|-------------|-----------------------------------|
| Backend LLD | `<targetRepoPath>/docs/ai-kit/LLD/backend/<featureName>-lld.md` | `feature-specs/<featureName>/reference-files/LLD/backend/<featureName>-lld.md` |
| UI LLD | `<targetRepoPath>/docs/ai-kit/LLD/ui/<featureName>-lld.md` | `feature-specs/<featureName>/reference-files/LLD/ui/<featureName>-lld.md` |

If a LLD filename doesn't exactly match `<featureName>-lld.md`, search the directory for `*<featureName>*-lld.md` and use the match.

If a required LLD does not exist, stop and tell the user — there is nothing to verify against.

---

## Phase 1 — Load reference material

Load the same files `implement-feature` would load for the given `--mode`:

| File | Load when |
|------|-----------|
| Experience card (`feature-specs/<featureName>/experience-card/*.md`) | always |
| Backend LLD | always (UI hooks must match backend contracts) |
| UI LLD | `--mode=both` or `--mode=ui` |

From each LLD extract:
- The **complete file manifest** from §3 Change Summary — every row, every bullet in the Change Required column
- The **§2 Data Flow** — used for wiring verification

From the experience card extract:
- Every user-facing screen and surface
- All acceptance criteria
- Every interaction, loading state, error state, and empty state

---

## Phase 2 — Compilation check

Derive the type-check command from the project's own configuration — do not assume a specific stack:

1. Read `UI_PLATFORM.md §Build Scripts` (if `--mode=both` or `--mode=ui`) and `BUILD_CONFIG.md §Build Scripts` (if `--mode=both` or `--mode=backend`). Use the script labelled `type-check`, `typecheck`, `ts:check`, or equivalent.
2. If neither card names one, read `<targetRepoPath>/package.json` `scripts` and use the first matching key: `type-check`, `typecheck`, `ts:check`.
3. If nothing is found, skip this phase and note it in the report.

Run the derived command in `<targetRepoPath>`. Capture all errors. A clean exit (code 0) is a pass. Do not attempt to fix errors in this skill — record them and continue to Phase 3.

---

## Phase 3 — Per-file spec audit

For each file row in the LLD Change Summary §3, perform the steps below.
Process files for the applicable `--mode` only (backend rows for `--mode=backend`,
UI rows for `--mode=ui`, all rows for `--mode=both`).

**Read the file efficiently:** read the full file for files under ~150 lines;
for larger files use targeted `grep` for identifiers named in the Change Required
bullets before reading surrounding context. Never skip a file.

### 3a — Existence check

| Action | Expected outcome | Finding if wrong |
|--------|-----------------|-----------------|
| CREATE | File exists at the path in the File column | `MISSING` — file was never created |
| MODIFY | File exists (it was pre-existing) | `MISSING` — file was deleted or path is wrong |

If a file is `MISSING`, record every Change Required bullet for that file as unmet
and move to the next file.

### 3b — Bullet-by-bullet audit

For each bullet in the Change Required column of the file's §3 row:

1. **Identify the verifiable claim** — what concrete artifact does the bullet require?
   Examples: a named export, a specific prop, a URL string, a CSS class, a constant key,
   a conditional render, an `enabled` guard expression, a specific HTTP method.

2. **Locate it in the file** — grep for the identifier or string; read the surrounding
   lines for context.

3. **Judge coverage:**
   - `PASS` — the artifact is present and matches the spec intent
   - `PARTIAL` — the artifact is present but deviates from the spec (wrong value,
     missing a sub-condition, wrong type, incomplete field list)
   - `MISSING` — the artifact is absent entirely

4. **For every non-PASS finding**, record:
   - The exact bullet text from the LLD
   - What was found in the file (or "not found")
   - A one-sentence fix description

**Audit posture:** judge semantic intent, not exact syntax. A prop named `marketSeg`
that serves the same purpose as spec-required `marketSegment` is `PARTIAL`, not
`MISSING`. Do not flag style differences, comments, or extra fields not in the
spec — only gaps and deviations.

---

## Phase 4 — Wiring audit

**Skip if `--mode=backend` or `--mode=ui`.**

Re-run the same checks as `implement-feature` Phase 4a against the actual generated files:

- [ ] Every hook's fetch URL matches the route handler path registered in `pages/api/`
- [ ] Every `type` dispatch value in a hook matches a case in the corresponding route handler
- [ ] Every request field sent by a hook is accepted by the controller
- [ ] Every response field accessed in a component is defined in the backend schema
- [ ] Auth / session params (access token, org ID, etc.) are forwarded on every call that requires them
- [ ] Navigation targets (page paths passed as hrefs or `router.push` args) exist as page files

For each item: `PASS` or `FAIL` with the specific mismatch.

---

## Phase 5 — Experience card coverage

For each acceptance criterion and user-facing surface in the experience card:

- [ ] A route or component covers the screen / surface
- [ ] Every described user interaction has a wired event handler
- [ ] Loading state is rendered during in-flight fetches
- [ ] Error state is rendered on fetch or mutation failure
- [ ] Empty state is rendered when data is absent

Mark each `COVERED` or `NOT COVERED` with the specific missing element.

---

## Phase 6 — Interactive browser check

**Skip if `--mode=backend`.**

This phase asks the user to verify the feature in a running browser session using a structured checklist derived from the experience card. It is the only phase that confirms runtime behaviour — static analysis cannot substitute for it.

### 6a — Ensure the dev server is running

1. Read the dev server start command from `UI_PLATFORM.md §Build Scripts` (or `package.json` `scripts.dev` / `scripts.start` if the card does not capture it).
2. Derive the expected local port (default: 3000; use the port in the start command if specified).
3. Try to reach `http://localhost:<port>` — if it responds, the server is already running.
4. If it is not running: start it using the derived command and wait until the port is responsive (up to 30 seconds). If it does not become responsive, tell the user to start it manually and ask them to reply when it is ready before continuing.

### 6b — Generate the acceptance criteria checklist

From the experience card's acceptance criteria and user-facing surfaces, produce a numbered checklist. Each item must be a concrete, browser-testable action — not a code claim. Format:

```
BROWSER CHECKLIST — <featureName>
Server: http://localhost:<port>

Please go through each item in the browser and reply with the number of any that fail (or "all pass").

 1. [ ] <specific action + expected outcome>
        Where: <page path or nav steps to reach it>

 2. [ ] <specific action + expected outcome>
        Where: <page path or nav steps to reach it>

 ...
```

Rules for writing checklist items:
- Derive each item directly from an acceptance criterion or described user interaction in the experience card — do not invent items
- Make the action unambiguous: "Click 'Add Promo Code' on a product in the checkout summary" not "test promo code"
- State the expected outcome: "— a discount discovery panel should appear inside the dialog"
- Include the exact URL path or navigation steps to reach the surface under test
- Cover every acceptance criterion, loading state, error state, and empty state described in the experience card

### 6c — Collect user responses

Present the checklist and wait for the user's reply. Accept:
- `"all pass"` — all items pass, record all as PASS
- A list of numbers (e.g. `"2, 5 fail"`) — record those as FAIL, the rest as PASS
- Per-item notes — record verbatim as the finding for that item

Record results to feed into Phase 7.

---

## Phase 7 — Report

Emit the report in this exact format. Never omit a section; write "none" when a
section has no findings.

```
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
verify-feature: <featureName>  [mode: both | backend | ui]
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

COMPILATION
  <Clean (0 errors) | N error(s)>
  <list each tsc error on its own line, or "none">

━━━ BACKEND FILES ━━━   (omit section if --mode=ui)

<path/to/file.ts>  [CREATE | MODIFY]  <PASS | PARTIAL | FAIL | MISSING>
  <bullet text from LLD>
    → <PASS | PARTIAL: <what was found> | MISSING>
    → Fix: <one sentence>   (omit for PASS)
  ...

<repeat for each backend file>

━━━ UI FILES ━━━        (omit section if --mode=backend)

<path/to/file.tsx>  [CREATE | MODIFY]  <PASS | PARTIAL | FAIL | MISSING>
  <bullet text from LLD>
    → <PASS | PARTIAL: <what was found> | MISSING>
    → Fix: <one sentence>   (omit for PASS)
  ...

<repeat for each UI file>

━━━ WIRING ━━━          (omit section if --mode=backend or --mode=ui)

  Hook endpoint URLs      <PASS | N mismatch(es)>
  Type dispatch values    <PASS | N mismatch(es)>
  Request shapes          <PASS | N mismatch(es)>
  Response field access   <PASS | N mismatch(es)>
  Auth param forwarding   <PASS | N mismatch(es)>
  Navigation targets      <PASS | N mismatch(es)>
  <detail each failure>

━━━ EXPERIENCE CARD ━━━

  <criterion text>  <COVERED | NOT COVERED: missing element>
  ...

━━━ BROWSER CHECK ━━━   (omit section if --mode=backend)

  <N>/<total> checklist items confirmed by user
   1. <item text>  <Pass | Fail: <user-reported finding>>
   2. ...

━━━ SUMMARY ━━━

  Files audited:    <N>
  Files passing:    <N>   (all bullets PASS)
  Files partial:    <N>   (≥1 bullet PARTIAL, none FAIL or MISSING)
  Files failing:    <N>   (≥1 bullet FAIL or file MISSING)
  Compilation:      <Clean | N error(s)>
  Wiring:           <Pass | N issue(s)>   (omit if mode is backend or ui)
  Experience card:  <All covered | N criterion/criteria not covered>
  Browser check:    <All pass | N item(s) failed | Skipped>   (omit if mode is backend)

  Overall: <PASS — implementation matches LLD | PARTIAL — see gaps above | FAIL — critical gaps found>
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
```

**Overall verdict rules:**
- `PASS` — compilation clean, all files PASS, wiring PASS, all experience card criteria covered, all browser checklist items pass
- `PARTIAL` — no missing files, no compilation errors, but ≥1 bullet is PARTIAL or ≥1 experience card criterion uncovered or ≥1 browser item failed
- `FAIL` — any missing file, any compilation error, any FAIL bullet, or any wiring failure

After the report, if overall is `PARTIAL` or `FAIL`, add a **Next steps** block:

```
NEXT STEPS
  1. <highest-priority fix — usually a missing file or compilation error>
  2. <next fix>
  ...
```

Order by severity: missing files → compilation errors → missing bullets → partial bullets → wiring gaps → uncovered criteria.
