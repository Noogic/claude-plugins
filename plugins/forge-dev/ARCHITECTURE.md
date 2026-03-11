# forge-dev Plugin - Architecture

## Overview

- **Name**: forge-dev
- **Version**: 1.0.0
- **Purpose**: Multi-stage feature implementation workflow. Implements one roadmap milestone at a time as a vertical slice.
- **Entry points**: `/forge-dev` (implement), `/forge-ticket` (generate ticket)
- **Hooks**: None

## Document Pipeline

```
specs.md            →  ROADMAP.md           →  tickets/m01-*.md     →  Implementation
(vision)               (plan + ordering)        (detail per milestone)   (forge-dev + test-dev)
```

| Document | Contains | Read when | Tokens |
|---|---|---|---|
| `specs.md` | Vision, domain model, navigation, views | Ticket generation only | Heavy, rarely |
| `ROADMAP.md` | Ordered milestones + checkboxes, no details | Every session start | Light |
| `tickets/m{NN}-*.md` | Behaviors, data model, contracts, test coverage | Only the current milestone | Medium, 1 at a time |
| `conventions.md` | Coding patterns and conventions | Every session (cached) | Medium, once |

## Commands

### /forge-dev (commands/forge-dev.md)

Implements one roadmap milestone. Reads the roadmap to find the milestone, reads its ticket file for details, explores the codebase, delegates to agents, tests, verifies with the user, and updates the roadmap.

### /forge-ticket (commands/forge-ticket.md)

Generates a detailed ticket file for a milestone. Reads specs.md + ROADMAP.md + existing code, launches a ticket-writer agent, saves to `docs/tickets/`.

## Agents

### ticket-writer (agents/ticket-writer.md)

- **Model**: sonnet
- **Color**: cyan
- **Tools**: Read, Write, Glob, Grep, Bash
- **Role**: Generates ticket files from specs + roadmap
- **Output**: `docs/tickets/m{NN}-{slug}.md` with behaviors, data model, interface contracts, scope, constraints, test coverage, and verification checks

### feature-explorer (agents/feature-explorer.md)

- **Model**: sonnet
- **Color**: yellow
- **Tools**: Glob, Grep, Read, Bash
- **Role**: Read-only codebase analysis before implementation
- **Focus**: Source context (existing code) and dependencies (what we build upon)

### feature-implementer (agents/feature-implementer.md)

- **Model**: sonnet
- **Color**: green
- **Tools**: Read, Write, Edit, Glob, Grep, Bash
- **Role**: Writes all implementation code following the ticket spec
- **Scope**: Backend (migrations, models, controllers, tests) or frontend (pages, components, types)

### feature-reviewer (agents/feature-reviewer.md)

- **Model**: sonnet
- **Color**: red
- **Tools**: Read, Grep, Glob, Bash
- **Role**: Read-only quality review of a single file against the ticket
- **Spawning**: One instance per file, always parallel
- **Confidence scoring**: 0-100 scale, only findings >= 75 reported

### feature-fixer (agents/feature-fixer.md)

- **Model**: sonnet
- **Color**: blue
- **Tools**: Read, Write, Edit, Glob, Grep, Bash
- **Role**: Fixes issues reported by reviewers
- **Rules**: Fix only what's reported, never break passing tests

## Workflow: /forge-ticket

1. Read ROADMAP.md → identify milestone
2. Read specs.md + conventions.md + existing code for context
3. Launch ticket-writer agent
4. Present summary to user for approval
5. Save to `docs/tickets/m{NN}-{slug}.md`

## Workflow: /forge-dev

### Phase 1: Identify the Milestone

- Read ROADMAP.md → find next unchecked milestone (or user-specified one)
- Verify milestone ordering: all prior milestones must be complete
- Read the ticket file for this milestone (required — no ticket = no implementation)
- Confirm scope and verification gate with user

### Phase 2: Conventions & Codebase Exploration

**Step 1 — Load conventions (cached, runs once per project)**:
- If `docs/conventions.md` exists: read it
- If not + existing code: launch explorer to extract, save to `docs/conventions.md`
- If not + empty project: copy from `templates/conventions.md`

**Step 2 — Feature-specific exploration (always runs)**:
- Launch 2 feature-explorer agents in parallel (source context + dependencies)
- Combine conventions + ticket + exploration findings

### Phase 3: Implementation (vertical slice)

- Backend first: 1 agent for migrations, models, controllers, routes, tests
- Frontend second: 1 agent for pages, components, types — receives actual backend files
- Ticket's Behavior and Interface Contract sections are the source of truth

### Phase 4: Test & Verify

- **Automated tests**: run test suite + build
- **Verification gate** (mandatory): user must confirm the feature works in browser
- Does NOT proceed until user confirms — catches "nothing loads" early

### Phase 5: Quality Review

- 1 reviewer agent per file, all in parallel
- Reviews against the ticket spec, not general expectations

### Phase 6: Fix Findings

- Feature-fixer agent addresses findings >= 75 confidence
- Re-verify tests pass after fixes

### Phase 7: Finalize

- Final test suite run
- Update ROADMAP.md checkboxes
- Update ticket status
- Suggest running `/test-dev` with the ticket's Test Coverage section
- Show next milestone

## test-dev Integration

forge-dev writes basic tests during implementation (Phase 3). For comprehensive test coverage:
- The ticket's Test Coverage section contains a COVERAGE.md-style checklist
- After forge-dev completes a milestone, the user can run `/test-dev` with those items
- test-dev uses its own specialized agents (test-explorer, test-implementer, test-reviewer, test-fixer) which are battle-tested for thorough test implementation

The plugins are kept separate — test-dev works standalone and forge-dev suggests it as a follow-up step.

## Key Design Principles

- **Vertical slices**: Every milestone delivers a complete feature (backend + frontend + tests)
- **One milestone at a time**: Sequential execution, never skip ahead
- **Ticket-driven**: No ticket, no implementation. The ticket is the source of truth.
- **Verification gates**: User must confirm the feature works visually before proceeding
- **Token efficiency**: Roadmap is lightweight (read every session). Ticket details only loaded for the current milestone. Specs only read during ticket generation.
- **Strict delegation**: Orchestrator never writes code (except trivial fixes)
- **Convention matching**: Copy existing patterns exactly
- **Honest about failures**: Document what couldn't be done
