---
description: Implement a milestone from the roadmap with automated exploration, implementation, testing, review, and fixes
---

# Feature Implementation Workflow

You are orchestrating a multi-stage feature implementation workflow. Your job is to coordinate specialized agents to implement one roadmap milestone at a time, delivering complete, visible features.

## Project Context

Read the project's docs to understand the stack and architecture:
- **Spec**: `docs/specs.md` | **Architecture**: `docs/tech-stack.md` | **Roadmap**: `docs/ROADMAP.md`
- **Conventions**: `docs/conventions.md` | **Tickets**: `docs/tickets/`

## Input

The user has requested: "$ARGUMENTS"

---

## Phase 1: Identify the Milestone

1. Create a todo list tracking all phases of this workflow.
2. Read `docs/ROADMAP.md` to find the next milestone to implement.
   - If `docs/ROADMAP.md` doesn't exist, tell the user: "No roadmap found. Run `/forge-roadmap` first to generate one from your specs." Do not proceed without a roadmap.
   - If the user specified a milestone: use that one.
   - If the user said "next" or gave no specifics: find the first milestone with unchecked `[ ]` items.
3. **Verify milestone ordering**: Check that all previous milestones are complete (`[x]`). If a prior milestone has unchecked items, warn the user — milestones should be completed in order. Suggest completing the earlier milestone first unless the user explicitly overrides.
4. **Read the ticket file** for this milestone from `docs/tickets/`. The ticket contains all implementation details: behaviors, data model, interface contracts, test coverage, and verification checks.
   - If no ticket file exists, tell the user: "No ticket found for this milestone. Run `/forge-ticket [milestone]` first to generate one." Do not proceed without a ticket.
5. Summarize to the user:
   - Milestone name and goal (from the ticket)
   - Key deliverables
   - Verification gate (what they'll see when it's done)
   - Confirm before proceeding.

---

## Phase 2: Conventions & Codebase Exploration

### Step 1: Load Conventions (cached — runs once per project)

Check if `docs/conventions.md` exists:

- **If it exists** → Read it. Skip to Step 2.
- **If it does NOT exist** → Determine if this is an existing or new project:
  - Check for existing code (look for `app/Http/Controllers/`, `app/Models/`, `resources/js/pages/`, or `tests/`)
  - **Existing project (has code)** → Launch a **feature-explorer agent** focused on conventions: "Explore the existing codebase to understand how things are done. Study: file structure and naming, coding style, controller patterns, Form Request patterns, page/component patterns, UI library usage, factory patterns, test patterns. Find 3-5 example files for each pattern. Output a comprehensive conventions document." Save the agent's output to `docs/conventions.md`.
  - **New project (no code yet)** → Copy the bundled defaults from the plugin's `templates/conventions.md` to `docs/conventions.md`.

### Step 2: Feature-Specific Exploration (always runs — varies per milestone)

Launch **2 feature-explorer agents in parallel**, each focused on the specific milestone:

- **Agent 1 (Source Context)**: "Explore the existing source code related to [milestone area]. Find controllers, actions, models, migrations, routes, form requests, policies, and resources that already exist and are relevant. For frontend: find pages, components, layouts, composables, types. Trace key code paths. List the 5-10 most important files."

- **Agent 2 (Dependencies)**: "Explore what [milestone area] depends on. Find models, migrations, factories, and routes that already exist and that our implementation builds upon. Check database schema for existing tables we'll reference. Check existing enums, services, and any shared code. Document what exists and what's missing."

After agents complete:
- Read ALL the key files they identified (this is critical — you need this context for implementation)
- Combine the conventions + ticket + exploration findings into a comprehensive understanding

---

## Phase 3: Implementation

Implement the milestone as a **vertical slice** using the ticket as the source of truth.

**The ticket's Behavior, Data Model, and Interface Contract sections define exactly what to build.** The implementer agents should follow these specifications precisely.

**Launch order:**
1. **Backend first**: Launch 1 feature-implementer agent for migrations, models, enums, factories, actions, controllers, form requests, policies, routes, and backend tests.
2. **Frontend second** (after backend completes): Launch 1 feature-implementer agent for pages, components, composables, TypeScript types, and frontend work. This agent receives the actual backend files just created (controllers, routes, resources) so the frontend matches the real API.

If the milestone is backend-only (e.g., scaffold, infrastructure), skip the frontend agent.

Each agent receives in its prompt:
- The ticket's relevant sections (Behavior, Data Model, Interface Contract, Scope, Constraints)
- The key files identified during exploration (list file paths and summarize their content)
- Patterns and conventions from `docs/conventions.md`
- Dependencies: what already exists and what the implementation builds upon
- For frontend: the actual backend files from step 1

**CRITICAL**: Provide the agent with enough context to implement WITHOUT needing to re-explore. Include file contents or detailed summaries from your exploration reads.

After implementation completes:
- Read the implemented files to understand what was created
- Note which files were created/modified

---

## Phase 4: Test & Verify

### Step 1: Automated Tests

Run the tests created during implementation:

```bash
php artisan test --filter=RelevantTestFile
```

```bash
npm run build
```

- If tests fail:
  - Analyze error messages carefully
  - Fix obvious issues: missing imports, wrong method signatures, incorrect factory usage, typos
  - Re-run after each fix
- If a test keeps failing after 3 fix attempts, comment it out with `// TODO: fix - [reason]` and note it

### Step 2: Verification Gate (MANDATORY)

Read the ticket's **Verify** section. For each item:

- **Automated checks** → Run the full test suite and build
- **Browser/visual checks** → Present the description to the user and ask them to confirm manually

**Do NOT proceed until the user confirms the verification gate passes.**

If the user reports issues:
1. Analyze the issue
2. Delegate fixes to a feature-fixer agent (do not fix substantial issues yourself)
3. Re-run tests
4. Re-verify with the user

This is the most important checkpoint. It catches "nothing loads" and "wrong feature" before more work is piled on top.

---

## Phase 5: Comprehensive Testing

After the verification gate passes, run comprehensive tests (unit, feature, and browser) in a **separate Claude session** to keep context clean. This uses the test-dev plugin's specialized agents.

### Step 1: Prepare

Identify the ticket file path for this milestone (e.g., `docs/tickets/m01-workspace.md`). The ticket's Test Coverage section contains the test items to implement.

### Step 2: Invoke test-dev

Run the test-dev workflow in a subprocess:

```bash
claude -p "Implement all tests from the Test Coverage section of [TICKET_FILE_PATH]. Read the ticket file first to understand the feature and find the test items (Unit, Feature, Browser). Use test-explorer agents to understand the source code and test patterns, test-implementer agents to write the tests, then run them. Review with test-reviewer agents and fix issues with test-fixer agents. Update the ticket's Test Coverage checkboxes when done." \
  --allowedTools "Read,Write,Edit,Bash,Glob,Grep,Agent" \
  --dangerously-skip-permissions
```

Replace `[TICKET_FILE_PATH]` with the actual ticket path.

### Step 3: Verify

After the subprocess completes:
- Run `php artisan test` to confirm all tests pass
- Read the ticket file to check which Test Coverage items were checked off
- If tests fail, attempt trivial fixes; for substantial failures, delegate to a feature-fixer agent
- Note any test items that were NOT implemented

---

## Phase 6: Quality Review

Launch **feature-reviewer agents in parallel** — one agent per created/modified file (including test files from Phase 5):

For each file, tell the agent:
- "Review this file: [path]. It implements these items from the ticket: [list]. Verify correctness, convention adherence, and completeness against the ticket's Behavior and Interface Contract sections. Do NOT flag missing functionality for items outside this ticket's scope."

Wait for ALL reviewer agents to complete.

After all reviewers complete:
- Consolidate findings, filter to confidence >= 75
- If no significant findings: skip Phase 7, go to Phase 8
- If there ARE findings: present to user by severity, proceed to Phase 7

---

## Phase 7: Fix Findings

**DO NOT fix findings yourself.** Launch a **feature-fixer agent** with:
- All review findings (confidence >= 75)
- The file paths to fix
- Key context from exploration + ticket

After the fixer completes:
- Verify all tests pass one final time

---

## Phase 8: Finalize

1. Run the complete test suite:
   ```bash
   php artisan test
   ```
   ```bash
   npm run build
   ```
2. Update `docs/ROADMAP.md`:
   - Change `[ ]` to `[x]` for each completed item in this milestone
   - Do NOT check off items that were not fully implemented
3. Update the ticket file: change **Status** to `done` (or `in-progress` if partially complete)
4. Present a summary:
   - **Milestone completed**: name and what was delivered
   - **Verification gate**: what the user should see
   - Files created/modified (with full paths)
   - Test results (pass/fail counts)
   - Any items NOT implemented and why
   - Any TODOs or known limitations
5. **Test coverage summary**: report which Test Coverage items from the ticket were completed (checked off by test-dev) and which remain unchecked
6. **Next milestone**: show the name and brief description of what comes next
7. Mark all todos complete

---

## Important Guidelines

- **One milestone at a time.** Each `/forge-dev` invocation implements exactly one milestone. Sequential — never skip ahead.
- **Ticket is the source of truth.** The ticket's Behavior and Interface Contract sections define what to build. Follow them precisely.
- **No ticket, no implementation.** If the ticket file doesn't exist, stop and tell the user to run `/forge-ticket` first.
- **Verification gate is mandatory.** The user must confirm the feature works visually before proceeding.
- **Backend before frontend within a milestone.** Backend completes before frontend starts.
- **Delegate all substantial code work to agents.** You may only make trivial fixes (imports, typos) in Phase 4.
- **Never skip exploration.** Understanding the codebase is essential for convention-matching code.
- **Match conventions exactly.** Copy existing patterns, never invent new ones.
- **One reviewer per file.** Never send multiple files to a single reviewer.
- **Fix only what's reported.** The fixer must not refactor beyond the findings.
- **Be honest about failures.** Document what couldn't be done rather than hiding it.
- **Comprehensive testing is automated.** After the verification gate, forge-dev invokes test-dev in a separate session to implement all tests from the ticket's Test Coverage section (unit, feature, and browser).
