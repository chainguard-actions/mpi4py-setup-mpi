<!-- markdownlint-disable -->

# Hardening Report: mpi4py--setup-mpi/v1.4.1

> This file was generated automatically by the hardening agent.

**Policy SHA:** `d636be7e43ef829af6e853da6b3c7566db9f72fe`

**Test Policy SHA:** `843adf9e4b8f85d0c08b27b9d0b09dd094b54702`

**Harden Agent Version:** `2`

Action **mpi4py--setup-mpi/v1.4.1** was hardened automatically. 3 finding(s) were identified and resolved across 2 iteration(s).

## Findings Fixed

### unpinned-uses (severity: high)

Multiple `uses:` references in .github/workflows/ci.yml are pinned to mutable tags rather than full 40-character commit SHAs, making the workflow vulnerable to supply-chain attacks if the tag is moved. Failing references include: `step-security/harden-runner@v2` (appears 6 times) and `actions/checkout@v6` (appears 5 times).

Locations:

- `.github/workflows/ci.yml:56`
- `.github/workflows/ci.yml:60`
- `.github/workflows/ci.yml:113`
- `.github/workflows/ci.yml:117`
- `.github/workflows/ci.yml:148`
- `.github/workflows/ci.yml:152`
- `.github/workflows/ci.yml:178`
- `.github/workflows/ci.yml:182`
- `.github/workflows/ci.yml:213`
- `.github/workflows/ci.yml:217`
- `.github/workflows/ci.yml:237`

### script-injection (severity: high)

Multiple `run:` blocks in ci.yml directly interpolate GitHub Actions expressions (`${{ ... }}`) inside shell commands, violating rule (a). This allows expression values to be interpreted as shell code before the shell ever sees them.

1. `run: echo "${{ steps.setup-mpi.outputs.mpi }}"` — the step output is interpolated directly into the shell command string.
2. `run: test ${{ steps.setup1.outputs.mpi }} == mpich` (and similar patterns for setup2/setup3/setup4 across Linux, container, macOS, and Windows jobs) — step outputs interpolated directly.
3. `run: ${{ !(contains(needs.*.result, 'failure')) }}` — an expression is used as the entire shell command in the ci-status job, which is a direct script injection vector.

Locations:

- `.github/workflows/ci.yml:63`
- `.github/workflows/ci.yml:120`
- `.github/workflows/ci.yml:124`
- `.github/workflows/ci.yml:128`
- `.github/workflows/ci.yml:155`
- `.github/workflows/ci.yml:159`
- `.github/workflows/ci.yml:163`
- `.github/workflows/ci.yml:185`
- `.github/workflows/ci.yml:189`
- `.github/workflows/ci.yml:220`
- `.github/workflows/ci.yml:224`
- `.github/workflows/ci.yml:238`
- `.github/workflows/ci.yml:239`

### github-env-injection (severity: high)

In setup-mpi.sh (called by action.yml), the `setup-env-intel-oneapi` and `setup-win-intel-oneapi-mpi-env` functions write inherited process environment variables to `$GITHUB_ENV` and `$GITHUB_PATH` without the required sanitization step (`printf '%s' ... | tr -d '\n\r'`). These variables (`ONEAPI_ROOT`, `I_MPI_ROOT`, `FI_PROVIDER_PATH`, `LD_LIBRARY_PATH`, `PKG_CONFIG_PATH`, `mpibindir`, `ofibindir`) are sourced from Intel OneAPI's `setvars.sh` or set from hardcoded-looking paths that could be influenced by the calling workflow environment. Writing unsanitized multi-line-capable values to `$GITHUB_ENV`/`$GITHUB_PATH` allows newline injection that can set arbitrary environment variables or add arbitrary entries to PATH.

Failing lines (setup-env-intel-oneapi):
  `echo "${I_MPI_ROOT}/bin" >> $GITHUB_PATH`
  `echo "ONEAPI_ROOT=${ONEAPI_ROOT}" >> $GITHUB_ENV`
  `echo "I_MPI_ROOT=${I_MPI_ROOT}" >> $GITHUB_ENV`
  `echo "FI_PROVIDER_PATH=${FI_PROVIDER_PATH}" >> $GITHUB_ENV`
  `echo "LD_LIBRARY_PATH=${LD_LIBRARY_PATH}" >> $GITHUB_ENV`
  `echo "PKG_CONFIG_PATH=${PKG_CONFIG_PATH}" >> $GITHUB_ENV`

Failing lines (setup-win-intel-oneapi-mpi-env):
  `echo "ONEAPI_ROOT=${ONEAPI_ROOT}" >> $GITHUB_ENV`
  `echo "I_MPI_ROOT=${I_MPI_ROOT}" >> $GITHUB_ENV`
  `echo "I_MPI_OFI_LIBRARY_INTERNAL=${I_MPI_OFI_LIBRARY_INTERNAL}" >> $GITHUB_ENV`
  `echo "${mpibindir}" >> $GITHUB_PATH`
  `echo "${ofibindir}" >> $GITHUB_PATH`

Locations:

- `setup-mpi.sh:62`
- `setup-mpi.sh:63`
- `setup-mpi.sh:64`
- `setup-mpi.sh:65`
- `setup-mpi.sh:66`
- `setup-mpi.sh:67`
- `setup-mpi.sh:103`
- `setup-mpi.sh:104`
- `setup-mpi.sh:105`
- `setup-mpi.sh:106`
- `setup-mpi.sh:107`

## Iteration Notes

### Iteration 1

**Fixes applied:** unpinned-uses, script-injection, github-env-injection

**Notes:**

Fixed all three findings in ci.yml and setup-mpi.sh:

1. unpinned-uses: Pinned all 6 occurrences of step-security/harden-runner@v2 to SHA bf7454d06d71f1098171f2acdf0cd4708d7b5920 and all 5 occurrences of actions/checkout@v6 to SHA d23441a48e516b6c34aea4fa41551a30e30af803, with original tags preserved as comments.

2. script-injection: Moved all ${{ steps.setupN.outputs.mpi }} expressions out of run: blocks into env: blocks (SETUP1_MPI, SETUP2_MPI, etc.), and replaced the dangerous `run: ${{ !(contains(needs.*.result, 'failure')) }}` in ci-status with a proper shell conditional using an env var CI_SUCCESS.

3. github-env-injection: In setup-mpi.sh, replaced all bare `echo "VAR=${VALUE}" >> $GITHUB_ENV` and `echo "${PATH_VALUE}" >> $GITHUB_PATH` calls in setup-env-intel-oneapi and setup-win-intel-oneapi-mpi-env with sanitized `printf '%s' ... | tr -d '\n\r'` patterns to prevent newline injection attacks.

### Iteration 2

**Fixes applied:** script-injection, github-env-injection

**Notes:**

Fixed in setup-mpi.sh: (1) script-injection - quoted all unquoted $MPI expansions: brew list/link/install now use "$MPI", and the if-test now uses [ "$MPI" == openmpi ]; the pwsh invocation was already safe as ${MPI} was inside a double-quoted string. (2) github-env-injection - replaced `echo "mpi=${MPI}" >> $GITHUB_OUTPUT` with a two-step sanitized write: `safe_mpi=$(printf '%s' "$MPI" | tr -d '\n\r')` followed by `printf 'mpi=%s\n' "$safe_mpi" >> "$GITHUB_OUTPUT"` to strip any embedded newlines before writing to the special environment file.

