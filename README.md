# php-skills

A collection of reusable AI agent skills for PHP/Laravel backend development,
plus a few general-purpose workflow skills (PR review, memory discipline,
skill authoring). Originally consolidated from a Laravel 5.4 / PHP 7.2 HR
backend project.

Each skill is a self-contained `SKILL.md` file: YAML frontmatter (`name`,
`description`, `license`) followed by a markdown body with the actual rules,
examples, and gotchas. This format works with any tool that can load a
markdown file into an agent's context — Claude Code skills, a system prompt
section for any other LLM, or just documentation a human reads before
writing code.

## Skills

| Skill | What it's for |
| --- | --- |
| [`laravel-best-practices`](skills/laravel-best-practices/SKILL.md) | BaseController/BaseRepo architecture, exception handling, migration keys, route constraints — the widest-scope Laravel skill here |
| [`php-domain-best-practices`](skills/php-domain-best-practices/SKILL.md) | PHP `strict_types`, `use` statement ordering, naming conventions |
| [`prism-laravel-audit-rules`](skills/prism-laravel-audit-rules/SKILL.md) | Rules mirrored from the PRISM code-audit tool's Laravel reviewer agents — useful reference even without PRISM itself |
| [`bitbucket-pr-resolver`](skills/bitbucket-pr-resolver/SKILL.md) | Fetch Bitbucket PR review comments, triage them (real bug / false positive / tech debt), fix and reply |
| [`comment-writer`](skills/comment-writer/SKILL.md) | Tone/voice guidance for PR feedback, issue replies, and review comments |
| [`judgment-day`](skills/judgment-day/SKILL.md) | Blind dual-judge adversarial code review protocol |
| [`memory-protocol`](skills/memory-protocol/SKILL.md) | Discipline for using Engram-style `mem_*` persistent-memory tools |
| [`engram-cloud-setup`](skills/engram-cloud-setup/SKILL.md) | Configuring a project for Engram Cloud memory sync |
| [`engram-save-sync`](skills/engram-save-sync/SKILL.md) | Save-and-sync workflow for Engram memory |
| [`project-initializer`](skills/project-initializer/SKILL.md) | Bootstrap SDD + memory + skill-registry setup for a new project |
| [`skill-creator`](skills/skill-creator/SKILL.md) | Meta-skill: how to write a new `SKILL.md` |

## Usage

**Claude Code**: drop a skill folder into `.agents/skills/` (project-local)
or your global skills directory (user-level) and it becomes available.

**Any other model/tool**: paste the relevant `SKILL.md`'s body into your
system prompt or context before starting work, or point your agent at the
raw file URL.

## Notes on portability

A few of these skills (`laravel-best-practices`, `prism-laravel-audit-rules`)
were written against one specific codebase and cite that project's exact PHP
(7.2.34) and Laravel (5.4.36) versions, plus internal incident references
(PR numbers, a Docker container name). Treat those version-pinned facts as
examples of *how to verify*, not universal truths — always confirm the
actual PHP/Laravel version of the repo you're applying them to before
trusting a version-gated rule.

## License

Apache-2.0 (see each `SKILL.md`'s frontmatter).
