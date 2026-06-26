# Generating Code

**Frequency:** Once per feature (re-runnable if needed)  
**Prerequisites:**
- Backend and Frontend Service cards are complete and reviewed
- Backend LLD is approved
- UI LLD is approved

---

## What This Step Does

This step uses the AI to generate working implementation code for both the backend and frontend, following the approved LLDs and the patterns documented in your service cards.

Unlike the LLD steps, this step writes real code directly into your project repos. The AI generates all files listed in both LLDs, then runs validation automatically.

You do not need to direct the AI through the generation — it works through the LLD systematically. Your job at this step is to:
1. Run the command
2. Monitor for validation failures
3. Complete a standard code review of the output

---

## Before You Run

Confirm the following are true:

- [ ] Backend service cards are accurate and up to date
- [ ] UI service cards are accurate and up to date
- [ ] Backend LLD is in its final approved state
- [ ] UI LLD is in its final approved state
- [ ] Your backend and frontend repos are on a clean branch (no uncommitted changes)

Create a feature branch in both repos before running. The generated code should land on a branch, not directly on main.

---

## Running the Skill

Open Claude Code in the `partner-ai-kit` directory and run:

```
/implement-feature <featureName> <path-to-backend-repo> <path-to-frontend-repo>
```

**Example:**

```
/implement-feature anytimeUpgrade ../my-partner-backend ../my-partner-frontend
```

The skill runs in 7 phases:

| Phase | What happens |
|---|---|
| 1 — Load reference material | Reads both LLDs and both sets of service cards |
| 2 — Generate backend code | Creates models, controllers, API routes, constants |
| 3 — Generate UI code | Creates fetch hooks, page components, sub-components, CSS modules, context updates |
| 4 — Wire up | Cross-checks contracts between backend and UI — URLs, request types, response shapes |
| 5 — Validate | Runs TypeScript diagnostics and lint in both repos |
| 6 — Smoke test | Generates and runs a temporary Playwright test for each new page |
| 7 — Update service cards | Appends new capabilities to the relevant service card sections |

This will take several minutes. Do not interrupt it between phases.

---

## What Gets Generated

### Backend files (in your backend repo):

| Type | Location (follows your service card paths) |
|---|---|
| Data models | `src/models/<feature>/` |
| Controllers | `src/controllers/<feature>/` |
| Route definitions | `src/routes/<feature>/` |
| Constants | `src/constants/<feature>/` |

### Frontend files (in your frontend repo):

| Type | Location (follows your service card paths) |
|---|---|
| Fetch hooks | `src/hooks/<feature>/` |
| Page components | `src/pages/<feature>/` |
| Sub-components | `src/components/<feature>/` |
| CSS modules | Co-located with components |
| Context updates | In the relevant context file (modified, not replaced) |

Exact paths will match the directory structure documented in your service cards. If the service cards are accurate, the generated files will land in the right places.

---

## Handling Validation Failures

The AI runs TypeScript, lint, and Playwright smoke tests automatically after generating code. If any of these fail, the AI will attempt to fix the issues before reporting back to you. Most validation issues are caught and resolved automatically.

If the AI cannot resolve a validation failure, it will stop and report the error with context. Here is how to handle the most common cases:

**TypeScript type error in a generated file:**

The most common cause is a mismatch between the response type inferred from the API spec and the actual structure the LLD uses. Read the error, find the field in question, and check whether the API spec and LLD are consistent. If they are inconsistent, fix the LLD and re-run. If they are consistent, the TypeScript error may be a genuine issue with the generated code — fix it manually.

**Lint error:**

Usually a style issue that the AI missed (import order, unused variable). Fix manually; these do not require re-running the skill.

**Playwright smoke test failure:**

The smoke test navigates to each new page and checks that it renders without a crash. A failure means a component threw an unhandled error on render. Read the Playwright output, find the component, and check whether the error is in the generated code or in an existing component the generated code depends on.

**If validation errors are pervasive (more than 3-4 files affected):**

Stop, do not try to fix them individually. The root cause is almost always inaccurate service cards or an inconsistency between the backend LLD and UI LLD. Identify the inaccuracy, fix it in the source (service card or LLD), and re-run.

---

## Re-Running the Skill

`/implement-feature` is safe to re-run. It overwrites generated files with fresh output. If you have made manual edits to generated files that you want to keep, save them before re-running.

Re-run when:
- Validation failures suggest a systemic issue traced back to the LLD or service cards
- You approved an LLD with an error and caught it after code generation
- The PM requests a change to the UI that requires a revised UI LLD

Do not re-run to fix minor issues like lint errors or small logic corrections — fix those manually.

---

## Code Review

After the skill completes successfully, create a pull request on each repo and conduct a standard code review. The LLD approval already covered design; the code review covers correctness.

**What to look for in code review:**

- Generated models and types accurately reflect the API spec shapes
- Error handling is complete — all error codes from the API spec are handled
- There are no hardcoded values that should come from configuration
- New routes have correct auth guards
- New components are accessible (semantic HTML, ARIA where needed)
- No console logs, debug comments, or TODO comments in production code
- Service card updates at the end of Phase 7 are accurate — the new capabilities are correctly described

**What code review is not for:**

- Changing the architecture. That was the LLD review.
- Adding features that were not in the LLD. That is a new feature workflow.
- Redesigning components. If the design is wrong, update the experience card and UI LLD and regenerate.

---

## After Code Review

Once both PRs are merged:

1. Verify the service cards were updated correctly in Phase 7. If the updates are missing or inaccurate, update them manually and commit.
2. Run your full test suite in both repos.
3. Deploy to your staging environment and perform manual QA against the experience card step by step.

The feature workflow for this feature is complete. The next feature starts at FW-1.