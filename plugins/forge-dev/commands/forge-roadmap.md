---
description: Generate a ROADMAP.md from project specs by decomposing features into vertical-slice milestones
---

# Roadmap Generation

You are generating a structured development roadmap from the project's specifications. The roadmap decomposes vague, narrative specs into ordered, implementable milestones that `/forge-dev` can execute one at a time.

## Input

The user has requested: "$ARGUMENTS"

---

## Step 1: Read Project Specs

Read all available project documentation:

- `docs/specs.md` — domain model, features, navigation, views, design decisions
- `docs/tech-stack.md` — architecture and technology choices (if it exists)
- Any other files in `docs/` that provide context

If `docs/specs.md` doesn't exist, stop and tell the user: "No specs found at `docs/specs.md`. Place your project specifications there first."

---

## Step 2: Extract Features and Map Dependencies

Analyze the specs to identify:

1. **All entities/models** — what data objects exist and their relationships
2. **All features** — what users can see and do (pages, interactions, workflows)
3. **Dependencies** — which entities reference others, which features require which models, which pages need which APIs
4. **Complexity** — simple CRUD vs. multi-step workflows vs. integrations

Build a mental dependency graph: what must exist before what else can be built.

---

## Step 3: Surface Gaps and Decisions

**This is critical.** Before creating the roadmap, review the specs for completeness. Present to the user:

### Gaps — Missing Information

For each gap, explain:
- What's missing
- Why it matters for the roadmap (which milestones are affected)
- A suggested default if the user doesn't have a preference

Examples of gaps to look for:
- Entities mentioned but not fully defined (missing columns, relationships, or behaviors)
- Features described vaguely ("users can manage X" — what does "manage" include?)
- Undefined user flows (what happens after action X? where does the user land?)
- Missing validation rules, error handling, or edge cases
- Authorization model gaps (who can do what?)
- Integration details missing (how do external systems connect?)

### Decisions — Choices That Affect the Roadmap

For each decision, present:
- The question
- 2-3 options
- Your recommendation with reasoning

Examples:
- "Should workspace creation and project creation be one milestone or two? I recommend two — workspace first, then projects — because projects depend on workspaces and each is complex enough for its own session."
- "The spec mentions real-time updates but doesn't detail when they're needed. I recommend deferring real-time to a later milestone and building basic CRUD first."

### What to do

- If there are **blocking gaps** (can't create a reasonable roadmap without answers): present them and wait for the user's response before proceeding.
- If there are **non-blocking gaps** (can proceed with reasonable defaults): list them with your defaults and proceed, noting the assumptions in the roadmap.
- If there are **no significant gaps**: note this and proceed directly.

---

## Step 4: Create the Roadmap

Decompose the specs into milestones following these principles:

1. **Vertical slices** — each milestone delivers from database to UI. No "all migrations first" or "all backend first."
2. **Incremental value** — each milestone adds something visible in the browser. After any milestone, the app works (even if incomplete).
3. **Small milestones** — each completable in one `/forge-dev` session. If a feature is too large, split it into milestones that each deliver a usable subset.
4. **Strict ordering** — dependencies flow forward. Milestone N must complete before N+1 starts.
5. **Verification gates** — each milestone ends with a concrete description of what the user sees in the browser.
6. **Foundation first** — early milestones establish core entities and layouts; later milestones build on them.

### Milestone structure

Each milestone should include:

```markdown
## Milestone N: [Short name — what the user gets]

> After this milestone: [What the user sees when they open the app. Be specific and visual.]

### Backend
- [ ] Migration: `table_name` (key columns...)
- [ ] Model: `ModelName` with relationships
- [ ] Factory: `ModelNameFactory` with states
- [ ] Enum: `EnumName` (if needed)
- [ ] Action: `ActionName`
- [ ] Controller: `ControllerName@method1`, `@method2`
- [ ] Form Request: `RequestName`
- [ ] Policy: `PolicyName`
- [ ] Routes: `METHOD /path`

### Frontend
- [ ] Page: `SectionName/PageName.vue`
- [ ] Component: `ComponentName.vue` (if needed)

### Tests
- [ ] Feature test: [what it tests]
- [ ] Unit test: [what it tests]
- [ ] Browser test: [what the user does and sees]

### Verify
- [ ] `php artisan test` passes
- [ ] `npm run build` succeeds
- [ ] [Describe what you see in the browser — be specific and visual]
```

### Splitting guidelines

- If a feature involves 3+ new models with UI, consider splitting: models+list first, then create/edit, then detail views.
- Group related items: a model, its controller, its page, and its tests belong together.
- Keep auth/permissions in the same milestone as the feature they protect (unless the auth system itself needs a dedicated milestone).
- Browser tests should be included in each milestone that has UI.

Write the roadmap to `docs/ROADMAP.md`.

---

## Step 5: Review and Confirm

Present to the user:

1. **Total milestones** and estimated scope
2. **Summary table**:
   | # | Milestone | What the user gets |
   |---|-----------|-------------------|
   | 1 | Name | Description |
   | 2 | Name | Description |
   | ... | ... | ... |
3. **Key ordering decisions** — why certain features come before others
4. **Assumptions made** — gaps that were filled with defaults (from Step 3)
5. **Deferred items** — features from the spec that were intentionally left for later milestones or out of scope

Ask: "Does this roadmap look right? Want to reorder, split, merge, or remove anything?"

If the user wants changes, edit `docs/ROADMAP.md` directly. Iterate until confirmed.

---

## Notes

- The roadmap is consumed by `/forge-ticket` (to generate detailed tickets per milestone) and `/forge-dev` (to implement milestones in order).
- Each milestone becomes one `/forge-ticket` session and one `/forge-dev` session.
- Getting the roadmap right saves significant time downstream — a poorly scoped milestone leads to failed implementations.
- Include browser tests in every milestone that has UI. These are critical for catching "tests pass but the UI is broken" situations.
- When in doubt, make milestones smaller rather than larger. It's easy to merge two small milestones but hard to recover from a failed large one.
