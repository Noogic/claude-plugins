# forge-dev Plugin - Architecture Diagram

## Document Pipeline

```
┌──────────────┐    ┌──────────────┐    ┌──────────────┐    ┌──────────────┐
│  specs.md    │    │ ROADMAP.md   │    │  tickets/    │    │ conventions  │
│              │    │              │    │  m01-*.md    │    │ .md          │
│ Vision,      │───▶│ Milestones,  │───▶│              │    │              │
│ domain model,│    │ ordering,    │    │ Behaviors,   │    │ Patterns,    │
│ navigation,  │    │ checkboxes   │    │ data model,  │    │ coding style │
│ views        │    │              │    │ contracts,   │    │              │
│              │    │ (lightweight │    │ test coverage │    │ (cached,     │
│ (read rarely)│    │  read often) │    │              │    │  generated   │
│              │    │              │    │ (1 per       │    │  once)       │
│              │    │              │    │  milestone)  │    │              │
└──────────────┘    └──────────────┘    └──────────────┘    └──────────────┘
        │                  │                   │                   │
        │   /forge-ticket  │                   │   /forge-dev      │
        └────────┬─────────┘                   └────────┬──────────┘
                 ▼                                      ▼
          Ticket generation                     Implementation
```

## /forge-ticket Flow

```
    ┌──────────────────────────────────┐
    │  /forge-ticket [milestone]       │
    │  Reads specs + roadmap + code    │
    └──────────────┬───────────────────┘
                   │
    ┌──────────────▼───────────────────┐
    │  ticket-writer agent             │
    │  🔵 sonnet                       │
    │                                  │
    │  Generates: behaviors, data      │
    │  model, interface contracts,     │
    │  scope, constraints, test        │
    │  coverage, verification checks   │
    └──────────────┬───────────────────┘
                   │
    ┌──────────────▼───────────────────┐
    │  User reviews and approves      │
    │  Saved to docs/tickets/m{NN}.md │
    └──────────────────────────────────┘
```

## /forge-dev Flow

```
┌─────────────────────────────────────────────────────────────────────────┐
│                       forge-dev Plugin (v1.0.0)                        │
│            One milestone at a time. Vertical slices only.              │
└─────────────────────────────────────────────────────────────────────────┘

                      ┌──────────────────────┐
                      │  /forge-dev command   │
                      │                      │
                      │  User requests a     │
                      │  milestone (or       │
                      │  "next")             │
                      └──────────┬───────────┘
                                 │
          ═══════════════════════╪═══════════════════════════
          ║      ORCHESTRATOR (the main Claude session)     ║
          ║      Delegates all work — never implements      ║
          ═══════════════════════╪═══════════════════════════
                                 │
    ┌────────────────────────────┼────────────────────────────┐
    │                 Phase 1: Identify Milestone             │
    │                                                        │
    │  • Read ROADMAP.md → find next unchecked milestone     │
    │  • Verify all prior milestones are complete            │
    │  • Read ticket file from docs/tickets/                 │
    │    (no ticket = no implementation → stop)              │
    │  • Confirm scope + verification gate with user         │
    └────────────────────────────┼────────────────────────────┘
                                 │
    ┌────────────────────────────┼────────────────────────────┐
    │     Phase 2: Conventions & Codebase Exploration        │
    │                                                        │
    │  Step 1: Load Conventions (cached — once per project)  │
    │  ┌──────────────────────────────────────────────┐      │
    │  │  docs/conventions.md exists?                 │      │
    │  │  ├── YES → Read it. Done.                    │      │
    │  │  └── NO → Has existing code?                 │      │
    │  │      ├── YES → Launch explorer → save output │      │
    │  │      └── NO  → Copy templates/conventions.md │      │
    │  └──────────────────────────────────────────────┘      │
    │                                                        │
    │  Step 2: Feature-Specific Exploration (always runs)    │
    │  Launches 2 feature-explorer agents IN PARALLEL        │
    │                                                        │
    │  ┌──────────────────────┐ ┌──────────────────────┐     │
    │  │  Explorer 1           │ │  Explorer 2           │    │
    │  │ 🟡 sonnet             │ │ 🟡 sonnet             │    │
    │  │                      │ │                      │     │
    │  │ Source Context       │ │ Dependencies         │     │
    │  │ Controllers, Actions,│ │ Existing models,     │     │
    │  │ Models, Pages,       │ │ migrations, factories│     │
    │  │ Components, Routes   │ │ routes, shared code  │     │
    │  └──────────────────────┘ └──────────────────────┘     │
    │                                                        │
    │  Combines: conventions + ticket + exploration          │
    └────────────────────────────┼────────────────────────────┘
                                 │
    ┌────────────────────────────┼────────────────────────────┐
    │     Phase 3: Implementation (vertical slice)           │
    │                                                        │
    │  Ticket = source of truth (Behaviors, Data Model,      │
    │  Interface Contracts)                                   │
    │                                                        │
    │  ┌──────────────────┐                                  │
    │  │ 1. Backend Impl. │                                  │
    │  │ 🟢 sonnet        │                                  │
    │  │                  │                                  │
    │  │ Migrations,      │                                  │
    │  │ Models, Enums,   │                                  │
    │  │ Factories,       │                                  │
    │  │ Actions,         │                                  │
    │  │ Controllers,     │                                  │
    │  │ Form Requests,   │                                  │
    │  │ Policies, Tests  │                                  │
    │  └────────┬─────────┘                                  │
    │           │ completes, then                             │
    │  ┌────────▼─────────┐                                  │
    │  │ 2. Frontend Impl.│                                  │
    │  │ 🟢 sonnet        │                                  │
    │  │                  │                                  │
    │  │ Vue pages,       │                                  │
    │  │ Components,      │                                  │
    │  │ Composables,     │                                  │
    │  │ TypeScript types │                                  │
    │  │                  │                                  │
    │  │ Receives actual  │                                  │
    │  │ backend files    │                                  │
    │  │ from step 1      │                                  │
    │  └──────────────────┘                                  │
    └────────────────────────────┼────────────────────────────┘
                                 │
    ┌────────────────────────────┼────────────────────────────┐
    │           Phase 4: Test & Verify                       │
    │                                                        │
    │  Step 1: Automated tests                               │
    │  $ php artisan test                                    │
    │  $ npm run build                                       │
    │  • Fixes trivial issues (imports, typos)               │
    │  • 3 retries max per failing test                      │
    │                                                        │
    │  Step 2: Verification Gate (MANDATORY)                 │
    │  ┌──────────────────────────────────────────────┐      │
    │  │ Read ticket's Verify section                 │      │
    │  │ Run automated checks (full suite, build)     │      │
    │  │ Ask user to confirm visual/browser checks    │      │
    │  │                                              │      │
    │  │ ⛔ DO NOT PROCEED until user confirms        │      │
    │  │ If issues found → fix and re-verify          │      │
    │  └──────────────────────────────────────────────┘      │
    └────────────────────────────┼────────────────────────────┘
                                 │
    ┌────────────────────────────┼────────────────────────────┐
    │              Phase 5: Quality Review                   │
    │                                                        │
    │  Launches feature-reviewer agents IN PARALLEL          │
    │  (1 per created/modified file)                         │
    │                                                        │
    │  ┌─────────────┐ ┌─────────────┐ ┌─────────────┐      │
    │  │ Reviewer A   │ │ Reviewer B   │ │ Reviewer N   │    │
    │  │ 🔴 sonnet    │ │ 🔴 sonnet    │ │ 🔴 sonnet    │    │
    │  │             │ │             │ │             │      │
    │  │ Reviews 1   │ │ Reviews 1   │ │ Reviews 1   │      │
    │  │ file against│ │ file against│ │ file against│      │
    │  │ the ticket  │ │ the ticket  │ │ the ticket  │      │
    │  └─────────────┘ └─────────────┘ └─────────────┘      │
    │                                                        │
    │  Confidence scoring 0-100, only ≥75 reported           │
    │  If no findings ≥75 → skip to Phase 7                  │
    └────────────────────────────┼────────────────────────────┘
                                 │
    ┌────────────────────────────┼────────────────────────────┐
    │              Phase 6: Fix Findings                     │
    │                                                        │
    │  ┌──────────────────────────────────────────┐          │
    │  │  feature-fixer  🔵 sonnet                │          │
    │  │                                          │          │
    │  │  • Receives all findings (conf ≥75)      │          │
    │  │  • Fixes Critical first, then Important  │          │
    │  │  • Runs tests after each file            │          │
    │  │  • Never breaks working code             │          │
    │  └──────────────────────────────────────────┘          │
    └────────────────────────────┼────────────────────────────┘
                                 │
    ┌────────────────────────────┼────────────────────────────┐
    │              Phase 7: Finalize                         │
    │                                                        │
    │  • Final run: php artisan test + npm run build          │
    │  • Update ROADMAP.md checkboxes                        │
    │  • Update ticket status → done                         │
    │  • Summary to user                                     │
    │  • Suggest: /test-dev with ticket's Test Coverage      │
    │  • Show next milestone                                 │
    └────────────────────────────────────────────────────────┘

                                 │
                                 ▼

    ┌────────────────────────────────────────────────────────┐
    │              test-dev Plugin (separate)                │
    │                                                        │
    │  User runs: /test-dev [Test Coverage from ticket]      │
    │                                                        │
    │  Uses its own battle-tested agents:                    │
    │  test-explorer → test-implementer → test-reviewer     │
    │  → test-fixer                                         │
    │                                                        │
    │  Comprehensive test coverage beyond what forge-dev     │
    │  writes during implementation                          │
    └────────────────────────────────────────────────────────┘
```
