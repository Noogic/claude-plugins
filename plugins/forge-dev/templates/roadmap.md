# Roadmap Template

> Guide for structuring project roadmaps that work with the `/forge-dev` plugin.
> Each milestone = one `/forge-dev` invocation = one visible, usable increment.

## Principles

1. **Vertical slices only.** Every milestone delivers a complete feature from database to UI. No "all migrations first" or "all backend first" — each milestone must produce something you can see and use.

2. **Incremental value.** Each milestone adds visible value on top of the previous one. You should be able to open the browser after any milestone and see a working (if incomplete) application. Milestone 1 might just show a workspace selector. Milestone 2 adds projects within that workspace. Milestone 3 adds task lists within projects.

3. **Strict ordering.** Milestones are numbered and executed in order. Milestone N must be fully verified before milestone N+1 starts. No skipping ahead, no parallelizing milestones.

4. **Verification gates.** Every milestone ends with a concrete description of what you should see in the browser or terminal. Not "tests pass" — describe the actual user experience: "After login, user sees their workspace name in the sidebar. Clicking Settings opens the workspace settings page."

5. **Self-contained scope.** Each milestone includes everything needed for its feature to work: migrations, models, factories, controllers, routes, pages, components, and tests. If a feature needs a new model AND a page that displays it, both go in the same milestone.

6. **Small milestones.** Each milestone should be completable in a single `/forge-dev` session. If a feature is too large, split it into milestones that each deliver a usable subset. For example, don't have one milestone "Task System" — instead: "Task list page (read-only)", then "Create/edit tasks", then "Task detail with status changes".

## Structure

```markdown
# Project Roadmap

## Milestone 1: [Short name — what the user gets]

> After this milestone: [What the user sees when they open the app. Be specific.]

### Backend
- [ ] Migration: `table_name` (columns...)
- [ ] Model: `ModelName` with relationships
- [ ] Factory: `ModelNameFactory`
- [ ] Action: `CreateModelName`
- [ ] Controller: `ModelNameController@index`, `@store`
- [ ] Form Request: `StoreModelNameRequest`
- [ ] Routes: `GET /models`, `POST /models`
- [ ] Policy: `ModelNamePolicy`

### Frontend
- [ ] Page: `resources/js/pages/Models/Index.vue`
- [ ] Component: `ModelCard.vue` (if needed)

### Tests
- [ ] Feature test: CRUD endpoints (auth, validation, happy path)
- [ ] Unit test: model relationships
- [ ] Unit test: action logic

### Verify
- [ ] `php artisan test` passes
- [ ] `npm run build` succeeds
- [ ] [Describe what you see in the browser — be specific and visual]

---

## Milestone 2: [Next increment]

> After this milestone: [Cumulative description — what the app looks like now with M1 + M2]

...
```

## Guidelines

### Splitting large features

If a feature touches many models, pages, or concerns, split it into milestones where each one delivers a usable piece:

**Bad** (horizontal):
- Milestone 1: All migrations and models
- Milestone 2: All controllers and routes
- Milestone 3: All pages and components

**Good** (vertical):
- Milestone 1: Workspaces — user can see and switch workspaces
- Milestone 2: Projects — user can create and list projects within a workspace
- Milestone 3: Tasks — user can create and list tasks within a project

### Foundation milestones

The first 1-2 milestones may be infrastructure (scaffold, auth, theme) with no feature UI. That's fine — but they should still have clear verification:

```markdown
## Milestone 0: Scaffold

> After this milestone: App loads at project.test with dark theme. Login/register works.

### Setup
- [ ] Scaffold Laravel 12 + Breeze + Inertia v2 + Vue 3
- [ ] Install shadcn-vue, Tailwind v4, Wayfinder
- [ ] Configure dark theme
- [ ] Configure Pest test infrastructure

### Verify
- [ ] App loads at project.test with dark-themed login page
- [ ] Can register, login, and see the dashboard
- [ ] `php artisan test` passes
- [ ] `npm run build` succeeds
```

### Ordering dependencies

If milestone N creates a model that milestone N+1 uses, that dependency is implicit in the ordering. No need for explicit "depends on" markers — the strict ordering handles it.

### What NOT to include

- No session/batch grouping — the milestones ARE the sessions
- No phase labels — the ordering speaks for itself
- No parallelism hints — milestones are sequential by design
- No layer labels (backend/frontend) at the milestone level — each milestone contains both
