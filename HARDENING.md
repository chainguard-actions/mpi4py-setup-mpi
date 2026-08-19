<!-- markdownlint-disable -->

# Hardening Report: mpi4py--setup-mpi/v1.4.0

> This file was generated automatically by the hardening agent.

**Policy SHA:** `d636be7e43ef829af6e853da6b3c7566db9f72fe`

**Test Policy SHA:** `843adf9e4b8f85d0c08b27b9d0b09dd094b54702`

**Harden Agent Version:** `2`

Action **mpi4py--setup-mpi/v1.4.0** was hardened automatically. 3 finding(s) were identified and resolved across 2 iteration(s).

## Findings Fixed

### unpinned-uses (severity: high)

Multiple `uses:` references in .github/workflows/ci.yml use mutable tag/version refs instead of pinned 40-character SHA commit hashes, making the workflow vulnerable to supply-chain attacks if the referenced tag is moved or overwritten. Failing references: `step-security/harden-runner@v2` (appears 6 times) and `actions/checkout@v6` (appears 5 times).

Locations:

- `.github/workflows/ci.yml:65`
- `.github/workflows/ci.yml:69`

### script-injection (severity: high)

Multiple `run:` blocks in ci.yml directly interpolate GitHub Actions expressions inside shell commands (sub-rule a), allowing an attacker to inject arbitrary shell code. Examples include: `run: echo "${{ steps.setup-mpi.outputs.mpi }}"` (steps.*.outputs.* context), `run: test ${{ steps.setup1.outputs.mpi }} == mpich` (repeated across Linux, container, macOS, Windows jobs), and `run: ${{ !(contains(needs.*.result, 'failure')) }}` (needs.*.result context used directly as the shell command). All of these flow through YAML template substitution before the shell sees them.

Locations:

- `.github/workflows/ci.yml:79`
- `.github/workflows/ci.yml:107`
- `.github/workflows/ci.yml:113`
- `.github/workflows/ci.yml:119`
- `.github/workflows/ci.yml:148`
- `.github/workflows/ci.yml:154`
- `.github/workflows/ci.yml:160`
- `.github/workflows/ci.yml:196`
- `.github/workflows/ci.yml:202`
- `.github/workflows/ci.yml:208`
- `.github/workflows/ci.yml:244`
- `.github/workflows/ci.yml:250`
- `.github/workflows/ci.yml:281`
- `.github/workflows/ci.yml:282`
- `.github/workflows/ci.yml:299`

### github-env-injection (severity: high)

In setup-mpi.sh, the `setup-env-intel-oneapi` function writes inherited process environment variables sourced from Intel's `/opt/intel/oneapi/setvars.sh` — specifically `I_MPI_ROOT`, `ONEAPI_ROOT`, `FI_PROVIDER_PATH`, `LD_LIBRARY_PATH`, and `PKG_CONFIG_PATH` — directly to `$GITHUB_PATH` and `$GITHUB_ENV` without the required sanitization step (`printf '%s' "$VAR" | tr -d '\n\r'`). These variables are set by the calling environment (setvars.sh), not computed as literals in the same run block, so a compromised or attacker-influenced setvars.sh could inject newlines to add arbitrary entries to GITHUB_ENV. Similarly, `setup-win-intel-oneapi-mpi-env` writes `ONEAPI_ROOT`, `I_MPI_ROOT`, `I_MPI_OFI_LIBRARY_INTERNAL`, `mpibindir`, and `ofibindir` to `$GITHUB_ENV`/`$GITHUB_PATH` without sanitization. The final `echo "mpi=${MPI}" >> $GITHUB_OUTPUT` is lower risk since MPI is normalized earlier, but the env/path writes are clearly unsanitized.

Locations:

- `setup-mpi.sh:57`
- `setup-mpi.sh:58`
- `setup-mpi.sh:59`
- `setup-mpi.sh:60`
- `setup-mpi.sh:61`
- `setup-mpi.sh:62`
- `setup-mpi.sh:100`
- `setup-mpi.sh:101`
- `setup-mpi.sh:102`
- `setup-mpi.sh:103`
- `setup-mpi.sh:104`

## Iteration Notes

### Iteration 1

**Fixes applied:** unpinned-uses, script-injection, github-env-injection

**Notes:**

Fixed all three findings:

1. **unpinned-uses** (.github/workflows/ci.yml): Pinned `step-security/harden-runner@v2` to SHA `bf7454d06d71f1098171f2acdf0cd4708d7b5920` and `actions/checkout@v6` to SHA `df4cb1c069e1874edd31b4311f1884172cec0e10` across all 6+5 occurrences respectively, preserving the original tag as a comment.

2. **script-injection** (.github/workflows/ci.yml): Moved all `${{ steps.*.outputs.mpi }}` expressions from `run:` blocks into `env:` blocks (as `SETUP1_MPI`, `SETUP2_MPI`, etc.) and referenced them as plain shell variables. The dangerous `run: ${{ !(contains(needs.*.result, 'failure')) }}` pattern (which executed the boolean expression as a shell command) was replaced with a proper `env:`/`run:` pattern that checks `NEEDS_RESULT` env var and exits with code 1 on failure.

3. **github-env-injection** (setup-mpi.sh): Added `printf '%s' "$VAR" | tr -d '\n\r'` sanitization for all environment variables written to `$GITHUB_PATH` and `$GITHUB_ENV` in both `setup-env-intel-oneapi` (6 variables: `I_MPI_ROOT/bin`, `ONEAPI_ROOT`, `I_MPI_ROOT`, `FI_PROVIDER_PATH`, `LD_LIBRARY_PATH`, `PKG_CONFIG_PATH`) and `setup-win-intel-oneapi-mpi-env` (5 variables: `ONEAPI_ROOT`, `I_MPI_ROOT`, `I_MPI_OFI_LIBRARY_INTERNAL`, `mpibindir`, `ofibindir`). Also properly quoted the `$GITHUB_ENV` and `$GITHUB_PATH` variable references.

### Iteration 2

**Fixes applied:** github-env-injection

**Notes:**

Fixed setup-mpi.sh line 198: replaced `echo "mpi=${MPI}" >> $GITHUB_OUTPUT` with a sanitized version that first strips newlines/carriage-returns via `safe_MPI=$(printf '%s' "${MPI}" | tr -d '\n\r')` and then writes `echo "mpi=${safe_MPI}" >> "$GITHUB_OUTPUT"`. This is consistent with how all other GITHUB_ENV/GITHUB_PATH writes in the script already use sanitized `safe_*` variables. Also added quotes around `$GITHUB_OUTPUT` for proper shell hygiene.

