# Implementation Plan for PR #13

## Problem Summary
The PR needs to:
1. Drop Python 3.11 from NumPy builds only (target 3.12+ i.e., 3.14, 3.13, 3.12)
2. Keep Python 3.11 for all other packages
3. Ensure Deploy job runs on main repo events but skips pull requests and forks
4. Remove temporary build artifacts from git history (force push)

## Current State (HEAD in worktree)

| File | State | Status |
|------|-------|--------|
| `.github/workflows/build.yml` (NumPy job, lines 294-308) | Uses `python: &python_3_12_up` → `["3.14", "3.13", "3.12"]` (no 3.11) | ✅ **Correct** - matches PR goal |
| `.github/workflows/build.yml` (Other jobs) | Use `*python_3_10_up` / `*python_3_11_up` (include 3.11) | ✅ **Correct** - keep 3.11 for other packages |
| `.github/workflows/build.yml` (Deploy job, line 562) | `if: ${{ always() && github.repository_owner == 'eeems' && github.event_name != 'pull_request' }}` | ✅ **Correct** - runs on main repo, skips PRs and forks |
| `build/action.yml` | Full choice list including 3.11 | ✅ **Correct** - reverted per "undo" request |
| `mirror/action.yaml:38` | `python-version: "3.11"` | ✅ **Correct** - leave unchanged per Eeems |
| `.gitignore` | Ignores `*.tgz`, `.agentty/`, `main_*.yml`, `main_*.yaml` | ✅ **Correct** |

## Approach
The source code changes are already complete and correct. The only remaining task is to **force push** the clean history (removing the temporary artifact commit) so CI can run against the correct state.

## Steps

### Step 1: Force Push Clean History
**File:** All files (git history)
**Action:** Execute `git push --force origin fix-ci-local-actions:fix-ci-local-actions` from the worktree
**Expected:** The artifact commit is removed from remote history; only the clean commit (`9c5fd923` or equivalent) remains on the PR branch
**Verification:** Check GitHub PR shows only the intended commits; CI starts automatically

### Step 2: Verify CI Results
**Action:** Wait for GitHub Actions workflow to complete
**Expected:** 
- NumPy builds run only on Python 3.14, 3.13, 3.12 (no 3.11)
- All other packages build on Python 3.11+
- Deploy job runs on push/workflow_dispatch/schedule (not on PR), only for eeems repo
- All jobs pass

## Testing
- GitHub Actions workflow runs automatically on push
- Verify in Actions tab that:
  - NumPy job matrix shows 3 versions (3.14, 3.13, 3.12)
  - Other jobs show 5+ versions including 3.11
  - Deploy job condition evaluates correctly

## Risks/Unknowns
- **Sandbox git push:** The worker sandbox may have permission issues pushing to the remote. If `git push --force` fails, the tool owner may need to force push from their local machine.
- **Deploy job condition:** Current condition `always() && github.repository_owner == 'eeems' && github.event_name != 'pull_request'` should skip PRs and forks. If Eeems wants different behavior (e.g., only on release), the condition would need adjustment.
- **CI verifier:** The automated verifier has been returning `no_available_model` / `invalid_response`. Real CI validation requires the force push to complete.