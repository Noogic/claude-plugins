# Project Conventions

> This file was generated as a starting point. Update it as your project's patterns evolve.
> Delete this file to force re-extraction from your codebase on the next `/forge-dev` run.

## Stack

- **Backend**: Laravel 12, PHP 8.5+
- **Frontend**: Vue 3, Inertia v2, TypeScript
- **UI**: shadcn-vue (new-york-v4), Tailwind CSS v4
- **Database**: MySQL, auto-increment IDs
- **Testing**: Pest
- **Real-time**: Laravel Reverb, Echo
- **Routes**: Wayfinder (typed route helpers)

## Modules

A module is a domain area named after its primary model (Projects, Tasks, Workspaces, etc.). No framework or package — just subfolders within standard Laravel directories.

**Grouped by module subfolder** (multiple files per model):

```
app/Actions/Projects/
app/Http/Controllers/Projects/
app/Http/Requests/Projects/
app/Http/Resources/Projects/
app/Events/Projects/
app/Exceptions/Projects/
app/Jobs/Projects/
```

**Stay flat** (one file per model):

```
app/Models/Project.php
app/Policies/ProjectPolicy.php
app/Enums/ProjectStatus.php
database/factories/ProjectFactory.php
database/migrations/
```

### Cross-module actions

When an action touches multiple models, it belongs to the module of the **primary model being mutated**. If both models are modified, it belongs to the module that owns the relationship (the one with the foreign key or the one initiating the association).

For truly cross-cutting logic with no clear owner (e.g., `GenerateMonthlyBillingReport` touching Invoices, Appointments, and Patients), create a domain-level module that doesn't map to a single model (e.g., `Billing/`). These are rare and decided case-by-case.

## Routes

Web routes in `routes/web.php`, API routes in `routes/api.php`. Always use explicit, CRUD-named routes with plural resource names. Group by module with prefix and middleware.

```php
// routes/web.php

Route::middleware(['auth'])->group(function () {

    // Projects
    Route::prefix('projects')->name('projects.')->group(function () {
        Route::get('/', [ProjectController::class, 'index'])->name('index');
        Route::get('/create', [ProjectController::class, 'create'])->name('create');
        Route::post('/', [ProjectController::class, 'store'])->name('store');
        Route::get('/{project}', [ProjectController::class, 'show'])->name('show');
        Route::get('/{project}/edit', [ProjectController::class, 'edit'])->name('edit');
        Route::put('/{project}', [ProjectController::class, 'update'])->name('update');
        Route::delete('/{project}', [ProjectController::class, 'destroy'])->name('destroy');
    });

});
```

Route names follow the pattern `{plural-model}.{action}`: `projects.index`, `projects.store`, `projects.show`, `projects.update`, `projects.destroy`.

For nested resources, chain the prefix: `projects.tasks.index` → `GET /projects/{project}/tasks`.

## Backend Patterns

### Controllers

Thin controllers — no business logic. Delegate to Actions. Return Inertia responses.

```php
class ProjectController extends Controller
{
    public function store(StoreProjectRequest $request, CreateProject $action)
    {
        $project = $action($request->validated());

        return redirect()->route('projects.show', $project);
    }

    public function index()
    {
        return Inertia::render('Projects/Index', [
            'projects' => Project::with('workspace')->get(),
        ]);
    }
}
```

### Actions

Single-responsibility classes in `app/Actions/`. One public method: `__invoke()`. Handle business logic, NOT validation.

```php
class CreateProject
{
    public function __invoke(array $data): Project
    {
        return Project::create($data);
    }
}
```

### Models

- Place in `app/Models/`
- Define `$fillable` (prefer over `$guarded`)
- Define `$casts` for enums, JSON, booleans, dates
- Type-hint relationship methods

```php
class Project extends Model
{
    protected $fillable = ['name', 'description', 'workspace_id', 'status'];

    protected $casts = [
        'status' => ProjectStatus::class,
    ];

    public function workspace(): BelongsTo
    {
        return $this->belongsTo(Workspace::class);
    }

    public function tasks(): HasMany
    {
        return $this->hasMany(Task::class);
    }
}
```

### Migrations

- `$table->id()` for primary keys (auto-increment, NOT UUIDs)
- `$table->foreignId('xxx_id')->constrained()->cascadeOnDelete()` for foreign keys (evaluate cascade behavior per-relationship)
- `$table->timestamps()` on every table
- Plural table names, snake_case columns

```php
Schema::create('projects', function (Blueprint $table) {
    $table->id();
    $table->foreignId('workspace_id')->constrained()->cascadeOnDelete();
    $table->string('name');
    $table->text('description')->nullable();
    $table->string('status')->default('active');
    $table->timestamps();
});
```

### Enums

String-backed enums in `app/Enums/`.

```php
enum ProjectStatus: string
{
    case Active = 'active';
    case Archived = 'archived';
}
```

### Form Requests

Place in `app/Http/Requests/`. Define `authorize()` (check policies/permissions) and `rules()`.

```php
class StoreProjectRequest extends FormRequest
{
    public function authorize(): bool
    {
        return $this->user()->can('create', Project::class);
    }

    public function rules(): array
    {
        return [
            'name' => ['required', 'string', 'max:255'],
            'description' => ['nullable', 'string', 'max:1000'],
        ];
    }
}
```

### API Resources

Per-endpoint resources in `app/Http/Resources/`, grouped by module. Never create a single generic resource per model — each endpoint gets its own resource that exposes only the fields it needs.

Name as `{Context}{Action}Resource`, never `{Model}Resource`.

```
app/Http/Resources/
  Projects/
    ProjectListResource.php      ← GET /api/projects
    ProjectDetailResource.php    ← GET /api/projects/{id}
```

```php
class ProjectListResource extends JsonResource
{
    public function toArray(Request $request): array
    {
        return [
            'id' => $this->id,
            'name' => $this->name,
            'status' => $this->status,
        ];
    }
}
```

```php
class ProjectDetailResource extends JsonResource
{
    public function toArray(Request $request): array
    {
        return [
            'id' => $this->id,
            'name' => $this->name,
            'description' => $this->description,
            'status' => $this->status,
            'workspace' => [
                'id' => $this->workspace->id,
                'name' => $this->workspace->name,
            ],
            'created_at' => $this->created_at,
        ];
    }
}
```

### Events / Broadcasting

Events in `app/Events/`, grouped by module. Named in past tense (`AppointmentUpdated`, `InvoiceCreated`). Listeners in `app/Listeners/`, grouped by module.

By default, events are synchronous and backend-only. Add interfaces as needed:

- `ShouldQueue` — when the event processing should be async
- `ShouldBroadcast` — when the event should reach the frontend via Reverb/Echo (requires `broadcastOn()`)

An event can implement both.

```php
class AppointmentUpdated implements ShouldBroadcast
{
    use Dispatchable, InteractsWithSockets, SerializesModels;

    public function __construct(public Appointment $appointment) {}

    public function broadcastOn(): array
    {
        return [
            new PrivateChannel('appointments.'.$this->appointment->id),
        ];
    }
}
```

Channel naming follows plural module names: `users.{id}`, `projects.{id}`, `appointments.{id}`.

Channel authorization in `routes/channels.php`:

```php
Broadcast::channel('appointments.{appointment}', function (User $user, Appointment $appointment) {
    return $user->can('view', $appointment);
});
```

Frontend listener pattern with Echo:

```vue
<script setup lang="ts">
import { onMounted, onUnmounted } from 'vue'

onMounted(() => {
    window.Echo.private(`appointments.${props.appointment.id}`)
        .listen('AppointmentUpdated', (e) => {
            // handle event
        })
})

onUnmounted(() => {
    window.Echo.leave(`appointments.${props.appointment.id}`)
})
</script>
```

### Exceptions

Custom exceptions in `app/Exceptions/`, grouped by module. Use `abort()` for simple HTTP errors (404, 403). Use custom exceptions for domain-specific errors that carry meaning or need a specific response.

```php
// app/Exceptions/Projects/ProjectLimitExceededException.php

class ProjectLimitExceededException extends Exception
{
    public function __construct(public int $limit)
    {
        parent::__construct("Project limit of {$limit} reached.");
    }

    public function render(Request $request): Response|JsonResponse
    {
        if ($request->expectsJson()) {
            return response()->json([
                'error' => 'project_limit_exceeded',
                'message' => $this->getMessage(),
                'limit' => $this->limit,
            ], 422);
        }

        return back()->withErrors(['limit' => $this->getMessage()]);
    }
}
```

API error responses follow a consistent shape:

```json
{
    "error": "snake_case_error_code",
    "message": "Human-readable description.",
}
```

Laravel's default exception rendering handles standard errors (validation, auth, 404) automatically. Only define `render()` on custom exceptions that need specific behavior.

### Policies

Place in `app/Policies/`. Methods: `viewAny`, `view`, `create`, `update`, `delete`.

### Factories

Place in `database/factories/`. Use `fake()` helper. Reference related factories.

```php
class ProjectFactory extends Factory
{
    protected $model = Project::class;

    public function definition(): array
    {
        return [
            'name' => fake()->words(3, true),
            'description' => fake()->sentence(),
            'workspace_id' => Workspace::factory(),
            'status' => ProjectStatus::Active,
        ];
    }

    public function archived(): static
    {
        return $this->state(['status' => ProjectStatus::Archived]);
    }
}
```

### Jobs

Jobs in `app/Jobs/`, grouped by module. Jobs are thin async wrappers — they contain **no business logic**. They delegate to Actions. This keeps logic testable synchronously.

```php
// app/Jobs/Projects/SyncProjectDataJob.php

class SyncProjectDataJob implements ShouldQueue
{
    use Dispatchable, InteractsWithQueue, Queueable, SerializesModels;

    public function __construct(public Project $project) {}

    public function handle(SyncProjectData $action): void
    {
        $action($this->project);
    }
}
```

### Commands

Commands in `app/Console/Commands/`, grouped by module. Delegate to Actions for anything non-trivial. Simple one-off commands (cleanup, seeding) can be self-contained.

```php
// app/Console/Commands/Projects/PurgeArchivedProjectsCommand.php

class PurgeArchivedProjectsCommand extends Command
{
    protected $signature = 'projects:purge-archived';
    protected $description = 'Purge archived projects older than 90 days';

    public function handle(PurgeArchivedProjects $action): int
    {
        $count = $action();

        $this->info("Purged {$count} archived projects.");

        return self::SUCCESS;
    }
}
```

## Frontend Patterns

### Vue Pages

- Place in `resources/js/pages/`
- `<script setup lang="ts">` with composition API
- Register layout via `defineOptions({ layout: LayoutComponent })`
- Use `useForm()` from `@inertiajs/vue3` for forms
- Define prop types inline per page — matching exactly what the controller sends, nothing more

```vue
<script setup lang="ts">
import GlobalLayout from '@/layouts/GlobalLayout.vue'

defineOptions({ layout: GlobalLayout })

interface ProjectListItem {
    id: number
    name: string
    status: string
}

const props = defineProps<{
    projects: ProjectListItem[]
}>()
</script>

<template>
    <div>...</div>
</template>
```

Shared types (enums, reusable value objects) go in `resources/js/types/`. Page-specific types stay in the page file.

### Vue Components

- Place in `resources/js/components/`
- `<script setup lang="ts">`
- Define props with `defineProps<{...}>()`
- Import shadcn-vue from `@/components/ui/`

### Forms

Always use `useForm()` from Inertia:

```vue
<script setup lang="ts">
import { useForm } from '@inertiajs/vue3'

const form = useForm({
    name: '',
    description: '',
})

function submit() {
    form.post('/projects')
}
</script>
```

### shadcn-vue

Import from `@/components/ui/button`, `@/components/ui/input`, etc. Use component variants as designed. Use `cn()` for conditional classes.

### Styling

- Tailwind CSS v4 classes
- Dark theme: dark backgrounds, light text, avoid purple
- Monospace for metadata: `font-mono text-xs text-muted-foreground`
- Status badges as small colored pills
- Minimal borders, no extra shadows
- Tight spacing, high information density

### State Management

No Vuex/Pinia. Use Inertia page props + provide/inject from layouts.

## Testing Patterns

### Pest

- `tests/Feature/` for HTTP/integration tests
- `tests/Unit/` for unit tests
- `uses(RefreshDatabase::class)` for database tests
- **Always freeze time** before using `now()`, `Carbon::now()`, etc.

```php
it('creates a project', function () {
    $user = User::factory()->create();
    $workspace = Workspace::factory()->create();
    $workspace->members()->attach($user, ['role' => 'owner']);

    $response = $this->actingAs($user)
        ->post('/projects', [
            'name' => 'Test Project',
            'description' => 'A test project',
        ]);

    $response->assertRedirect();
    $this->assertDatabaseHas('projects', [
        'name' => 'Test Project',
        'workspace_id' => $workspace->id,
    ]);
});

it('requires a name', function () {
    $user = User::factory()->create();

    $response = $this->actingAs($user)
        ->post('/projects', ['name' => '']);

    $response->assertSessionHasErrors(['name']);
});

it('requires authentication', function () {
    $response = $this->get('/projects');
    $response->assertRedirect('/login');
});

it('belongs to a workspace', function () {
    $project = Project::factory()->create();

    expect($project->workspace)->toBeInstanceOf(Workspace::class);
});
```
