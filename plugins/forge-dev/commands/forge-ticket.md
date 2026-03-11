---
description: Generate a detailed ticket file for a roadmap milestone
---

# Ticket Generation

You are generating a detailed implementation ticket for a specific roadmap milestone.

## Input

The user has requested: "$ARGUMENTS"

---

## Step 1: Identify the Milestone

1. Read `docs/ROADMAP.md` to find the requested milestone.
2. If the request is ambiguous (e.g., "workspaces" could match multiple milestones), list the options and ask the user to pick one.
3. Confirm the milestone with the user: number, title, and items.

## Step 2: Check for Existing Ticket

1. Check if a ticket already exists in `docs/tickets/` for this milestone.
2. If it exists, ask the user: "A ticket already exists for this milestone. Do you want to regenerate it (overwrites), update it, or skip?"

## Step 3: Gather Context

Read the following documents to build context for the ticket writer:

- `docs/specs.md` — domain model, navigation, views, design decisions
- `docs/conventions.md` — coding patterns and conventions (if it exists)
- `docs/tech-stack.md` — architecture decisions (if it exists)
- Any existing tickets in `docs/tickets/` — for format reference

Also check the codebase for what already exists:
- Existing models, migrations, routes that this milestone depends on or builds upon
- This helps the ticket writer set accurate "Depends on" and "Pattern References"

## Step 4: Generate the Ticket

Launch a **ticket-writer agent** with:
- The milestone number, title, and all its items from ROADMAP.md
- The full content of specs.md (or the relevant sections)
- The conventions document (if it exists)
- A summary of what already exists in the codebase
- Any existing ticket files for format reference

The agent will write the ticket to `docs/tickets/m{NN}-{slug}.md`.

## Step 5: Review Gaps and Decisions

After the agent completes:
1. Read the generated ticket file
2. **Check for a `## Decisions` section** in the ticket. If present:
   - Present each question to the user with the agent's recommended options
   - Wait for the user to choose or confirm the recommendations
   - Update the ticket based on the user's answers (edit the Decisions section to record final choices, and update Behavior/Data Model/Interface Contract sections accordingly)
   - Remove the `## Decisions` section header once all questions are resolved (fold the final decisions into the relevant sections)
3. Present a summary to the user:
   - Milestone covered
   - Goal (one-liner)
   - Number of behaviors (Given/When/Then scenarios)
   - Number of test coverage items (unit + feature + browser)
   - Key deliverables
   - Constraints (what it won't do)
   - Verification checks
   - Assumptions made (gaps filled with defaults)
4. Ask: "Does this look right? Any changes needed before we proceed?"
5. If the user wants changes, edit the ticket directly (small changes) or re-run the agent (major changes).

---

## Notes

- The ticket is the source of truth for implementation. Getting it right here saves debugging later.
- Tickets should be detailed enough that an implementing agent can work without re-reading specs.md.
- The Test Coverage section will be used by the `/test-dev` plugin for comprehensive testing.
- Keep constraints tight — every milestone should be focused and self-contained.
