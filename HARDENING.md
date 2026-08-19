<!-- markdownlint-disable -->

# Hardening Report: mpi4py--setup-mpi/v1.4.2

> This file was generated automatically by the hardening agent.

**Policy SHA:** `d636be7e43ef829af6e853da6b3c7566db9f72fe`

**Test Policy SHA:** `843adf9e4b8f85d0c08b27b9d0b09dd094b54702`

**Harden Agent Version:** `2`

Action **mpi4py--setup-mpi/v1.4.2** was hardened automatically. 3 finding(s) were identified and resolved across 1 iteration(s).

## Findings Fixed

### unpinned-uses (severity: high)

The workflow uses action references pinned to mutable tags rather than immutable 40-character SHA digests. `step-security/harden-runner@v2` and `actions/checkout@v6` appear in every job (test, Linux, container, macOS, Windows, ci-status). If the tag is moved to a different commit, the workflow will silently execute different code — a supply-chain attack vector.

Locations:

- `.github/workflows/ci.yml:63`
- `.github/workflows/ci.yml:67`

### script-injection (severity: high)

Multiple `run:` blocks directly interpolate `${{ ... }}` expressions into shell commands (sub-rule a). Examples: `run: echo "${{ steps.setup-mpi.outputs.mpi }}"` (test job), `run: test ${{ steps.setup1.outputs.mpi }} == mpich` (Linux/container/macOS/Windows jobs), and most critically `run: ${{ !(contains(needs.*.result, 'failure')) }}` in the ci-status job where the entire run command is a template expression. Any of these allow an attacker who can influence the step output or needs result to inject arbitrary shell commands.

Locations:

- `.github/workflows/ci.yml:73`
- `.github/workflows/ci.yml:155`
- `.github/workflows/ci.yml:185`
- `.github/workflows/ci.yml:220`
- `.github/workflows/ci.yml:250`
- `.github/workflows/ci.yml:270`

### github-env-injection (severity: high)

In setup-mpi.sh, the composite action writes unsanitized values to special GitHub environment files without the required `printf '%s' ... | tr -d '\n\r'` sanitization step:

1. `setup-env-intel-oneapi()` (lines ~67-72): writes inherited process env vars `ONEAPI_ROOT`, `I_MPI_ROOT`, `FI_PROVIDER_PATH`, `LD_LIBRARY_PATH`, `PKG_CONFIG_PATH` (populated by sourcing Intel's setvars.sh, but also inheritable from the calling workflow) directly to `$GITHUB_ENV` and `$GITHUB_PATH` without sanitization. A calling workflow could pre-set these vars with newline-containing values to inject additional environment entries.

2. `echo "mpi=${MPI}" >> $GITHUB_OUTPUT` (line ~175): writes a value derived from `inputs.mpi` (user-controlled) to `$GITHUB_OUTPUT`. Although the value is normalized via `tr '[:upper:]' '[:lower:]' | tr -d '-'`, the required sanitization pattern (`printf '%s' ... | tr -d '\n\r'`) is not applied before the write.

Locations:

- `setup-mpi.sh:67`
- `setup-mpi.sh:68`
- `setup-mpi.sh:69`
- `setup-mpi.sh:70`
- `setup-mpi.sh:71`
- `setup-mpi.sh:72`
- `setup-mpi.sh:175`

## Iteration Notes

### Iteration 1

**Fixes applied:** unpinned-uses, script-injection, github-env-injection

**Notes:**

Fixed all three findings in ci.yml and setup-mpi.sh:

1. unpinned-uses: Pinned step-security/harden-runner@v2 to SHA bf7454d06d71f1098171f2acdf0cd4708d7b5920 and actions/checkout@v6 to SHA d23441a48e516b6c34aea4fa41551a30e30af803 across all 6 jobs.

2. script-injection: Moved all ${{ steps.*.outputs.mpi }} expressions into env: blocks with named variables (SETUP_MPI_OUTPUT, SETUP1_MPI, etc.) referenced as plain shell variables in run: blocks. The dangerous ci-status job's `run: ${{ !(contains(needs.*.result, 'failure')) }}` was replaced with a proper shell script that reads needs results via an ALL_RESULTS env var and uses grep to detect failures.

3. github-env-injection: Fixed setup-env-intel-oneapi() and setup-win-intel-oneapi-mpi-env() in setup-mpi.sh to sanitize all values written to $GITHUB_ENV, $GITHUB_PATH, and $GITHUB_OUTPUT using `printf '%s' ... | tr -d '\n\r'` before writing.

