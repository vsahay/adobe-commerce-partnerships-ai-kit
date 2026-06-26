# Generating the UI LLD

**Frequency:** Once per feature  
**Prerequisites:**
- UI service card is complete and reviewed (UI service cards exist)
- Feature Experience Card is complete
- Backend LLD is complete and is approved

---

## What This Step Does

This step uses the AI to generate a **UI Low-Level Design (LLD)** — a precise engineering plan for all frontend changes required to implement the feature.

Like the backend LLD, this is not generated code — it is a structured specification describing which routes to add, which components to build, how fetch hooks should work, and what state changes are required.

The AI produces the UI LLD by reading four inputs together:
1. Your experience card — what the user sees and does at each step
2. Your backend LLD — what data is available from the backend and its exact shape
3. Your UI service cards — how your frontend is coded
4. Your Figma design (if a URL is present in the experience card) — visual specifications for components

If the UI LLD does not accurately reflect the experience card, the generated code will not match your intent — and this will not be visible until QA.

---

## Running the Skill

Open Claude Code in the `partner-ai-kit` directory and run:

```
/apply-experience-card <featureName> <path-to-frontend-repo>
```

**Example:**

```
/apply-experience-card anytimeUpgrade ../my-partner-frontend
```

If a Figma URL is present in the experience card, the AI will fetch the design automatically. See the Figma section below.

---

## What Gets Produced

The skill produces a single file at:

```
.claude/feature-specs/<featureName>/reference-files/BridgeLLD/ui/<feature-name>-ui-lld.md
```

The LLD contains 10 sections:

| Section | What it contains |
|---|---|
| **Summary** | One-paragraph description of the UI changes; notes Figma source if used |
| **Route Changes** | New routes to add, params, auth guard, stores required |
| **Component Specs** | Each new component: props, state, visual specs (from Figma if available) |
| **Fetch Hook Specs** | Each new hook: endpoint URL, request shape, response handling, error states |
| **State & Context Changes** | Changes to existing stores or context; new state required |
| **Data Flow** | Step-by-step data flow per user action, from UI event to rendered state |
| **Reuse Summary** | Existing components, hooks, and utilities to reuse |
| **Design Decisions** | Non-obvious choices and reasoning |
| **Acceptance Criteria Coverage** | How each acceptance criterion from the experience card is met |
| **Out of Scope** | What the UI will not handle in this implementation |

---

## Figma Integration

If your experience card includes a Figma URL, the AI will fetch the design and extract visual specifications — spacing, color tokens, component structure, typography. These are incorporated into the Component Specs section of the UI LLD.

**To ensure Figma works correctly:**

1. The Figma URL in your experience card must include the node ID. A URL without a node ID points to the whole file, not a specific frame.

   Example of a correct URL:
   ```
   https://www.figma.com/file/xxxx/feature-name?node-id=12%3A345
   ```

   To get the node ID: In Figma, right-click the frame → "Copy link" — this includes the node ID automatically.

2. The Figma file must be publicly accessible, or your Claude Code session must have Figma access configured.

3. If Figma is unavailable or the URL is wrong, the AI will proceed without visual specs. The LLD will describe components functionally but without visual guidance. In this case, note the gap in the UI LLD review and fill in visual specs manually before approving.

---

## Reviewing the UI LLD — Decision Gate DG-4

**Partners must review and approve this LLD before proceeding.**

Open the [Decision Gate Guide — DG-4](../decision-gates.md#dg-4-ui-lld-review) for the full review checklist.

### Review focus:
- Route definitions are correct and match the existing routing structure
- Component breakdown is sensible — each component has a clear single responsibility
- Fetch hook specs match the backend LLD exactly: same URL, same request body shape, same response field names
- State changes are minimal — not introducing global state for things that should be local
- Reuse decisions are accurate — the referenced components and hooks actually exist
- The step-by-step user journey in the UI LLD matches the experience card exactly
- Every error state from the experience card appears as a named UI state in the LLD
- If Figma was provided, the visual specs in the Component Specs section reflect the design intent
- The "Out of Scope" list does not exclude anything that was in the experience card

**The Review of the UI LLD is the last opportunity to catch a mismatch between product intent and implementation plan.** After this, the next human review is the engineer's code review — which checks correctness, not whether the feature does what was intended.

---

## Editing the UI LLD

Edit the LLD file directly to make corrections, same as the backend LLD.

Common edits:
- Correct a component that does not match the Figma design
- Add a missing error state that was in the experience card but not in the LLD
- Change a UI flow (e.g., "navigate to confirmation page" → "show confirmation modal in place")
- Add visual detail if Figma was not available
- Correct a fetch hook URL that does not match the backend LLD
- Fix a response field name that does not match the backend LLD
- Update a component path to match the project's directory structure
- Change a state design decision

After reviews are done and edits applied, the Partner's should explicitly agree the LLD is approved before the code generation begins.

---

## After This Step

Both LLDs are now approved. Proceed to **[Generating Code](generate-code.md)**.