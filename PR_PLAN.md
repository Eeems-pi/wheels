# Plan for PR Eeems/wheels#13

## Objective
Remove Python 3.11 from NumPy wheel builds only, while keeping 3.11 for all other packages.

## Current State (from investigation)

### `.github/workflows/build.yml`
- **numpy job** (lines 297-315): Uses `python: *python_3_12_up` which defines `["3.14", "3.13", "3.12"]` — **correct** (3.11 excluded)
- **standard, protobuf, cffi, indexed-gzip, pillow, arm-3_11-up, bcrypt, cryptography, wxpython**: All use `*python_3_10_up` or `*python_3_11_up` anchors which include 3.11 — **correct**
- **Deploy job** (line 557): Has `if: ${{ always() && github.repository_owner == 'eeems' }}` — per owner: "do not modify the deploy job"

### `build/action.yml`
- **python_version** input: Has full choice list including `"3.11"` — **correct** (owner said "undo the changes / revert this file")
- **build_on** input: Full choice list intact — **correct**

### `mirror/action.yaml`
- **python-version**: Still `"3.11"` (line 38) — owner said "This should not have been changed" — **leave as-is**

### Temporary Artifacts (need removal)
The workspace contains build artifacts committed in error:
- `np-*.tgz` (NumPy wheels)
- `cy*.tgz`, `cy*.toml` (cryptography wheels)
- `pyproject-*.toml`, `pp-*.txt`, `mb-*.txt`, `u246.txt` (metadata)
- `main_*.yml`, `main_*.yaml` (draft workflows)
- `.agentty/` directory

`.gitignore` already updated to ignore `*.tgz`, `.agentty/`, `main_*.yml`, `main_*.yaml`

## Required Actions

### 1. Remove Temporary Artifact Commit
The artifacts were accidentally committed. Need to:
- Remove all temporary files from git history
- Force-push to clean the branch

### 2. Verify NumPy Matrix is Correct
Current state: `python: *python_3_12_up` → `["3.14", "3.13", "3.12"]` (no 3.11)
This matches the PR objective: "drop 3.11 from numpy builds"

### 3. No Changes Needed to Other Files
- `build/action.yml`: Already reverted (full choice list with 3.11)
- `mirror/action.yaml`: Leave at 3.11
- Deploy job: Leave unchanged per owner instruction

## Verification
After force-push:
1. Trigger CI run (push test commit or re-run workflow)
2. Verify:
   - NumPy builds run only on 3.14, 3.13, 3.12
   - All other packages still build on 3.11+
   - Deploy job runs only on eeems repository (fork skip works)
   - No build failures from the changes

## Blockers
- Cannot run git commands in sandbox (`.git` file points to inaccessible worktree)
- Owner must force-push from local machine or grant access
- CI verifier returns `no_available_model` / `invalid_response` — manual CI run needed

## Next Steps for Owner
1. **Force-push** to remove artifact commit from history
2. **Confirm** NumPy matrix is correct as-is (3.14/3.13/3.12 only)
3. **Trigger CI** and verify all jobs pass
4. **Approve PR** if CI passes