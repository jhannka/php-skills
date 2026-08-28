---
name: project-initializer
description: >
  Initializes a project with SDD, validates Engram memories and project names, checks specialized skills, and synchronizes to Engram Cloud.
  Trigger: When the user asks to initialize a project, setup a new project, or says "project init", "iniciar proyecto", "preparar proyecto".
license: Apache-2.0
metadata:
  author: gentleman-programming
  version: "1.0"
---

## When to Use

- Starting work on a new or existing project.
- Initializing SDD (Spec-Driven Development) context for a workspace.
- Verifying Engram memory configurations and correcting project names.
- Synchronizing project state to Engram Cloud.

## Critical Patterns

1. **SDD Init**: Always start by invoking the `sdd-init` skill to bootstrap the Spec-Driven Development project context.
2. **Engram Memory Validation**: 
   - Ensure memories are saved under the correct **project name**.
   - **NEVER** save memories under the generic "antigravity" name. If any are found, they must be corrected.
   - Always ensure that the prompts written by the user are being saved (`capture_prompt: true` in `mem_save`, or using `mem_save_prompt`).
3. **Skill Review**: 
   - Review specialized skills (typically in `.agents/skills/` or `.gemini/skills/`) for the specific project.
   - Adjust the code or markdown of these skills if there are errors or outdated instructions.
4. **Engram Cloud Synchronization**: 
   - Synchronize everything to Engram cloud.
   - Validate that the uploaded data uses the correct project name (NOT "antigravity").
   - Always ensure the following are uploaded during sync: 
     - `Sessions`
     - `Observations`
     - `Prompts`
     - `Mutations`

## Execution Steps

1. Run SDD Init context initialization (e.g., using `sdd-init`).
2. Call `mem_review` or equivalent tools to check the current project name in memories. Fix any memories saved as "antigravity" using `mem_update` to the correct project name.
3. Check the project's custom skills. Read them and fix any syntax or logical errors found.
4. Invoke `engram-cloud-setup` or run the appropriate sync command, explicitly verifying the correct project name is used for synchronization, and confirming that Sessions, Observations, Prompts, and Mutations are pushed.

## Examples

### Correct Memory Save Example
When saving a memory, ensure `capture_prompt` is enabled and it is in the `project` scope (which must resolve to the real project name):
```json
{
  "title": "Configured project initialization",
  "type": "config",
  "scope": "project",
  "capture_prompt": true,
  "content": "Initialized project structure and SDD."
}
```
