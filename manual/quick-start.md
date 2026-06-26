# Quick Start

This guide walks you through any VIP MP feature using AI-assisted code generation. Start with a quick visual overview of the user journey, try the implementation on Adobe's reference app, or go straight to the full workflow on your own project. For a deeper understanding of how the toolkit works and why it is structured the way it is, read the [Manual](README.md) before you start.

---

## Optional — Visualize the experience before implementing

Want a quick visual overview of the user journey? Pass the experience card to any AI agent and ask it to generate a visual reference:

> Take `feature-specs/flexible-discounts/experience-card/flexible-discounts.md` and tell your agent:
> *"Can you generate a visual representation of the user experience described in this file?"*

Most AI agents (Claude, ChatGPT, Gemini, GitHub Copilot) can generate a visual representation of the user journey — no code changes required.

---

## Try it out on commerce-partnerships-ref-app

A hands-on way to explore what the toolkit can do. Uses Adobe's reference LLD and service cards to generate a feature implementation on **commerce-partnerships-ref-app** — Adobe's reference VIP MP partner app — in a single skill command.

**What you need:**
- [commerce-partnerships-ref-app](https://github.com/OneAdobe/commerce-partnerships-ref-app) cloned on the `develop` branch
- this repo cloned alongside it

**Directory layout:**
```
~/Documents/
  partner-ai-kit/                    ← this repo
  commerce-partnerships-ref-app/     ← develop branch
```

### Step 1 — Clone and install commerce-partnerships-ref-app

```
git clone -b develop https://github.com/OneAdobe/commerce-partnerships-ref-app.git ../commerce-partnerships-ref-app
cd ../commerce-partnerships-ref-app && npm install && cp .env.sample .env
```

Fill in your VIP MP credentials in `.env`. See [configuration](https://github.com/OneAdobe/commerce-partnerships-ref-app#configuration) for details.

### Step 2 — Open your AI coding agent in the kit directory

```
cd ../partner-ai-kit
```

Open your AI coding agent here. Skills are invoked as slash commands (e.g. `/implement-feature`).

> **Using Claude Code?** See [Setting Up Claude Code](setup/claude-code.md) to wire the kit before running skills.

### Step 3 — Run the feature skill

```
/implement-feature flexible-discounts ../commerce-partnerships-ref-app --use-reference-files
```

The `--use-reference-files` flag tells the skill to load the pre-approved LLD and service cards from `feature-specs/<featureName>/reference-files/` in this repo instead of generating them.

### Step 4 — Verify the feature

```
/verify-feature flexible-discounts ../commerce-partnerships-ref-app --use-reference-files
```

Audits every generated file against the LLD spec, checks wiring, and runs the project's type-check. When static checks pass, the skill starts the dev server automatically at [http://localhost:9000](http://localhost:9000) and presents a browser checklist — go through each item and reply with any that fail.

---

## Run the full workflow

To integrate a VIP MP feature into your codebase, follow these steps to generate service cards, produce LLDs from Adobe's feature specs, and implement the integration code.

**Directory layout:**
```
~/Documents/
  partner-ai-kit/      ← this repo
  your-project/        ← your VIP MP partner app
```

### Step 1 — Open your AI coding agent in the kit directory

All skills must be run from the `partner-ai-kit/` working directory. Open your AI coding agent here.

> **Using Claude Code?** See [Setting Up Claude Code](setup/claude-code.md).

### Step 2 — Generate backend service card

> Run once per project. Re-run after a major refactor.

```
/generate-backend-service-card <targetRepoPath>
```

Example:
```
/generate-backend-service-card ../your-project
```

Scans your backend codebase and produces a service card set capturing its patterns, contracts, and conventions. Written to `<targetRepo>/docs/ai-kit/service-cards/backend/`.

### Step 3 — Generate UI service card

> Run once per project. Re-run after a major refactor.

```
/generate-ui-service-card <targetRepoPath>
```

Example:
```
/generate-ui-service-card ../your-project
```

Scans your frontend codebase and produces a service card set covering component patterns, state management, data layer, and routing conventions. Written to `<targetRepo>/docs/ai-kit/service-cards/ui/`.

### Step 4 — Apply API spec → backend LLD

```
/apply-api-spec <targetRepoPath> <featureSpecPath>
```

Example:
```
/apply-api-spec ../your-project feature-specs/flexible-discounts/APISpec/flexible-discounts-apispec.md
```

Reads the API spec and your backend service cards, then produces a Low-Level Design for all backend changes needed. Written to `<targetRepo>/docs/ai-kit/LLD/backend/`.

**Review the LLD before continuing** — this is the cheapest point to catch design issues.

### Step 5 — Apply experience card → UI LLD

```
/apply-experience-card <featureSpecPath> <targetRepoPath> <backendLLDPath>
```

| Argument | Required | Description |
|----------|----------|-------------|
| `featureSpecPath` | Yes | Path to the experience card file |
| `targetRepoPath` | Yes | Absolute path to the target UI repo |
| `backendLLDPath` | No | Path to the backend LLD. Pass `none` if the feature makes no new backend calls. |

Example:
```
/apply-experience-card \
  feature-specs/flexible-discounts/experience-card/flexible-discounts.md \
  ../your-project \
  ../your-project/docs/ai-kit/LLD/backend/flexible-discounts-lld.md
```

Reads the experience card and the approved backend LLD, reads your UI service cards, and produces a UI Low-Level Design covering components, data fetch units, state, and navigation. Written to `<targetRepo>/docs/ai-kit/LLD/ui/`.

**Review the LLD before continuing.**

### Step 6 — Implement the feature

```
/implement-feature <featureName> <targetRepoPath> [--mode=both|backend|ui] [--use-reference-files]
```

| Argument | Required | Description |
|----------|----------|-------------|
| `featureName` | Yes | Kebab-case feature name (e.g. `flexible-discounts`) |
| `targetRepoPath` | Yes | Absolute path to the target repo |
| `--mode` | No | `--mode=both` (default) — backend + UI. `--mode=backend` — backend only. `--mode=ui` — UI only. |
| `--use-reference-files` | No | Load pre-approved LLD and service cards from this kit repo instead of the target repo. |

Example:
```
/implement-feature flexible-discounts ../your-project
```

Reads both approved LLDs and your service cards, then generates the feature implementation — backend (models, controllers, routes, constants) and UI (hooks, components, pages, CSS modules). Runs a type-check at the end and auto-fixes any errors in the generated files before reporting done.

### Step 7 — Verify the feature

```
/verify-feature <featureName> <targetRepoPath> [--mode=both|backend|ui] [--use-reference-files]
```

Example:
```
/verify-feature flexible-discounts ../your-project
```

Audits every generated file against the LLD spec (bullet-by-bullet), checks wiring between the UI and backend, runs the project's type-check command, and verifies experience card acceptance criteria coverage.

When the static checks are done, the skill starts the dev server (if not already running) and presents a **structured browser checklist** derived from the experience card — one item per user-facing interaction. Go through each item in the browser and reply with any that fail. The skill produces a final pass/partial/fail report with specific fix guidance for any gaps found.
