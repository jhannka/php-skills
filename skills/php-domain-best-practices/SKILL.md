---
name: php-domain-best-practices
description: >
  Global PHP strict-types, use-statement, and naming conventions for backend classes.
  Trigger: When creating/refactoring PHP classes in this Laravel repo.
license: Apache-2.0
metadata:
  author: gentleman-programming
  version: "1.1"
---

# PHP Strict Types & Naming Conventions

This skill consolidates global rules for PHP file structure and naming in this repo.

⚠️ **2026-08-28**: sections 2–4 of this skill (DTO Pattern, Enums, Domain Model
Architecture & Layer Isolation) were pure Angular/TypeScript content —
misplaced in a PHP-backend skill file. Moved to
`frontendRh/.agents/skills/domain-model-best-practices/SKILL.md`. This file
now covers PHP-only conventions.

## 1. PHP Strict Types & Use Statements

- **Strict Types**: Every single PHP file created must begin with `declare(strict_types=1);` immediately after the opening `<?php` tag. (Remember to add this manually if generated via Artisan).
- **Use Statements**: Always import classes/facades with `use` statements at the top of files instead of using fully qualified class names inline. This applies to any FQCN, not just framework classes — e.g. `use Carbon\Carbon;` + `Carbon::now()`, never `\Carbon\Carbon::now()` inline. Caught via PR 2298 review follow-up (`CiudadRepo::upsertBatch`), 2026-08-27.
- **Ordering**: 1. Framework facades (Illuminate), 2. Cms classes, 3. App classes, 4. Exceptions, 5. Traits/Interfaces.
- **Naming language**: New variables, methods, and classes must be named in English, even in files under the `Cms\` domain layer. This repo's DB schema and existing domain models are Spanish (`nombre`, `codigo`, `departamento_id`, `Ciudad`, `Perfil`, `CiudadRepo`...) — keep using those exact Spanish names when referencing an existing column, model, or class (renaming them is a much bigger, separate change). But local variables and any new method/class you introduce should be English (`$cityIdsByName`, `upsertBatch()`, not `$ciudadesPorNombre`, `procesarLote()`). Confirmed via PR 2298 review follow-up, 2026-08-27.

**Example**:
```php
<?php
declare(strict_types=1);

namespace App\Services;

use Illuminate\Support\Facades\DB;
use App\Exceptions\SettlementException;

class MyService { ... }
```
