# Plan for PR #13: Fix NumPy build on debian-armv7l

## Problem Summary
The NumPy job in `.github/workflows/build.yml` builds on `debian-armv7l` with Python 3.12, 3.13, 3.14 (via the `python_3_12_up` anchor). The build is failing. The `eeems/nuitka-arm-builder:bullseye-{python}` images exist for Python 3.9-3.14, so Docker image availability is not the issue. The failure is likely due to NumPy 3.12+ requiring `gcc >= 11`, but the Debian Bullseye-based builder images only have `gcc 10`.

## Approach
1. Keep the NumPy matrix on `python_3_12_up` (3.14, 3.13, 3.12) — dropping 3.11 as requested
2. Keep `debian-armv7l` as the build target (per Eeems' request after confirming image availability)
3. Fix the NumPy build by installing a newer GCC (gcc-11 or gcc-12) from Debian backports inside the container

## Steps

### 1. Update the NumPy job setup script to install gcc-11 from bullseye-backports
**File:** `.github/workflows/build.yml` — NumPy job setup section (lines ~315-335)

Current setup installs:
```yaml
setup: |
  $SUDO apt-get install -y \
    gcc \
    g++ \
    gfortran \
    libopenblas-dev \
    liblapack-dev \
    pkg-config \
    python3-pip \
    python3-dev \
    cmake
```

**Change to:**
```yaml
setup: |
  $SUDO apt-get update
  $SUDO apt-get install -y \
    debian-archive-keyring
  $SUDO apt-get update
  $SUDO apt-get install -y -t bullseye-backports \
    gcc-11 \
    g++-11 \
    gfortran-11
  $SUDO apt-get install -y \
    libopenblas-dev \
    liblapack-dev \
    pkg-config \
    python3-pip \
    python3-dev \
    cmake
  export CC=gcc-11
  export CXX=g++-11
  export FC=gfortran-11
```

### 2. Verify the fix works
- Push the fix and monitor CI
- Ensure all 3 Python versions (3.12, 3.13, 3.14) build successfully on debian-armv7l

## Testing
- Trigger CI (push to branch or re-run workflow)
- Verify all 3 NumPy build matrix combinations pass:
  - numpy-py3.14-debian-armv7l
  - numpy-py3.13-debian-armv7l
  - numpy-py3.12-debian-armv7l
- Verify other package builds are unaffected

## Risks/Unknowns
- **Exact failure mode** — Not yet confirmed it's a gcc version issue. Need CI logs after fix attempt.
- **Debian backports availability** — Need to ensure `gcc-11` is available in bullseye-backports for armv7l architecture.
- **Docker image differences** — The `eeems/nuitka-arm-builder:bullseye-{python}` images for 3.12-3.14 may have different base packages than 3.11.
- **If gcc-11 from backports doesn't work** — May need to use Ubuntu toolchain PPA or compile GCC from source.

## Files to Modify
- `.github/workflows/build.yml` — NumPy job setup section (lines ~315-335)

---

**Note:** The other items (Deploy job condition, `build/action.yml` revert, `mirror/action.yaml` pin at 3.11, artifact cleanup) are already complete per the review history. The only remaining work is fixing the NumPy build on debian-armv7l.