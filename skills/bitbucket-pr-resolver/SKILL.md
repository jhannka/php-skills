---
name: bitbucket-pr-resolver
description: "Trigger: Bitbucket PR, PR comments, PRISM findings, review comments, resolve comments, PR number in the softgbq workspace. Fetch, triage, fix, reply and resolve review comments, then record new rules in the project skills."
license: Apache-2.0
metadata:
  author: rh-team
  version: "2.0"
  scope: project
  stack: "Bitbucket REST 2.0, PowerShell scripts, workspace softgbq, base branch develop"
---

## Activation Contract

Load when the user asks to resolve, review or answer comments of a Bitbucket pull request, or gives a PR number. Needs three per-developer scripts (`get_pr_comments.ps1`, `post_pr_reply.ps1`, `resolve_comment.ps1`) in `<your-scripts-folder>`; never commit them or their credentials.

## Hard Rules

- Backend PRs live in repo `rh_new`: pass `-repo rh_new` to every script call (they default to `frontend_rh`).
- Verify each claim in the code before accepting it. Wrong premise: reply with evidence and leave it open. Half right: fix the wording, reply with the precise limit, leave the debt open.
- Fixes follow strict TDD (skill `rh-testing-phpunit`); run one test class per call.
- Reply and leave open for false positives and tech debt. Resolve a comment only after its fix is pushed.
- Commit and push are separate decisions: commit on the PR branch, ask before pushing, then publish replies and resolve in one pass.
- Conventional commits; no AI attribution and no `Co-Authored-By`.
- Spanish text with accents breaks from the shell: write the `.ps1` as UTF-8 **with BOM** and run it exactly once.
- Never print the API token.

## Decision Gates

| Comment is | Action |
|------------|--------|
| Real bug or simple typo | Fix, reply `Corregido en <sha>: ...`, resolve after the push |
| False positive | Reply with evidence, leave open |
| Architecture / tech debt | Reply with the justification, leave open |
| Rule someone could break again | Record it in a project skill (Execution Steps, step 6) |

## Execution Steps

1. Fetch the active comments with `get_pr_comments.ps1 -prId <ID> -repo rh_new`.
2. Read `.agents/skills/README.md` and the `SKILL.md` that match the files each comment touches.
3. Write a checklist: comment ID, file, diagnosis, decision, and the skills to update.
4. Present the plan and wait for explicit approval (code and skill changes) before editing.
5. Apply the fixes, run the tests, commit.
6. Update skills (mandatory after the fixes are committed). For each fixed comment, or false positive worth recording, ask whether someone could break the rule again in this repo; if not, stop. Otherwise add one verifiable rule to the skill in the README table that covers the area (recurring false positives go to `rh-prism-audit`); if none covers it, create `rh-<topic>/SKILL.md` following the README standard and add it to the table. Bump `metadata.version`, cite the proof, commit with `docs(skills): ...`, and refresh your tool's skill index (README, "Registry").
7. After the user approves the push, post replies and resolve with the other two scripts.

## Output Contract

Final report: comments fixed, replied, and left open (with reasons), commits, tests run with results, and the skills updated or created. Edge cases and API details: `references/gotchas.md`.
