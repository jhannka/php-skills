---
name: engram-cloud-setup
description: >
  Configures any project for Engram Cloud, enrolls it, repairs legacy session directory mismatches,
  runs doctor diagnostics, applies cloud upgrades, and synchronizes state.
  Trigger: When the user asks to configure, setup, enroll, or connect a project to Engram Cloud.
license: Apache-2.0
metadata:
  author: gentleman-programming
  version: "1.0"
---

## When to Use

Use this skill to onboard or configure a project to sync with the Engram Cloud server (`http://192.168.54.128:18080`). It ensures that:
- The Engram Cloud server is properly set.
- The project is enrolled.
- Any legacy DB inconsistencies (missing session directories, observation payload issues) are repaired using `repair-missing-session-directory.sh`.
- The Engram Cloud upgrade flow (doctor, repair) is run successfully.
- The project state is synced with the cloud.

## Critical Patterns

When executing this configuration:
1. **Project Name Restriction**: The project name MUST NOT be `antigravity`. Never enroll or configure a project named `antigravity` to Engram Cloud.
2. **Server Address**: Always use `http://192.168.54.128:18080`.
2. **ENGRAM_CLOUD_TOKEN**: This token must be set in the environment. If it is not present in the current terminal environment, the script will prompt the user or fail gracefully.
3. **Repair Script**: The script `repair-missing-session-directory.sh` is located globally at `C:\Users\DESARROLLADOR\Documents\repair-missing\repair-missing-session-directory.sh`. Always execute it using `bash` with `--apply --all` options.
4. **Execution Order**: The steps must be followed exactly in this order:
   1. Configure server: `engram cloud config --server http://192.168.54.128:18080`
   2. Export/set `ENGRAM_CLOUD_TOKEN`
   3. Enroll project: `engram cloud enroll <project_name>`
   4. Run local repair: `bash C:/Users/DESARROLLADOR/Documents/repair-missing/repair-missing-session-directory.sh --apply --all <project_name>`
   5. Run doctor: `engram cloud upgrade doctor --project <project_name>`
   6. Run upgrade repair: `engram cloud upgrade repair --project <project_name> --apply`
   7. Sync cloud: `engram sync --cloud --project <project_name>`

## Commands

To run the setup manually:

### Windows PowerShell
```powershell
# 1. Set server config
engram cloud config --server http://192.168.54.128:18080

# 2. Set token (replace <your_token> with actual token)
$env:ENGRAM_CLOUD_TOKEN = "<your_token>"

# 3. Enroll project
engram cloud enroll <project_name>

# 4. Repair session directory using bash
bash C:/Users/DESARROLLADOR/Documents/repair-missing/repair-missing-session-directory.sh --apply --all <project_name>

# 5. Diagnostics
engram cloud upgrade doctor --project <project_name>

# 6. Apply upgrades
engram cloud upgrade repair --project <project_name> --apply

# 7. Synchronize
engram sync --cloud --project <project_name>
```

### Bash / Git Bash
```bash
# 1. Set server config
engram cloud config --server http://192.168.54.128:18080

# 2. Export token (replace <your_token> with actual token)
export ENGRAM_CLOUD_TOKEN="<your_token>"

# 3. Enroll project
engram cloud enroll <project_name>

# 4. Repair session directory
bash C:/Users/DESARROLLADOR/Documents/repair-missing/repair-missing-session-directory.sh --apply --all <project_name>

# 5. Diagnostics
engram cloud upgrade doctor --project <project_name>

# 6. Apply upgrades
engram cloud upgrade repair --project <project_name> --apply

# 7. Synchronize
engram sync --cloud --project <project_name>
```

## Resources

- **Helper Script**: A script is available at [scripts/setup-cloud.sh](scripts/setup-cloud.sh) to execute all steps sequentially.
- **Repair Script**: The main repair utility resides at [repair-missing-session-directory.sh](file:///C:/Users/DESARROLLADOR/Documents/repair-missing/repair-missing-session-directory.sh).
