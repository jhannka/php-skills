# Gotchas (learned resolving PR 1824, 2425, 2427, 1828, 2442 and 2460)

## Scripts and shell

1. **Right repo.** All three scripts default to `-repo frontend_rh`. Backend PRs live in `rh_new`: pass `-repo rh_new` to every call, or you read and reply to the wrong PR.
2. **Accents get corrupted from the shell.** Spanish text with tildes passed as an argument is mangled. Write the calls in a `.ps1` saved as UTF-8 with BOM (`printf '\xEF\xBB\xBF' > f.ps1`, then append) and run `powershell -NoProfile -ExecutionPolicy Bypass -File f.ps1`. The fetch script's console output also shows mojibake; that is only display.
3. **Script creation.** Create the `.ps1` as a file, then prepend the BOM with a small PowerShell step. Long shell heredocs with quotes and accents break the parse. Run the posting script exactly once, never twice in one command.
4. **Integer keys in `[ordered]@{}` are positional.** `$d[$k]` with an integer `$k` returns `$null`: a reply script once posted 28 EMPTY comments. Iterate `$d.GetEnumerator()` (`[int]$e.Key`, `$e.Value`) or use string keys.
5. **Allow-list rules match the whole command.** A rule for `powershell ... -File *get_pr_comments.ps1*` does not cover a compound command (`printf ... && powershell ...`). Prepend the BOM in its own call, then run the script alone. Reading comments and posting/resolving need separate rules, and only the user can add them.

## Triage

6. **Verify each claim before accepting it.** Some findings are wrong (e.g. a "payroll is not accounted" claim when the fixture already defaults to CONTABILIZADA). Reply with the evidence instead of changing code.
7. **A finding can be half right.** PRISM said `withoutOverlapping(60)` is a timeout; it is only a lock expiry. Fix the misleading wording, reply with the precise limit, leave the debt open.
8. **Range validations check every date, not the ends.** A finding on "only start/end are validated" is real for holidays and any per-day block; for working-day rules it can be a business decision (Fri-Mon permits), so reply and leave open.
9. **Eloquent casts neutralize `(bool) '0'` findings.** Check `$casts` on the model before accepting a truthiness finding.
10. **Count and classify first.** Replies-only (false positives, tech debt) vs FIX NOW. Expected volume: a PRISM review per PR yields about 12-30 findings, roughly 25% real fixes and 75% reply-only.

## Fixes

11. **TDD.** Write the failing test first when strict TDD is on, run each PHPUnit or Karma class separately, report RED then GREEN. A change that alters timing (e.g. a delayed `revokeObjectURL`) can break an existing spec that asserted the old behaviour; update that spec.
12. **Run PHPUnit inside the backend container.** Host PHP cannot resolve the `db` host: `docker compose -f docker-compose.dev.yml exec rh-backend vendor/bin/phpunit <file>`, one test class per call.
13. **`down()` must not destroy what `up()` skipped.** If `up()` is guarded with `hasTable`, guard `down()` too (verify in dev with and without rows).
14. **No AI attribution or `Co-Authored-By` on fix commits.** `git commit --amend` before the push if one slips in.

## Publishing

15. **Resolve only after the push.** Resolving before the push claims a fix the reviewer cannot see.
16. **Verify what was published through the API.** `GET .../comments?pagelen=100`: parent present, `content.raw` non-empty, `deleted` flag. Bitbucket keeps deleted comments as placeholders (`deleted=true`), so totals stay inflated. `DELETE .../comments/{id}` removes your own wrong replies.
17. **Creating PRs.** `POST /2.0/repositories/softgbq/{repo}/pullrequests` with `source.branch.name`, `destination.branch.name` = `develop`, `close_source_branch` = true, title and description in Spanish. The token is parsed at runtime from `post_pr_reply.ps1` and never printed.
