# FR-2 — UI Service Card Generation

**Who runs this:** Engineer  
**Frequency:** Once per project onboarding, then again after significant frontend restructuring  
**Prerequisite:** Backend service cards exist and have been reviewed

---

## What This Step Does

This step scans your frontend codebase and produces a set of **UI service card files** — structured documents that describe how your frontend is coded. These cards are the frontend equivalent of the backend service cards.

The cards capture things like:
- Your route structure, including which routes require auth and which stores they need
- How fetch hooks are structured, including how auth tokens are injected
- Which state management stores exist, what they own, and their initialization order
- Your component naming conventions, directory structure, and design system usage
- Your runtime stack, build scripts, and environment variables

Every piece of frontend code the toolkit generates will follow the patterns in these cards. The consistency between the generated code and your existing frontend depends entirely on the accuracy of these cards.

---

## Before You Run

- Backend Service Card generation must be complete. The UI service cards reference patterns that may connect to backend contracts — having both sets complete allows the LLD steps to cross-check them.
- You must have access to the frontend repository path on your machine.
- The frontend repo should be in a stable state.

---

## Running the Skill

Open Claude Code in the `partner-ai-kit` directory and run:

```
/generate-ui-service-card <path-to-frontend-repo> --feature-name <featureName>
```

**Example:**

```
/generate-ui-service-card ../my-partner-frontend --feature-name anytimeUpgrade
```

The skill scans your frontend repo in 10 steps:
1. Reads manifest file to identify the framework, build tooling, and key dependencies
2. Scans the route registry to find all routes, auth guards, and navigation patterns
3. Reads fetch units to understand how API calls are structured and how auth is injected
4. Reads state management files to find stores, actions, and initialization order
5. Reads component files to identify naming patterns, directory structure, and design system usage
6. Reads CSS/styling files to identify styling approach (modules, utility classes, tokens)
7. Identifies capability boundaries — what the frontend can already do
8. Reads environment variable configuration
9. Reads build scripts and deployment config
10. Assembles all findings into the 7 card files

---

## What Gets Produced

The skill produces **7 card files** at:

```
.claude/feature-specs/<featureName>/reference-files/BridgeServiceCards/ui/
```

| File | What it captures |
|---|---|
| `UI_SERVICE_CARD.md` | Identity: app name, framework, what the UI owns, data contracts summary |
| `UI_MODULE_INDEX.md` | Capability catalogue: what the frontend can do, mapped to implementation files, with key design decisions |
| `ROUTES.md` | Route registry: path, component, params, auth guard, stores required per route |
| `DATA_LAYER.md` | Fetch hooks: structure, auth injection method, error surface, mutation pattern |
| `STATE_MANAGEMENT.md` | Stores, actions, initialization order, browser storage usage |
| `UI_CODE_PATTERNS.md` | Naming conventions, directory structure, design system map, test skeletons |
| `UI_PLATFORM.md` | Runtime stack, build scripts, env vars, key dependencies |

---

## Reviewing the Cards — Decision Gate DG-2

**Do not proceed to any feature workflow step until you have reviewed the UI cards.**

Open the [Decision Gate Guide — DG-2](../decision-gates.md#dg-2-ui-service-card-review) and complete the review checklist. Pay particular attention to:

- `DATA_LAYER.md` — if auth injection or error handling is wrong, all generated fetch hooks will have the same bug
- `ROUTES.md` — if auth guards or store dependencies are wrong, generated routes will not integrate correctly
- `UI_CODE_PATTERNS.md` — if naming conventions or directory structure are wrong, generated files will be placed in the wrong locations

---

## When to Re-Run vs. When to Edit

| Situation | Action |
|---|---|
| A new route was added | Edit `ROUTES.md` — add the route entry |
| Auth token injection method changed | Edit `DATA_LAYER.md` |
| A new global store was added | Edit `STATE_MANAGEMENT.md` |
| Naming conventions changed project-wide | Edit `UI_CODE_PATTERNS.md` |
| A new major UI framework or build tool was adopted | Re-run `/generate-ui-service-card` |
| The routing library changed | Re-run `/generate-ui-service-card` |

When re-running, the generator overwrites existing card files. Back up manual edits first, then reapply them.

---

## After This Step

Your project is now fully onboarded. Both backend and UI service cards exist, have been reviewed, and are committed to the repo.

Proceed to **[Reviewing the Experience Card](../feature-workflow/review-experience-card.md)** to begin the first feature workflow.