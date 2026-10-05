# Implementation Plan for PR #13

## Problem Summary
The PR aims to drop Python 3.11 from NumPy builds only (targeting 3.12+), while keeping 3.11 for all other packages. Additionally, the Deploy job should run on all main-repo events except pull requests, and should skip forks. Temporary build artifacts were accidentally committed and need to be removed from history via force push.

## Current State (Verified)
All source changes are **already correct** in the worktree:

| File | Current State | Matches Requirements |
|------|---------------|---------------------|
| `.github/workflows/build.yml` (NumPy job, line 294-305) | Uses `python: *python_3_12_up` anchor → `["3.14", "3.13", "3.12"]` (3.11 dropped) | ✅ Yes |
| `.github/workflows/build.yml` (other jobs) | Use `*python_3_10_up` / `*python_3_11_up` anchors (include 3.11) | ✅ Yes |
| `.github/workflows/build.yml` (Deploy job, line 557) | `if: ${{ always() && github.repository_owner == 'eeems' && github.event_name != 'pull_request' }}` — runs on all main-repo events except PRs, skips forks | ✅ Yes |
| `build/action.yml` | Full choice list including 3.11 (reverted per "undo" request) | ✅ Yes |
| `mirror/action.yaml:38` | `python-version: "3.11"` (unchanged per request) | ✅ Yes |
| `.gitignore` | Ignores `*.tgz`, `.agentty/`, `main_*.yml`, `main_*.yaml` | ✅ Yes |

## Remaining Step

### Step 1: Force Push Clean History
The artifact commit (`2b2489480c3688245517d727b087d816f3a415b3`) containing temporary build outputs (`.tgz` wheels, `.toml` configs, `.txt` metadata, draft workflow YAMLs) must be removed from history.

**Worker action required:**
```bash
cd /var/home/pi/.local/state/sheepdog/Eeems/wheels/13/worktree
git push --force origin fix-ci-local-actions:fix-ci-local-actions
```

**Note:** The planner's sandbox cannot access the git repo (`.git` points to an inaccessible worktree path at `/var/home/pi/.local/share/sheepdog/Eeems/wheels/repo/worktrees/worktree`). However, the worker environment should have proper git access and be able to execute the force push directly.

## Testing / Verification
Once force-pushed, GitHub Actions will run automatically on the branch and verify:
- NumPy builds run only on Python 3.14, 3.13, 3.12 (no 3.11)
- All other packages (crcmod, zstandard, pynacl, etc.) still build on 3.11+
- Deploy job runs on push to main, workflow_dispatch, schedule — but **skips** on pull_request events
- Deploy job skips forks (only runs when `github.repository_owner == 'eeems'`)
- No regressions in wheel building or artifact upload

## Risks / Unknowns
- **Force push permissions**: The worker must have write access to the repository. If the push fails due to permissions, the repository owner may need to push locally.
- **Git worktree setup**: If the worker also cannot access the git repo, the `.git` file may need to be re-initialized or the worktree re-created.
- **CI verification**: No automated verifier verdict is available (returns `no_available_model` / `invalid_response`). Manual inspection of the GitHub Actions run is required after push.

## Expected Outcome
After force push and CI completion:
- Clean git history without artifact commits
- NumPy builds target 3.12+ only
- All other packages target 3.10+
- Deploy job correctly gated on main-repo non-PR events
- PR ready for merge