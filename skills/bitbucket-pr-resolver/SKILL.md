---
name: bitbucket-pr-resolver
description: >
  Connects to Bitbucket, fetches PR comments, creates a resolution plan, and applies project-specific coding skills to resolve them.
  Trigger: When the user asks to resolve PR comments, review a PR, or provides a PR number.
license: Apache-2.0
metadata:
  author: gentleman-programming
  version: "1.0"
---

## When to Use

- When resolving coding review comments on Bitbucket.
- When creating an action plan based on PRISM or human feedback on a PR.

## Critical Patterns

1. **Information Gathering**:
   - Fetch the active comments using the script `C:\Users\DESARROLLADOR\.claude\get_pr_comments.ps1`.
   - Pass the correct `-$prId` and adapt the workspace/repo in the script if needed.

2. **Project-Specific Best Practices**:
   - Before executing code changes, read the local `.atl/skill-registry.md` or use `mem_search` to load the current project's coding rules and conventions.

3. **Analysis and Planning**:
   - Generate a markdown checklist mapping each comment ID to the affected file and a diagnosis.
   - Evaluate if the finding is:
     - **False positive**: Document why.
     - **Real bug**: Plan the code fix.
     - **Architecture / Tech debt**: Plan to defer the fix and document the justification.

4. **Approval**:
   - Present the plan to the user. Wait for their explicit confirmation (e.g., "Sí") before modifying any code.

5. **Execution**:
   - Fix real bugs in the code.
   - Use `C:\Users\DESARROLLADOR\.claude\post_pr_reply.ps1` to reply to false positives or tech debt with technical justification (do not resolve them).
   - Use `C:\Users\DESARROLLADOR\.claude\resolve_comment.ps1` to resolve comments that are real bugs or simple typos that you have already fixed.

6. **Documentation and Sync**:
   - ALWAYS save the final decisions, justifications, and bugs fixed in Engram using `mem_save`.
   - After saving the decisions locally, immediately sync the memories to the cloud using the terminal command: `engram sync --cloud --project <project_name>`.
   - **CRITICAL**: NEVER use `antigravity` as the project name when saving to Engram or syncing to the cloud. You must explicitly pass the actual name of the project repository (e.g. `frontend_rh`, `backend_rh`, etc.).

7. **Skill Registry Update**:
   - If a comment resulted in a code adjustment (e.g. fixing an anti-pattern) or revealed a new project-specific coding rule, you must extract that lesson.
   - Trigger the `skill-registry` skill (e.g., by saying "update skills" or invoking it directly) to document the new pattern in the project's `.atl/skill-registry.md` so the AI doesn't repeat the same error in the future.

## Code Examples

*Example plan format:*
```markdown
### 1. Typo in HTML (Comment 12345)
- **File**: `src/app/...`
- **Diagnosis**: Real bug. Missing accent.
- **Action**: Fix code, then resolve comment.
```

## Commands

```bash
# Fetch comments
$prId = "<PR_ID>"
# Execute local modified snippet or the original script
C:\Users\DESARROLLADOR\.claude\get_pr_comments.ps1 -prId $prId

# Reply to a comment
C:\Users\DESARROLLADOR\.claude\post_pr_reply.ps1 -prId $prId -commentId <ID> -message "Justification..."

# Resolve a comment
C:\Users\DESARROLLADOR\.claude\resolve_comment.ps1 -prId $prId -commentId <ID>
```
