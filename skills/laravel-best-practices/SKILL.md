---
name: laravel-best-practices
description: >
  Global Laravel best practices: architecture (BaseController/BaseRepo), exception handling/security, migration keys (bigIncrements), and route parameter constraints.
  Trigger: When creating, refactoring, or updating Laravel controllers, repositories, migrations, routes, or handling exceptions.
license: Apache-2.0
metadata:
  author: gentleman-programming
  version: "1.0"
---

# Laravel Global Best Practices

This skill consolidates global rules and patterns for Laravel development.

## 1. Architecture: BaseController & BaseRepo

Enforce the usage of BaseController and BaseRepo architecture for any CRUD operations.

- **Models**: Place them in the root `Cms/` folder with the `Cms\...` namespace (NOT under `app/Cms`).
- **Repositories**: Create a Repository that extends `Cms\Base\BaseRepo` and implements `getModel()`.
- **Controllers**: Controllers must extend `Cms\Base\BaseController`. Pass the repository and a validation array to the parent constructor. Let `BaseController` handle the CRUD automatically (don't write manual `index`, `store`, etc. unless overriding).
- **Never type-hint `$id` as `int` when overriding `BaseRepoInterface::update()`/`destroy()`.** `BaseController::update()`/`destroy()`/`show()` only run `is_numeric($id)` — they never cast it, so `$id` reaches the repo as a string from the route segment. If the repo file has `declare(strict_types=1)` (the project default) and the override declares `int $id`, PHP throws a `TypeError` on that string. `TypeError` extends `Error`, not `Exception`, so the repo's own `catch (\Exception $e)` never catches it — uncaught 500 on every real request to that route. `array $data`/`array $request` are safe to type (they always arrive as real arrays via `$request->all()`); only `$id` is off-limits. Hit in production via PR 2284 (`GeneroRepo`), 2026-08-26.
- **Generalization of the rule above — this is not only about `$id`.** Any parameter that `Cms\Base\BaseRepo` declares **without** a type hint (its base `all($column = 'id', $filter = 'ASC')`, `update($request, $id)`, `destroy($id)`, etc. are all untyped) cannot be given a type hint in a subclass override — PHP enforces parameter contravariance on class overrides, not just interfaces: a child method's parameter type must be the same as or wider than the parent's, and "no type" is the widest possible type, so adding *any* scalar/class type hint on an override narrows it and violates LSP. This is a **fatal `Error` at class-load time** ("Declaration of X::method() must be compatible with BaseRepo::method()"), not a `TypeError` at call time, and **not a syntax error** — `php -l` reports "No syntax errors detected" on a file that still fatals the instant Laravel autoloads/instantiates the class, because `php -l` only parses, it doesn't load the class hierarchy. Confirmed by reproducing directly inside the backend container: typing `Formato2276ExgoneaDianRepo::all(string $column = 'start_date', string $filter = 'DESC'): Collection` (overriding `BaseRepo::all($column = 'id', $filter = 'ASC')`, itself untyped) throws exactly this fatal. `php -l` had already passed clean on that file; the bug only surfaced because the user manually caught and reverted it. PR 2120, 2026-08-28. **To actually verify an override of a `BaseRepo`/`BaseController` method is safe, `php -l` is not enough — load the class** (e.g. `php -r "require 'vendor/autoload.php'; require_once 'bootstrap/app.php'; require '<file>'; new <Namespace>\<Class>();"` inside the backend container, or hit the real route) before trusting new parameter type hints on such an override. A **return type** is fine to add even when the parent declares none — it's specifically parameter type hints narrowing an untyped parent parameter that fatal.
- **`BaseController::validateRequest()` is `private`, not `protected`.** A subclass overriding `store()`/`update()` cannot call `$this->validateRequest($request)` — it's inherited-invisible from a private parent method and fatals with "Call to private method Cms\Base\BaseController::validateRequest() from scope ...". Both `$this->model` and `$this->validation` ARE `protected` on `BaseController` and safe to use directly, so the fix is to inline the same check in the override: `$validator = Validator::make($request->all(), $this->validation); if ($validator->fails()) { throw new \InvalidArgumentException('Error de validación: ' . $validator->errors()->first()); }`. Hit in an unreviewed PR via `Formato2276ExgoneaDianController::store()` (PR 2120, 2026-08-28) — the endpoint would have fataled on every real request; PRISM's review missed it entirely since it's a runtime-only failure, not a static pattern.
- **`BaseRepo::destroy($id)` unconditionally toggles a `status` column** (`$dato->status = $dato->status ? 0 : 1;`) — it assumes every model using the default `Route::resource(...)` DELETE action has one. If a table has no `status` column, the inherited `destroy()` throws a SQL error on `DELETE {resource}/{id}` the first time it's called. Before leaving `Route::resource()` to expose the inherited `destroy()` unmodified, check the model's migration for a `status` column; if there isn't one, override `destroy($id)` in the Repo to do a real `$this->getModel()->findOrFail($id)->delete()` instead. Hit via PR 2120 (`Formato2276ExgoneaDianRepo`, table `formato2276_exgonea_dian` has no `status` column), 2026-08-28.
- **Never narrow `Request $request` to a specific `FormRequest` subtype when overriding `BaseController::index()/store()/update()/destroy()/show()/history()/Softdelete()`.** These are concrete (non-abstract) methods on `BaseController`, and PHP enforces parameter contravariance on class overrides, not just interfaces — narrowing `Request` to e.g. `VacancyRequest` throws "Declaration should be compatible with BaseController->store(request: Request)". Keep the parameter as `Request $request` and resolve the specific FormRequest manually inside the method body instead: `$formRequest = app(VacancyRequest::class);` — in this project's pinned Laravel v5.4.36, `FormRequestServiceProvider::boot()` registers `$this->app->afterResolving(ValidatesWhenResolved::class, fn($r) => $r->validate())`, so that plain `app()` call already triggers `authorize()` + full rule validation as a side effect of container resolution, no route-level type-hint required. Then use `$formRequest->only(array_keys($formRequest->rules()))` for a mass-assignment-safe payload. This constraint does NOT apply to a controller's own custom-named methods that don't override anything on `BaseController` (e.g. `ApplicationPublicController::storeApplication()`, or any controller extending Laravel's base `Controller` directly instead of `BaseController`) — those can type-hint the FormRequest directly in the signature with no LSP conflict. Hit via PR 2289 (`VacancyController`), 2026-08-26.

**Example**:
```php
namespace App\Http\Controllers\Disciplinary;

use Cms\Base\BaseController;
use Cms\Rh\InternalRuleArticle\InternalRuleArticleRepo;

class InternalRuleArticleController extends BaseController
{
    public function __construct(InternalRuleArticleRepo $repo)
    {
        $validation = [
            'article_number' => 'required|string',
            'title' => 'required|string',
        ];
        
        parent::__construct($repo, $validation);
    }
}
```

## 1b. `BaseController`'s `$validation` Array Does NOT Gate Mass Assignment

**`BaseController::validateRequest()` returns `$request->all()` unconditionally.** The `$validation` array passed to `parent::__construct($model, $validation)` is used ONLY to run `Validator::make($request->all(), $this->validation)` — it does not filter which keys get passed to `$this->model->update($requestData, $id)`. Removing a sensitive field from `$validation` does NOT stop a client from setting it via `store`/`update` — undeclared keys pass through `$request->all()` untouched.

- **The actual mass-assignment gate is the model's `$fillable` array.** If a field is listed in the Eloquent model's `$fillable`, it CAN be set by any client request through a `BaseController`-based endpoint, regardless of whether it's declared in `$validation` or not.
- Before treating "a sensitive field is present in a controller's `$validation` array" as the vulnerability, check the model's `$fillable` first — that's where to actually remove it. Removing it only from `$validation` fixes nothing.
- Hit via PR 2279 (`EmpleadoPerfilController` / `Perfil::$fillable` had `activo`, allowing clients to toggle active status through the generic profile-update endpoint). Worth auditing other `BaseController` subclasses for the same gap — not yet done project-wide.

## 2. Exception Handling & Security

Never expose exception details (like SQL syntax errors or stack traces) to clients. Use generic user-facing messages and log details server-side.

- **Rule 1**: Never Concatenate Exception Messages.
- **Rule 2**: Use Named Exceptions with Specific Status Codes (e.g., `ModelNotFoundException` instead of generic `\Exception`).
- **Rule 3**: Separate User Message from Debug Details (log the real error, throw a generic safe message).
- **Rule 4**: For Catch-and-Rethrow, Use Chaining (`throw new \Exception('Safe msg', 0, $e);`).

**Example**:
```php
try {
    $result = DB::select('SELECT * FROM my_table');
} catch (QueryException $e) {
    \Log::error('Database query failed', [
        'message' => $e->getMessage(),
        'sql' => $e->getSql() ?? 'N/A',
        'bindings' => $e->getBindings() ?? []
    ]);
    throw new \Exception('Database operation failed'); // Generic message for client
}
```

- **Rule 5**: In a controller's generic `catch (\Exception $e)` block, don't reflexively apply "generic message to client" to every exception type that lands there — split `InvalidArgumentException` (or any exception the app itself throws for a domain/business-rule violation, e.g. date-range overlap, validation failure) into its own `catch` clause **before** the generic one, and return `$e->getMessage()` as-is: that message was written by this codebase specifically to be shown to the user, and genericizing it destroys real user-facing feedback for no security benefit. Reserve the log-real/return-generic pattern for the true catch-all (`catch (\Exception $e)` after every specific type), which is what actually risks leaking SQL/internal details. Also don't use `$e->getCode()` to pick the HTTP status for that catch-all — an app-thrown exception's code isn't guaranteed to be a valid HTTP status (could be 0, a DB error code, anything); use a fixed status (422 for the `InvalidArgumentException` branch, 500 for the true catch-all) instead of trusting `getCode()`. Hit via PR 2120 (`Formato2276ExgoneaDianController::store()/update()`), 2026-08-28.

## 3. Migration BigInt Keys

Laravel migrations must declare primary keys with `bigIncrements()` and foreign-key columns with `unsignedBigInteger()` across all new tables and relations.

- **Rule 1**: Primary keys use `bigIncrements('id')`.
- **Rule 2**: Foreign-key columns use `unsignedBigInteger('user_id')`. Use this even if the referenced table currently uses legacy `increments`.
- **Rule 3**: Pivot/junction tables — both columns are `unsignedBigInteger`.
- **Rule 4**: Nullable FKs keep the same type (`unsignedBigInteger('x')->nullable()`).
- **Rule 5**: Short, explicit FK/index names when table+column would exceed 63 chars (PostgreSQL limit).
- **Rule 6**: Eloquent casts mirror the column type (cast FK columns as `'integer'`, NOT `'string'`).

**Example**:
```php
Schema::create('recurring_incentive_configs', function (Blueprint $table) {
    $table->bigIncrements('id');
    $table->unsignedBigInteger('user_id');
    
    $table->foreign('user_id')->references('id')->on('users');
});
```

## 4. Route Parameter Constraints

Restrict route parameters to specific formats (digits, UUIDs, slugs) to prevent invalid inputs, collisions with literal segments, and security issues.

- **Rule 1**: Always Constrain Numeric IDs (e.g., `->where('id', '\d+')`).
- **Rule 2**: Use `Route::pattern()` for Global Constraints (e.g., `Route::pattern('id', '\d+');` in `RouteServiceProvider` or `routes/api.php`).
- **Rule 3**: Order Routes to Avoid Ambiguity (literal segments before parameterized).

**Example**:
```php
Route::pattern('id', '\d+'); // Global

// Or inline:
Route::get('posts/{id}', 'PostController@show')->where('id', '\d+');
```

## 5. Migration Performance and Safety

When writing migrations that manipulate large datasets or perform schema changes, ensure performance and safe rollbacks.

- **Rule 1**: Avoid `get()` on large tables. Use `chunk(200)` to limit memory consumption and collect data into arrays for bulk `insert()`.
- **Rule 2**: Never run single `insert()` statements inside a loop. Accumulate data and perform bulk inserts per chunk.
- **Rule 3**: In `down()` methods, completely reconstruct the full state of previous tables (e.g., using `chunk` and `join` if splitting tables) instead of just grabbing the first record.
- **Rule 4**: Always wrap `dropColumn` with `if (Schema::hasColumn(...))` inside `down()` methods to prevent rollback failures if the column never existed or failed to create.

## 6. FormRequest Authorization

Never leave the `authorize()` method of a `FormRequest` returning `true` unconditionally unless the endpoint is explicitly public.

- **Rule 1**: Validate permissions explicitly. At minimum, use `return auth()->check();` to ensure the user is authenticated.
- **Rule 2**: If role-based access is required, implement checks using Gates, Policies, or explicit permissions (e.g., `return $this->user()->can('update-config');`).
- **Accepted alternative**: `return auth()->check();` alone is fine when the route already applies an Entrust `permission:*` middleware (e.g. `Route::post('vacancies', ...)->middleware('permission:create_vacancies');` in `routes/api.php`) — RBAC is enforced at the route layer instead of inside `authorize()`. Confirmed project convention (`VacancyRequest`, PR 2284); don't flag this combination as a missing-permission gap.
- **Silent enforcement gap: a `FormRequest` only used to source `$validation` in a `BaseController` constructor never runs `authorize()` at all.** A common `BaseController` pattern is `parent::__construct($model, (new SomeRequest())->rules());` — this only calls `->rules()` on a throwaway instance to get the validation array; it never lets Laravel resolve the `FormRequest` as an injected dependency. Laravel only runs `authorize()` (and auto-validation) when a `FormRequest` is *type-hinted directly on a route-bound controller method* (`FormRequestServiceProvider::boot()` hooks container resolution, not manual `new`). If no action method type-hints the `FormRequest`, its `authorize()` — including any `can('some_permission')` check inside it — is dead code that silently never executes, and the endpoint has **no enforced permission gate** beyond whatever route middleware exists (often none). Verify by grepping the controller for the `FormRequest` class name outside the constructor; if it never appears as a method parameter type-hint, `authorize()` isn't running. The fix that actually enforces it: add `->middleware('permission:xyz')` at the route/group level (matches this project's established per-route convention, see `routes/api.php`), since resolving the `FormRequest` manually per read-only action is more invasive. Hit via PR 2120 (`Formato2276ExgoneaDianRequest`/`Formato2276ExgoneaDianController`) — the migration granted a `formato_2276` permission to Administradores, but every authenticated user could hit every endpoint since neither the dead `authorize()` nor the route ever enforced it. 2026-08-28.

## 7. Laravel Version Is v5.4.36 Exactly — Verify Against `vendor/`, Not `composer.json`

`composer.json` pins `"laravel/framework": "5.4.*"`, and `composer.lock` confirms the exact installed version: **v5.4.36**. Several convenience APIs that "everyone knows" from modern Laravel don't exist yet at this version — check `vendor/laravel/framework/src/...` directly before relying on any validation-related return value:

- **`FormRequest::validated()` does not exist in 5.4.x** (added in 5.5). Calling `$request->validated()` on a `FormRequest` is a fatal "Method not found" on every request.
- **`Controller::validate($request, $rules)`** (the `ValidatesRequests` trait, used by every controller) **is `void` in 5.4.x** — it does not return the validated array (that return value is also 5.5+). Assigning its result (`$data = $this->validate(...)`) gives `$data = null`; `$data['user_id'] = ...` then silently auto-vivifies `$data` into a one-key array, discarding every other field with no error at all.
- **`Validator::valid()` is not a substitute either** — it returns every input key that has no validation error, including keys never declared in `rules()` at all. Same mass-assignment hole as `$request->all()`.
- **Correct 5.4.x pattern** to get "only the fields declared in `rules()`, already validated": let validation run for its side effect (throwing on failure) via `$this->validate($request, $rules)` or FormRequest auto-validation, then separately do `$request->only(array_keys($rules))` (or `$formRequest->only(array_keys($formRequest->rules()))` — see §1 for why the FormRequest often has to be resolved manually via `app()` instead of type-hinted).

Hit in production: PR 2284 (`VacancyController`, merged to `develop` with this exact bug — `$data=null` after `$this->validate()`) and PR 2287 (`ApplicationPublicController`, caught before merge — `$request->validated()` fatal). 2026-08-26.
