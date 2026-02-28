---
description: Implement tests with automated exploration, implementation, review, and fixes
---

# Test Development Workflow

You are orchestrating a multi-stage test development workflow. Your job is to coordinate specialized agents to implement high-quality tests efficiently.

## Input

The user has requested: "$ARGUMENTS"

---

## Phase 1: Understand the Request

1. Create a todo list tracking all phases of this workflow.
2. Read `tests/COVERAGE.md` to understand the test coverage matrix.
3. Parse the user's request to identify:
   - Which test area(s) to implement (e.g., "Appointments Feature API tests", "Exercises Unit tests")
   - Whether it's Unit, Feature (API), or Browser tests
   - The specific test items from COVERAGE.md that are currently unchecked `[ ]`
4. If the request is ambiguous (e.g., "appointments tests" could mean Unit or Feature), present the options from COVERAGE.md and ask the user to confirm which specific tests to implement.
5. Summarize what you'll implement. If there are more than 5 test items, confirm with the user before proceeding.

---

## Phase 2: Codebase Exploration

Launch **2-3 test-explorer agents in parallel**, each with a different focus tailored to the test area:

- **Agent 1 (Source Code)**: "Explore the source code for [feature area]. Find the controllers, actions, services, models, routes, form requests, policies, and resources involved. Trace the key code paths from route to response. List the 5-10 most important files."

- **Agent 2 (Test Patterns)**: "Explore existing tests in this project to understand testing patterns and conventions. Look at: test directory structure, test helpers and base classes, Pest vs PHPUnit usage, authentication in tests, factory patterns, assertion styles. Find 3-5 test files most similar to what we need to write for [feature area]. Also check tests/Pest.php and phpunit.xml for configuration."

- **Agent 3 (Data Layer)** _(launch only if the tests involve database operations)_: "Explore the database structure for [feature area]. Find migrations, factories, seeders, and model relationships. Document the key tables, columns, foreign keys, and factory definitions including all states."

After agents complete:
- Read ALL the key files they identified (this is critical — you need this context for implementation)
- Build a comprehensive understanding of:
  - The code being tested and how it works
  - The project's testing conventions and patterns (copy them exactly)
  - Available factories, helpers, and test traits
  - API routes, middleware, validation rules, and expected responses
  - Database schema and relationships

---

## Phase 3: Implementation

Based on the exploration findings, launch **test-implementer agent(s)**:

- **Single area**: Launch 1 test-implementer agent with full context from exploration.
- **Multiple areas**: Launch parallel test-implementer agents, one per area. Each gets the exploration context relevant to their area.

Each agent receives in its prompt:
- The specific test items to implement (copy the unchecked items from COVERAGE.md)
- The key files identified during exploration (list file paths)
- Testing patterns and conventions found (describe them specifically)
- Factory definitions and available test helpers
- Route definitions and expected request/response formats

**CRITICAL**: Provide the agent with enough context to write tests WITHOUT needing to re-explore the codebase. Include file contents or detailed summaries from your exploration reads.

Wait for ALL implementer agents to complete before proceeding.

After implementation completes:
- Read the implemented test files to understand what was created
- Note which files were created and which test items they cover

---

## Phase 4: Run Tests

Run the implemented tests to verify they pass:

```bash
php artisan test path/to/test/file.php
```

- Run each test file individually first, then together
- If tests fail:
  - Analyze the error messages carefully
  - Fix obvious issues: missing imports, wrong method signatures, incorrect factory usage, assertion mismatches
  - Re-run after each fix
- If a test keeps failing after 3 fix attempts, comment it out with a `// TODO: fix - [reason]` comment and note it for the review phase
- Continue until all tests pass (or are documented as blocked)

---

## Phase 5: Quality Review

Launch **test-reviewer agents in parallel** — spawn **one agent per test file** created:

For each test file, tell the agent:
- "Review this test file: [path/to/file.php]. Check it against ONLY these specific COVERAGE.md items: [list items]. Do NOT flag missing coverage for other unchecked items in the same COVERAGE.md section — those are scheduled for separate implementation. Verify completeness against the listed items, correctness, and adherence to project conventions."

Wait for ALL reviewer agents to complete.

After all reviewers complete:
- Consolidate all findings across test files
- Filter to only findings with **confidence >= 75**
- If there are NO significant findings, skip Phase 6 and go to Phase 7
- If there ARE findings, present them to the user organized by severity (Critical then Important) and proceed to Phase 6

---

## Phase 6: Fix Findings

**DO NOT fix findings yourself.** You are the orchestrator — you delegate, you do not implement. Launch a **test-fixer agent** with:
- All review findings (confidence >= 75) from Phase 5
- The test file paths to fix
- Key context from the exploration phase (relevant source code, conventions)

The fixer should:
- Address each finding systematically
- Run the tests after fixes to ensure they still pass
- Report what was fixed and what was skipped (with reason)

After the fixer completes:
- Verify all tests pass by running them one final time

---

## Phase 7: Finalize

1. Run the complete test suite for all affected test files one final time:
   ```bash
   php artisan test path/to/tests/
   ```
2. Update `tests/COVERAGE.md`:
   - Change `[ ]` to `[x]` for each test item that was successfully implemented
   - Add the test file name reference after each item (e.g., `— NewTestFile.php`)
3. Present a summary:
   - Tests implemented (list of test items checked off)
   - Files created/modified
   - Test results (pass/fail counts)
   - Any items NOT implemented and why
   - Any tests commented out as TODO and why
4. Mark all todos complete

---

## Important Guidelines

- **Delegate all substantial code work to agents.** Implementation (Phase 3) goes to test-implementer agents. Reviewer findings (Phase 6) go to the test-fixer agent. You may only make trivial fixes yourself in Phase 4 (missing imports, typos) to get tests passing — anything beyond that must be delegated.
- **Never skip the exploration phase.** Understanding the codebase is essential for writing correct tests.
- **Match conventions exactly.** The implementer must copy the style of existing tests, not invent new patterns.
- **Run tests early and often.** Don't wait until the end to discover failures.
- **One reviewer per file.** Never send multiple test files to a single reviewer agent.
- **Fix only what's reported.** The fixer should not refactor or "improve" beyond the findings.
- **Be honest about failures.** If a test can't be made to pass, document why rather than hiding it.
