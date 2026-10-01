# PR #13 Fix Plan

## Key Findings (from worktree inspection)

### 1. **Deploy Job Issues in `.github/workflows/build.yml`**
- **Line 549**: `runs-on: self-hosted` → should be `ubuntu-latest` (all other 12 jobs use `ubuntu-latest`)
- **Line 555**: `needs:` list missing `fuse-python` job dependency (the `fuse-python` job exists at line 340)
- **Line 548**: `if: ${{ always() }}` — this runs Deploy even if other jobs fail; consider if this should be `if: github.event_name == 'release'` to match environment protection rules

### 2. **Job Inventory**
Total build jobs: **12** (excluding commented `nuitka`)
| Job ID | Line | In Deploy `needs:` |
|--------|------|-------------------|
| standard | 12 | ✅ |
| protobuf | 55 | ✅ |
| cffi | 95 | ✅ |
| indexed-gzip | 166 | ✅ |
| pillow | 246 | ✅ |
| numpy | 294 | ✅ |
| **fuse-python** | **340** | ❌ **MISSING** |
| wxpython | 378 | ✅ |
| bcrypt | 429 | ✅ |
| cryptography | 488 | ✅ |
| arm-3_11-up | 220 | ✅ |
| mirror (Deploy) | 546 | N/A |

### 3. **`mirror/action.yaml` line 41**
- Already has the correct path: `python -u "${{ github.action_path }}/create_dirs.py"` — NO FIX NEEDED

### 4. **Pre-existing Bugs (NOT in scope for this PR)**
- numpy fails on `debian-armv7l` for ALL Python versions on `main` (3.11, 3.12, 3.13, 3.14)
- This is a separate issue; dropping 3.11 from matrix fixes nothing

---

## Action Plan

### Immediate Fixes (this PR)

1. **Fix `.github/workflows/build.yml` Deploy job (lines 548-555)**
   ```yaml
   mirror:
     name: Deploy
     environment: wheels.eeems.codes
     runs-on: ubuntu-latest          # CHANGE: self-hosted → ubuntu-latest
     needs:
       - arm-3_11-up
       - bcrypt
       - cffi
       - cryptography
       - fuse-python                 # ADD: missing dependency (job at line 340)
       - indexed-gzip
       # - nuitka
       - numpy
       - pillow
       - protobuf
       - standard
       - wxpython
     if: ${{ always() }}             # REVIEW: consider github.event_name == 'release'
   ```

### Verification Steps

1. After fixes, trigger a test CI run (push a commit or use workflow_dispatch)
2. Check Deploy job logs for exit 255 root cause (rsync/ssh failure, not runner issue)
3. Verify all 12 jobs complete successfully and Deploy runs after them

### Out of Scope
- numpy armv7l failures (pre-existing, affects main)
- Any matrix changes for numpy
- Self-hosted runner infrastructure
- `mirror/action.yaml` (already correct)

---

## Assumptions
- The worktree represents the PR head branch state
- The goal is to make Deploy job work on `ubuntu-latest` with correct dependencies
- Environment `wheels.eeems.codes` has protection rules that may require `github.event_name == 'release'`