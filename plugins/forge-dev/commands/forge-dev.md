---
description: Implement features from the roadmap with automated exploration, implementation, testing, review, and fixes
---

# Feature Implementation Workflow

You are orchestrating a multi-stage feature implementation workflow for **The Forge** project. Your job is to coordinate specialized agents to implement high-quality features from the project roadmap efficiently.

## Project Context

The Forge is a Laravel 12 + Vue 3 + Inertia v2 application. Key details:
- **Backend**: Laravel 12, Pest tests, Actions for business logic, thin controllers, Form Requests
- **Frontend**: Vue 3, Inertia v2, shadcn-vue (new-york-v4), Tailwind CSS v4, TypeScript
- **Database**: MySQL (dev and test)
- **Real-time**: Laravel Reverb, Echo
- **Routes**: Wayfinder (typed route helpers)
- **Design**: Linear-style dark theme (charcoal base, purple accent, monospace metadata, minimal borders)
- **State**: No Vuex/Pinia — Inertia page props + provide/inject from layouts
- **Forms**: useForm() from Inertia for all form submissions
- **Spec**: `docs/specs.md` | **Architecture**: `docs/tech-stack.md` | **Roadmap**: `docs/ROADMAP.md`

## Input

The user has requested: "$ARGUMENTS"

---

## Phase 1: Understand the Request

1. Create a todo list tracking all phases of this workflow.
2. Read `docs/ROADMAP.md` to understand the full implementation roadmap.
3. Parse the user's request to identify:
   - Which roadmap item(s) to implement (e.g., "1.1 Workspace & Members", "Migration: workspaces table")
   - Whether it's backend, frontend, or both
   - The specific unchecked `[ ]` items that need implementation
4. If the request is ambiguous (e.g., "workspace stuff" could mean many items), present the matching items from ROADMAP.md and ask the user to confirm which specific items to implement.
5. Summarize what you'll implement. If there are more than 8 items, confirm with the user before proceeding.

---

## Phase 2: Codebase Exploration

Launch **2-3 feature-explorer agents in parallel**, each with a different focus tailored to the feature area:

- **Agent 1 (Source Context)**: "Explore the existing source code related to [feature area]. Find controllers, actions, models, migrations, routes, services, form requests, policies, and resources that already exist and are relevant to what we need to build. For frontend: find pages, components, layouts, composables, types. Trace key code paths. List the 5-10 most important files."

- **Agent 2 (Patterns & Conventions)**: "Explore the existing codebase to understand how things are done. Study: file structure and naming, coding style, controller patterns (thin controllers + Actions), Form Request patterns, Inertia page/component patterns, shadcn-vue usage, factory patterns, test patterns (Pest). Find 3-5 files most similar to what we need to create for [feature area]. Also check route files, middleware, and any base classes or traits."

- **Agent 3 (Dependencies)**: "Explore what [feature area] depends on. Find models, migrations, factories, and routes that already exist and that our implementation builds upon. Check database schema for existing tables we'll reference. Check existing enums, services, and any shared code. Document what exists and what's missing. Also read `docs/specs.md` and `docs/tech-stack.md` for architecture decisions relevant to this feature."

After agents complete:
- Read ALL the key files they identified (this is critical -- you need this context for implementation)
- Build a comprehensive understanding of:
  - The existing code and how it works
  - The project's conventions and patterns (copy them exactly)
  - Available factories, helpers, traits, and base classes
  - Routes, middleware, validation patterns
  - Database schema and relationships
  - Frontend component patterns, layout usage, shadcn-vue usage
  - What already exists vs what needs to be created

---

## Phase 3: Implementation

Based on the exploration findings, launch **feature-implementer agent(s)**:

- **Backend-only items** (migrations, models, enums, factories, actions, controllers, requests, policies, tests): Launch 1 backend feature-implementer agent.
- **Frontend-only items** (Vue pages, components, composables, types): Launch 1 frontend feature-implementer agent.
- **Mixed items**: Launch both backend and frontend implementer agents in parallel if the work is independent (e.g., backend API + frontend page). If the frontend depends on backend (e.g., needs Inertia props from a controller), launch backend first, then frontend.

Each agent receives in its prompt:
- The specific roadmap items to implement (copy the unchecked items from ROADMAP.md)
- The key files identified during exploration (list file paths and summarize their content)
- Patterns and conventions found (describe them specifically with examples)
- Dependencies: what already exists and what the implementation builds upon
- For backend: database schema, existing models/factories, route patterns, controller patterns, Action patterns, Form Request patterns
- For frontend: existing page/component patterns, layout usage, shadcn-vue components used, useForm() patterns, TypeScript type patterns

**CRITICAL**: Provide the agent with enough context to implement the feature WITHOUT needing to re-explore the codebase. Include file contents or detailed summaries from your exploration reads.

Wait for ALL implementer agents to complete before proceeding.

After implementation completes:
- Read the implemented files to understand what was created
- Note which files were created/modified and which roadmap items they cover

---

## Phase 4: Test

Run the relevant tests to verify the implementation works:

For **backend** changes:
```bash
php artisan test --filter=RelevantTestFile
```

For **frontend** changes:
```bash
npm run build
```

- Run tests for each implemented file/area
- If tests fail:
  - Analyze the error messages carefully
  - Fix obvious issues: missing imports, wrong method signatures, incorrect factory usage, typos, missing route definitions
  - Re-run after each fix
- If a test keeps failing after 3 fix attempts, comment it out with a `// TODO: fix - [reason]` comment and note it for the review phase
- Continue until all tests pass (or are documented as blocked)

---

## Phase 5: Quality Review

Launch **feature-reviewer agents in parallel** -- spawn **one agent per created/modified file**:

For each file, tell the agent:
- "Review this file: [path/to/file]. This implements these specific ROADMAP.md items: [list items]. Do NOT flag missing functionality for other unchecked items in the same ROADMAP.md section -- those are scheduled for separate implementation. Verify correctness, convention adherence, and completeness against the listed items."

Wait for ALL reviewer agents to complete.

After all reviewers complete:
- Consolidate all findings across files
- Filter to only findings with **confidence >= 75**
- If there are NO significant findings, skip Phase 6 and go to Phase 7
- If there ARE findings, present them to the user organized by severity (Critical then Important) and proceed to Phase 6

---

## Phase 6: Fix Findings

**DO NOT fix findings yourself.** You are the orchestrator -- you delegate, you do not implement. Launch a **feature-fixer agent** with:
- All review findings (confidence >= 75) from Phase 5
- The file paths to fix
- Key context from the exploration phase (relevant source code, conventions)

The fixer should:
- Address each finding systematically
- Run the tests after fixes to ensure they still pass
- Report what was fixed and what was skipped (with reason)

After the fixer completes:
- Verify all tests pass by running them one final time

---

## Phase 7: Finalize

1. Run the complete test suite for all affected areas one final time:
   ```bash
   php artisan test
   ```
   And for frontend:
   ```bash
   npm run build
   ```
2. Update `docs/ROADMAP.md`:
   - Change `[ ]` to `[x]` for each roadmap item that was successfully implemented
   - Do NOT check off items that were not fully implemented
3. Present a summary:
   - Features implemented (list of roadmap items checked off)
   - Files created/modified (with full paths)
   - Test results (pass/fail counts)
   - Any items NOT implemented and why
   - Any issues encountered and how they were resolved
   - Any TODOs or known limitations
4. Mark all todos complete

---

## Important Guidelines

- **Delegate all substantial code work to agents.** Implementation (Phase 3) goes to feature-implementer agents. Reviewer findings (Phase 6) go to the feature-fixer agent. You may only make trivial fixes yourself in Phase 4 (missing imports, typos) to get tests passing -- anything beyond that must be delegated.
- **Never skip the exploration phase.** Understanding the codebase is essential for writing correct, convention-matching code.
- **Match conventions exactly.** The implementer must copy the style of existing code, not invent new patterns.
- **Run tests early and often.** Don't wait until the end to discover failures.
- **One reviewer per file.** Never send multiple files to a single reviewer agent.
- **Fix only what's reported.** The fixer should not refactor or "improve" beyond the findings.
- **Be honest about failures.** If something can't be implemented correctly, document why rather than hiding it.
- **Backend before frontend.** When items have dependencies, implement backend first so the API and data layer exist before the frontend is built.
- **Always write tests.** Every backend implementation should include Pest tests. Every migration should have a relationship test. Every controller should have feature tests.
