# FR-1 — Backend Service Card Generation

**Who runs this:** Engineer  
**Frequency:** Once per project onboarding, then again after significant codebase restructuring  
**Prerequisite:** None — this is the first step in onboarding a project

---

## What This Step Does

This step scans your backend codebase and produces a set of **service card files** — structured documents that describe how your backend is coded. These cards are not documentation for humans to read casually; they are precision inputs that the AI reads before generating any code for your project.

The cards capture things like:
- How your project names controllers, models, and route handlers
- How auth tokens are injected into outbound API calls
- What external dependencies your service uses, and how they behave on failure
- Your database schema and entity relationships
- Your error handling and validation patterns

Every piece of backend code the toolkit generates will follow the patterns in these cards. If the cards are accurate, generated code will look like it was written by a member of your team. If the cards are inaccurate, generated code will have subtle pattern mismatches that compound across the codebase.

---

## Before You Run

- You must have access to the backend repository path on your machine.
- Claude Code must be installed and running.
- The backend repo should be in a stable state — avoid running this during an active large refactor, as the scan will capture in-progress patterns.

---

## Running the Skill

Open Claude Code in the `partner-ai-kit` directory and run:

```
/generate-backend-service-card <path-to-backend-repo> --feature-name <featureName>
```

**Example:**

```
/generate-backend-service-card ../my-partner-backend --feature-name anytimeUpgrade
```

The skill will scan your backend repo across four phases:
1. **Locate & Scaffold** — identifies the project structure and creates output files
2. **Code Scan** — reads controllers, models, routes, connectors, and validation patterns
3. **Config Scan** — reads build config, environment variables, secrets, and deployment config
4. **Assembly** — writes all findings into the structured card files

This will take a few minutes depending on the size of your repo. Do not interrupt it.

---

## What Gets Produced

The skill produces **9 card files** and **4 standards files** at:

```
.claude/feature-specs/<featureName>/reference-files/BridgeServiceCards/backend/
```

### Card Files

| File | What it captures |
|---|---|
| `SERVICE_CARD.md` | Service identity: name, owner, repo, tier, what the service owns, what is fragile |
| `MODULE_INDEX.md` | Capability catalogue: what the service can do, mapped to implementation files |
| `CONTRACTS.md` | Every inbound interface: HTTP endpoints and async event consumers |
| `CONNECTORS.md` | Every outbound dependency: external APIs, auth approach, timeouts, retry behavior, error posture |
| `BUILD_CONFIG.md` | Tech stack, key dependencies, secrets paths, build scripts |
| `CODE_PATTERNS.md` | Naming conventions, package layout, DTO patterns, error handling, validation approach |
| `PLATFORM.md` | Deployment topology, monitoring, feature flags, database connection |
| `DB_SCHEMA.md` | Entity-to-table mapping, columns, relationships |
| `CHANGELOG.md` | Append-only record of card changes (do not edit this manually) |

### Standards Files (copied verbatim — do not edit)

| File | What it defines |
|---|---|
| `CONNECTOR_STANDARDS.md` | Timeouts, retry rules, error classification |
| `EVENT_STANDARDS.md` | Async consumer/publisher rules |
| `LOGGING_STANDARDS.md` | Log levels, MDC fields |
| `API_DOCS_STANDARDS.md` | API annotation rules |

---

## Reviewing the Cards — Decision Gate DG-1

**Do not proceed to any feature workflow step until you have reviewed the cards.**

Open the [Decision Gate Guide — DG-1](../decision-gates.md#dg-1-backend-service-card-review) and complete the review checklist. Pay particular attention to:

- `CODE_PATTERNS.md` — if this is wrong, every generated file will have pattern mismatches
- `CONNECTORS.md` — if auth or retry behavior is wrong, generated API calls will be incorrect
- `MODULE_INDEX.md` — if capabilities are missing or wrong, the AI may re-implement things that already exist

Edit the card files directly to correct any inaccuracies. Do not re-run the generator to fix small errors — edit the files and commit the changes.

---

## When to Re-Run vs. When to Edit

The cards are meant to be maintained over time, not regenerated from scratch each time something changes. Use this table to decide:

| Situation | Action |
|---|---|
| A new external API dependency was added | Edit `CONNECTORS.md` — add the new connector entry |
| The auth token mechanism changed | Edit `CONNECTORS.md` and `CODE_PATTERNS.md` |
| A new database table was added | Edit `DB_SCHEMA.md` |
| The naming convention changed project-wide | Edit `CODE_PATTERNS.md` |
| A major framework upgrade changed the codebase structure | Re-run `/generate-backend-service-card` |
| Significant new modules were added and you are unsure what changed | Re-run `/generate-backend-service-card` |

When you re-run, the generator overwrites the existing card files. Back up any manual edits first, then apply them on top of the re-generated output.

---

## After This Step

Proceed to **[UI Service Card Generation](ui-service-card.md)** to produce the equivalent cards for your frontend codebase.

Once both the Service cards are complete and reviewed, your project is fully onboarded. You can then run the feature workflow for any feature.