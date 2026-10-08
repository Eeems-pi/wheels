# Implementation Plan — PR Eeems/wheels#13: Fix NumPy build on debian-armv7l

## 1. Problem Summary
NumPy builds fail on `debian-armv7l` because `build/action.py` contains a special case for `debian-armv7l` that runs `sudo apt-get install -y gcc-11 g++-11` inside the container. This command:
- Runs **without** `apt-get update` (packages may be stale/unavailable)
- Assumes `sudo` exists and works (container may run as root)
- Is **redundant** because the workflow's numpy job `setup` script already installs gcc-11, g++-11, gfortran-11, libopenblas-dev, liblapack-dev, pkg-config, python3-pip, python3-dev, and cmake — with proper sudo detection, `DEBIAN_FRONTEND=noninteractive`, and `apt-get update`

The reviewer confirmed: "revert back to `debian-armv7l` and fix the numpy build itself" — the target stays `debian-armv7l`; the fix is to let the workflow's `setup` script handle all dependencies (matching the fuse-python and pillow job patterns).

## 2. Approach
Remove the `debian-armv7l` special case from `build/action.py`. The workflow's numpy job `setup` script (already correct) will handle all necessary dependencies. This matches the pattern used by fuse-python and pillow jobs.

## 3. Steps

### Step 1 — Remove debian-armv7l special case from build/action.py
- **File**: `/var/home/pi/.local/state/sheepdog/Eeems/wheels/13/worktree/build/action.py`
- **Location**: Lines 67–70 (the `if args.build_on == "debian-armv7l":` block)
- **Change**: Delete these lines entirely:
  ```python
  if args.build_on == "debian-armv7l":
      # Fix gcc compiler for numpy on Debian ARMv7L
      # Ensure gcc-11 and g++ are installed for numpy build
      script.append("sudo apt-get install -y gcc-11 g++-11")
  ```
- **Rationale**: The workflow's numpy job `setup` script already installs all required packages (including gcc-11/g++-11) with proper sudo detection, apt update, and non-interactive frontend. The action.py addition is redundant and broken (no apt update, assumes sudo).

### Step 2 — Verify workflow numpy job setup is correct (no changes needed)
- **File**: `/var/home/pi/.local/state/sheepdog/Eeems/wheels/13/worktree/.github/workflows/build.yml`
- **Location**: Lines 317–334 (numpy job `setup:` block)
- **Current state (correct)**:
  ```yaml
  setup: |
    SUDO=''
    if [ "$EUID" -ne 0 ]; then
      SUDO=sudo
    fi
    export DEBIAN_FRONTEND="noninteractive"
    $SUDO apt-get -y update
    $SUDO apt-get install -y -t \
      bullseye-backports \
      gcc-11 \
      g++-11 \
      gfortran-11 \
      libopenblas-dev \
      liblapack-dev \
      pkg-config \
      python3-pip \
      python3-dev \
      cmake
  ```
- **Verification**: Confirm this matches the reviewer's suggested fix and the fuse-python job pattern. No changes required.

### Step 3 — Push and verify CI
- Push the change to the PR branch
- Watch the numpy job for `debian-armv7l` × `3.12, 3.13, 3.14` matrix entries
- All 3 should complete successfully (green check)

## 4. Testing

| Step | Verification Method | Expected Outcome |
|------|---------------------|------------------|
| 1 | `grep -n "debian-armv7l" build/action.py` | No matches (special case removed) |
| 2 | `git diff build/action.py` | Only the 7-line removal shown above |
| 3 | Read build.yml lines 317–334 | Setup script has SUDO detection, apt update, DEBIAN_FRONTEND, all required packages |
| 4 | CI: Push and watch numpy job for `debian-armv7l` × `3.12, 3.13, 3.14` | All 3 matrix entries complete successfully (green check) |

## 5. Risks / Unknowns

- **Other packages on debian-armv7l**: The pillow job (python 3.11) also uses `debian-armv7l`. Its setup script installs dev libraries but not gcc-11/g++-11. The base image `eeems/nuitka-arm-builder:bullseye-3.11` likely includes a suitable compiler. If pillow fails, its setup script can be extended — but the reviewer's focus is numpy.
- **Container user**: The nuitka-arm-builder images may run as root or non-root. The workflow's setup script handles both via sudo detection; the removed action.py code did not.
- **No CI log access**: Cannot verify the exact error from the failed build; the fix is based on code inspection, reviewer direction, and consistency with other jobs.
- **Force push**: The worktree is clean; a normal push is expected (not force push) unless the branch was rebased.