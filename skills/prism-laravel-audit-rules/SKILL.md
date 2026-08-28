---
name: prism-laravel-audit-rules
description: >
  Laravel/PHP-only project-local rules mirrored from the prism code-audit
  tool — NOT general-purpose, do not trigger for non-PHP code (Angular, TS,
  React, etc). The static ast-grep catalog has ZERO PHP/Laravel rules (it
  only covers angular/ag-grid/react/nestjs + TS-only shared rules) — every
  rule here comes from prism's live multi-agent reviewer prompts, which run
  based on framework detection (this repo detects as `laravel`, routing to
  php-reviewer, blade-reviewer, security-reviewer, architecture-reviewer,
  testing-reviewer, config-reviewer, performance-reviewer). Scoped to this
  repo's stack: Laravel 5.4.x, PHP 7.2.x (composer.json). Trigger: when
  writing or editing .php/.blade.php files, migrations, routes, config,
  Docker Compose files, or PHPUnit tests in this project, or when
  auditing/resolving PRISM PR review comments on backend code.
license: Apache-2.0
metadata:
  author: prism-derived
  version: "1.0"
  source: C:\Users\DESARROLLADOR\Documents\www\prism\src
---

# Prism Laravel Audit Rules (this project)

## Sources and routing

Unlike the frontendRh Angular mirror, **the static ast-grep rule catalog
(`prism/src/rule-catalog/`) does not apply to this repo at all** — confirmed
by reading the catalog directly: it only has `angular/`, `ag-grid/`,
`react/`, `nestjs/`, and `shared/{general,security,typescript}`, and every
pattern in those `shared/` rulesets is scoped to `*.ts/*.tsx/*.js/*.jsx`
(`language: 'typescript'`). No PHP, Blade, or Laravel-specific ast-grep
patterns exist in prism's source at all.

Everything below comes from prism's **live multi-agent reviewer prompts**
(`prism/src/ai/agents/prompts/*.txt`) instead. Routing is framework-driven
(`prism/src/ai/agents/framework-rules/routing-map.ts`,
`FRAMEWORK_AGENT_MAP`): a repo detected as `laravel` (this one — Laravel is
in `composer.json`'s `require`) gets exactly these 7 agents, and no others:

`php-reviewer`, `blade-reviewer`, `security-reviewer`,
`architecture-reviewer`, `testing-reviewer`, `config-reviewer`,
`performance-reviewer`.

Notably **absent**: `ts-reviewer`, `html-reviewer`, `css-reviewer` — those
only fire for JS/TS-framework repos (angular/react/vue/etc), not Laravel.

## Version gating (CRITICAL — read before suggesting any language feature)

This repo's `composer.json` pins `"php": "7.2.*"` and
`"laravel/framework": "5.4.*"`. `php-reviewer`'s own prompt is explicit:

> Only use the explicit "Language version" field from the framework context.
> NEVER invent or guess the PHP version. Do not infer it from Laravel
> version, code patterns, or any other indirect signal.

Given PHP 7.2, do **NOT** suggest: union types, named arguments, `match`,
nullsafe `?->`, enums, readonly properties, intersection types, fibers,
`never` return type, readonly classes, DNF types, typed class constants,
`json_validate()` (all 8.0+). Typed class **properties** need PHP 7.4+.

⚠️ **Correction (2026-08-28, PR 2120):** a prior version of this note claimed
"7.2.* covers 7.2–7.4 patch releases, so typed properties ARE fair game" —
**this is false and was never verified against the actual runtime.** The
backend container (`rh-rh-backend-1`) runs **PHP 7.2.34 exactly**, confirmed
via `docker exec rh-rh-backend-1 php -v`. Adding a native typed property
(e.g. `private Formato2276ExgoneaDianRepo $repo;`) on this runtime is a
**parse error**, confirmed via `docker exec rh-rh-backend-1 php -l <file>`:
`Parse error: syntax error, unexpected 'ClassName' (T_STRING), expecting
function (T_FUNCTION) or const (T_CONST)`. **Never infer the true PHP
version from the `composer.json` `X.Y.*` constraint** — it only pins the
minor version, not the deployed patch. Before suggesting or applying any
version-gated syntax, verify the real running version with
`docker exec rh-rh-backend-1 php -v`, and `php -l` any edited PHP file
through the same container (there is no `php` binary on the host — confirmed
`php: command not found`). Until the deployed container is actually bumped
past 7.4, use `/** @var Type */` docblocks instead of native property type
hints in this codebase. When in doubt, stick to what's available since
7.0–7.2: scalar type hints, return types, `??`, `<=>`, `void` return,
nullable `?type`, `strict_types`.

Same discipline applies to **Laravel framework API**, not just PHP syntax —
`composer.lock` confirms the exact installed version is **v5.4.36**, meaning
several 5.5+-only APIs (`FormRequest::validated()`, `Controller::validate()`
returning a value) don't exist yet. Full detail, the correct 5.4.x
replacement pattern, and the production incidents that hit this: see
`laravel-best-practices` skill §7 (single source of truth for this fact,
kept here only as a pointer to avoid drift between two copies).

**`Illuminate\Database\Query\Builder::upsert()` does not exist in 5.4.36**
(added in Laravel 8). Confirmed absent via
`vendor/laravel/framework/src/Illuminate/Database/Query/Builder.php`. Never
suggest `Model::upsert([...], $uniqueBy, $update)` as a batch-write fix in
this repo — fatal "Method not found". The correct 5.4.x batch pattern is:
one `whereIn()` read to find which unique keys already exist, then a plain
`update()` per existing row plus one bulk `insert()` for the new ones. Hit
via PR 2298 (`SyncDaneCommand` bulk city/department sync), 2026-08-27.

**The `now()` global helper does not exist in 5.4.36** (added in Laravel
5.8). Confirmed absent via `grep '^function now(' vendor/laravel/framework/src/Illuminate/Foundation/helpers.php`
(and across all of `vendor/`) returning nothing. Never suggest `now()` as a
timestamp shortcut in this repo — fatal "Call to undefined function". Use
`\Carbon\Carbon::now()` instead (established pattern, e.g. `GenerosSeeder`).
Hit via PR 2298 review follow-up (`CiudadRepo::upsertBatch`), 2026-08-27.

## php-reviewer (`ai/agents/prompts/php-reviewer.txt`)

1. **`declare(strict_types=1)` — ONLY flag on brand-new files** where the
   entire diff is addition lines (no context/deletion). **For existing
   files being modified: never flag it, not even at low severity** — adding
   it retroactively is a breaking change (silent type coercion that worked
   before now throws `TypeError`).
   ⚠️ **Tension with `php-domain-best-practices` skill**: that skill's §1 says
   *every* PHP file must start with `declare(strict_types=1)`. Read literally
   that's new-file-only in intent (Artisan-generated files don't get it by
   default) — but if you're ever asked to add it to an *existing* file being
   otherwise modified, treat prism's rule as authoritative: don't, unless the
   user explicitly asks for the migration.
2. Missing type declarations gated to the confirmed PHP version (see above).
   ⚠️ **Repo-specific fact**: `jsend_success()`/`jsend_error()`/`jsend_fail()` (`vendor/shalvah/laravel-jsend`, global helpers used across this codebase's controllers) all return `\Illuminate\Http\Response` via the `response($content, $status, $headers)` helper — **never `\Illuminate\Http\JsonResponse`**, despite returning JSON content. A controller method whose every return path goes through these helpers should be typed `: \Illuminate\Http\Response`; suggesting `: \Illuminate\Http\JsonResponse` is a fatal return-type mismatch at runtime (`TypeError`, since the actual object isn't an instance of `JsonResponse`). Verify in `vendor/shalvah/laravel-jsend/src/helpers.php` before suggesting either type. Hit via PR 2120 (`Formato2276ExgoneaDianController`), 2026-08-28.
   ⚠️ **Repo-specific exception**: never suggest typing `$id` as `int` (or
   any type hint at all) on a `BaseRepo` subclass override of a parameter the
   parent left untyped (`update()`/`destroy()`/`all()`, etc.) — `BaseRepo`
   declares these untyped, and PHP's parameter contravariance rule makes any
   override narrowing fatal at class-load time (not a `TypeError`, not a
   syntax error — `php -l` won't catch it). Full mechanism, the `$id`-string
   origin (`BaseController` never casts it), and the reproduction: see
   `laravel-best-practices` skill §1 (single source of truth). Hit in
   production via PR 2284 (`GeneroRepo`); the general-parameter version hit
   via PR 2120 (`Formato2276ExgoneaDianRepo::all()`), 2026-08-28.
3. Null safety — property/method access on nullables without `??`/null
   checks (`?->` is 8.0+, not available here).
4. **SQL injection via raw queries** — string concat in SQL;
   `DB::raw()`/`whereRaw()`/`selectRaw()` with unescaped input. (critical)
5. Type juggling — loose `==` across types; `in_array()` without
   `strict: true`.
6. Swallowed exceptions — empty catch, catch-all without rethrow.
7. Resource leaks — unclosed handles/streams/DB connections, missing
   `finally`.
8. Deprecated-for-this-version function usage: `mysql_*` (removed 7.0),
   `each()` (deprecated 7.2 — **applies here**), `create_function()`
   (deprecated 7.2 — **applies here**).
9. **Unsafe deserialization** — `unserialize()` on user data without
   `allowed_classes`. (critical)
10. Fire-and-forget — silently lost exceptions from queued jobs/events.

PSR compliance checked (structure, not formatting): PSR-1 (no short `<?`
tags, one declaration-or-side-effect per file, `StudlyCaps`/`UPPER_CASE`),
PSR-4 (namespace matches directory, one class per file), PSR-3 (log via
`LoggerInterface`, interpolation placeholders not concatenation), PSR-12
(only the structural placement of an existing `declare(strict_types=1)`
block — never formatting/indentation/braces).

## blade-reviewer (`ai/agents/prompts/blade-reviewer.txt`)

1. **Unescaped output** — `{!! $var !!}` on anything user-controlled MUST be
   `{{ }}` instead. (critical if user data)
2. **Missing `@csrf`** on every POST/PUT/PATCH/DELETE form. (critical)
3. Missing `@method('PUT'|'PATCH'|'DELETE')` spoofing directive.
4. Business logic in templates — `@php` blocks with DB queries or complex
   computation (belongs in a controller/service).
5. `@inject` pulling services directly into views.
6. Raw HTML attribute injection — unescaped dynamic values in attributes,
   `javascript:` URLs.
7. Accessibility — missing `alt`, unlabeled inputs, invalid ARIA.
8. Component anti-patterns — inline HTML that should be extracted, 3+
   nesting levels, missing slots.

PHP code quality *inside* `@php` blocks is php-reviewer's job, not
blade-reviewer's — don't duplicate.

## security-reviewer (`ai/agents/prompts/security-reviewer.txt`, live, runs on every file)

Broader OWASP sweep, `medium` severity minimum:

- SQL injection (concat/template literals in queries).
- Auth bypass — hardcoded creds, missing auth checks, JWT without signature
  verification, session fixation.
- Hardcoded secrets/tokens/keys in source.
- XSS via unsanitized DOM/template output.
- Path traversal — user input in file paths without normalization.
- Insecure deserialization — `unserialize()`/`eval()`/`Function()` on
  untrusted input without schema validation.
- SSRF — user-controlled URLs to server-side HTTP clients, unchecked.
- Missing rate limiting on login/OTP/sensitive endpoints.
- **Repo-specific exception**: a `FormRequest::authorize()` returning only
  `auth()->check()` is NOT a missing-permission finding when the route
  already carries an Entrust `permission:*` middleware — RBAC is enforced at
  the route layer for this project. Check the route before flagging this;
  full detail and the inverse gotcha (a `FormRequest` never type-hinted on
  any action method means `authorize()` never runs at all, permission or
  not) are in `laravel-best-practices` skill §6.
- Sensitive data (passwords, tokens, PII) in logs or API error responses.
- Insecure crypto — MD5/SHA1 for security, ECB mode, weak RNG for tokens.
- **Repo-specific note (mass assignment)**: when a `BaseController` subclass's `$validation` array includes a sensitive field (status flags, role/permission ids, etc.), the fix is NOT "remove it from `$validation`" — `BaseController::validateRequest()` returns `$request->all()` regardless of that array, it's validation-only, not a mass-assignment filter. Check the model's `$fillable` array instead — that's the actual gate. Flag the model's `$fillable`, not the controller's `$validation`, as the file to fix. See `laravel-best-practices` skill §1b. Hit via PR 2279 (`EmpleadoPerfilController`/`Perfil`), 2026-08-27.
- **Repo-specific exception**: `Log::error($msg, ['message' => $e->getMessage()])`
  (or equivalent) is NOT an exception-detail leak when the client-facing
  response stays a fixed generic string — that's the established, correct
  pattern in this repo (log real error, return generic message; already in
  26+ occurrences across 12 controllers). Only flag this if the raw
  exception message actually reaches the HTTP response body or a
  client-visible field. Full rule, including which exception types are safe
  to show verbatim vs which need genericizing: `laravel-best-practices`
  skill §2 (Rules 3 and 5).

## architecture-reviewer (`ai/agents/prompts/architecture-reviewer.txt`, live, whole-diff)

Emits **0-3 findings max**, critical/high only — high-signal by design:

- Circular dependencies between modules/packages.
- SRP violations — a class doing unrelated things (e.g. a model handling
  HTTP calls; a service mixing business logic with persistence).
- Layer boundary crossings — e.g. UI/presentation importing DB models
  directly; infra code importing domain entities incorrectly.
- Inappropriate coupling between components that know too much of each
  other's internals.
- Leaked abstractions — one layer's implementation detail (e.g. a raw SQL
  type, an HTTP status code) bleeding into a layer that shouldn't know it.

This agent has **no built-in awareness** of this repo's
`BaseController`/`BaseRepo` convention (that's this repo's own pattern from
`laravel-best-practices`, not something prism ships by default) — don't
expect it to enforce that pattern; it reasons from generic SRP/coupling
principles only.

- **Repo-specific exception**: don't suggest type-hinting a specific
  `FormRequest` directly in the signature of a `BaseController` override
  (`index`/`store`/`update`/`destroy`/`show`/`history`/`Softdelete`) to "let
  the standard Laravel pipeline handle validation/authorization". These are
  concrete methods on `BaseController` declaring `Request $request` — PHP
  enforces parameter contravariance on class overrides (not just
  interfaces), so narrowing to a `FormRequest` subtype throws "Declaration
  should be compatible with BaseController->store(request: Request)". The
  fix in this repo is to resolve the FormRequest manually inside the method
  body (`app(VacancyRequest::class)`), which already triggers `authorize()`
  + validation as a side effect of container resolution in this project's
  pinned Laravel v5.4.36 (`FormRequestServiceProvider::boot()` registers
  `afterResolving(ValidatesWhenResolved::class, ...)`) — so "the pipeline is
  being skipped" is not actually true here. See `laravel-best-practices`
  skill §1. This exception does NOT apply to a controller's own
  custom-named methods that don't override `BaseController` at all (e.g.
  `ApplicationPublicController::storeApplication()`) — those can type-hint
  the FormRequest directly with no conflict. Hit via PR 2289
  (`VacancyController`), 2026-08-26.

- **Repo-specific exception**: don't flag a migration that runs
  `DROP VIEW ... CASCADE` without recreating the view in the same file (`up()`
  or `down()`). This repo's deploy pipeline runs `php artisan sql:scripts`
  immediately after `php artisan migrate` on every deploy (see
  `docker-entrypoint.sh`) — every file in `database/scripts/*.sql` is
  `CREATE OR REPLACE VIEW`/`FUNCTION`, so it always rebuilds every view right
  after migrations run. Recreating the view inside the migration itself would
  be redundant, not a fix for a gap. Precedent documented in
  `2026_03_20_104500_change_observaciones_to_text_in_nomina_novedades_table.php`.
  Hit via PR 2277 (`alter_empleados_sexo_to_varchar`), 2026-08-27.
- **Repo-specific exception**: don't flag a migration that calls a Seeder's
  `run()` method directly (e.g. `(new GenerosSeeder())->run()`), or that uses
  `PermissionRoleSeeder`/`App\Role`/`App\Permission` to assign permissions,
  as improper coupling between schema and application code. `php artisan
  db:seed` is never run by this project's deploy pipeline — only `migrate`
  and `sql:scripts` run automatically — so seeding catalog data or
  permissions has to happen inside a migration for it to reach production.
  This is an established, repeated pattern (16+ prior permission migrations,
  e.g. `2026_06_15_000000_add_news_permissions_to_administradores.php`), not
  a new violation to fix per-PR. Flag it only if the seeder itself is not
  idempotent (no `updateOrInsert`/`firstOrCreate` guard). Hit via PR 2277
  (`add_generos_permissions_to_administradores`,
  `run_generos_seeder`), 2026-08-27.

## testing-reviewer (`ai/agents/prompts/testing-reviewer.txt`, live — applies to PHPUnit tests here)

Tests that give false confidence:

1. Weak/missing assertions — checking a function was called but not what it
   returned; bare `toBeDefined()`-equivalents when correctness matters.
2. Always-passing tests — asserting on a mock's own return value.
3. Shared mutable state across tests without reset between cases.
4. Mock overuse — every collaborator mocked, only mock interactions
   asserted, no real logic verified.
5. Flaky async — timers without fakes, missing awaits, race conditions.
6. Asserting on private/internal implementation details that break on valid
   refactors without catching real bugs.

Explicitly out of scope: missing tests for untested code, naming
conventions, file organization, coverage gaps.

## config-reviewer (`ai/agents/prompts/config-reviewer.txt`, live — relevant: this repo has Dockerfile/Dockerfile.dev/docker-compose.yml/docker-compose.dev.yml)

1. Accidental secrets — `password`/`secret`/`api_key`/`token`/`connection_string`
   keys with real-looking values (32+ char hex/base64, `sk_live_*`, `AKIA*`,
   `xoxb-*`). `.env.example` placeholders (`YOUR_*`, `CHANGEME`, etc.) are
   correct and must NOT be flagged.
2. Hardcoded env-specific values — literal `localhost`/IPs/absolute paths
   outside Docker-internal files; DB names with `prod`/`staging` not behind
   an env var.
3. **Docker Compose security** (directly applicable — this repo has 2
   compose files): `privileged: true` without justification; ports bound to
   `0.0.0.0` exposing internal-only services; bind mounts to `/`, `/etc`,
   `/root`, `/var/run/docker.sock`; `network_mode: host` unjustified;
   missing `healthcheck` on DB/queue services; `latest` image tag (medium).
4. Duplicate keys in JSON/YAML (last-wins silently); TOML table
   redefinition.
5. Permissive defaults — `cors: "*"`, `debug: true`/`DEBUG=true` outside dev
   configs, `ssl: false`/`verify: false` in non-dev contexts.
6. Missing required fields in known schemas — Compose services without a
   `restart` policy, etc.

Never flagged: lockfiles (`composer.lock` included), `vendor/`, IDE settings,
test fixture configs (relaxed to critical-only).

## performance-reviewer (`ai/agents/prompts/performance-reviewer.txt`, live — ZERO static-catalog coverage, 100% live judgment)

Every finding must name the concrete degrading scenario, not a theoretical
optimization:

1. **N+1 queries** — the classic Eloquent trap: relation access inside a
   loop without eager-loading (`->with(...)`); DB/API calls once per item
   instead of batched. When suggesting a batch-write fix for repeated
   `updateOrCreate()`/`create()`/`update()` inside a loop, do NOT suggest
   `Model::upsert()` — see the version-gating note above, it doesn't exist
   in this repo's Laravel 5.4.36.
2. Unbounded loops/recursion with no cap on collection size.
3. Memory leaks — growing collections, closures over large objects,
   listeners never removed.
4. **Missing pagination** — endpoints/queries returning unlimited result
   sets with no `limit`/`offset`/cursor (`paginate()` vs `get()` in
   Eloquent).
5. **Unbounded nested array validation** — a `FormRequest` rule like
   `'questions' => 'nullable|array'` with wildcard sub-rules
   (`questions.*.text`, `questions.*.options.*`) has no cap on how many
   elements the caller can submit; validation cost scales with payload size
   with no ceiling. Suggest `array|max:N` on both the outer array and any
   nested wildcard array (e.g. `questions` and `questions.*.options`).
   Real fix applied: `VacancyRequest`, PR 2289, 2026-08-26.
6. Large bundle imports (less relevant server-side, still checked).
6. Synchronous I/O in hot paths — blocking file reads/network calls in
   request handlers or queued job handlers.
