<!-- markdownlint-disable -->

# Hardening Report: mpi4py--setup-mpi/v1.4.4

> This file was generated automatically by the hardening agent.

**Policy SHA:** `d636be7e43ef829af6e853da6b3c7566db9f72fe`

**Test Policy SHA:** `843adf9e4b8f85d0c08b27b9d0b09dd094b54702`

**Harden Agent Version:** `2`

Action **mpi4py--setup-mpi/v1.4.4** was hardened automatically. 3 finding(s) were identified and resolved across 1 iteration(s).

## Findings Fixed

### unpinned-uses (severity: high)

Multiple `uses:` references in .github/workflows/ci.yml use mutable tag-based refs instead of pinned 40-character SHA commit digests, making the workflow vulnerable to supply-chain attacks if the referenced action tags are moved. Failing references include: `actions/checkout@v7`, `actions/setup-python@v6`, `step-security/harden-runner@v2` (used in multiple jobs).

Locations:

- `.github/workflows/ci.yml:22`
- `.github/workflows/ci.yml:25`
- `.github/workflows/ci.yml:57`
- `.github/workflows/ci.yml:62`

### script-injection (severity: high)

Multiple `run:` blocks in ci.yml directly interpolate `${{ steps.*.outputs.mpi }}` (a `steps.*.outputs.*` context value) into shell commands without routing through an `env:` variable. This allows an attacker who can influence the step output to inject arbitrary shell commands. Offending lines include:
- `run: echo "${{ steps.setup-mpi.outputs.mpi }}"` (test job, sub-rule a)
- `run: test ${{ steps.setup1.outputs.mpi }} == mpich` and similar (Linux, container, macOS, Windows jobs, sub-rule a)
- `run: ${{ !(contains(needs.*.result, 'failure')) }}` in ci-status job — the entire shell command is a `${{ }}` expression (sub-rule a)

Locations:

- `.github/workflows/ci.yml:71`
- `.github/workflows/ci.yml:100`
- `.github/workflows/ci.yml:107`
- `.github/workflows/ci.yml:114`
- `.github/workflows/ci.yml:155`
- `.github/workflows/ci.yml:162`
- `.github/workflows/ci.yml:169`
- `.github/workflows/ci.yml:209`
- `.github/workflows/ci.yml:216`
- `.github/workflows/ci.yml:247`
- `.github/workflows/ci.yml:254`
- `.github/workflows/ci.yml:285`
- `.github/workflows/ci.yml:287`
- `.github/workflows/ci.yml:302`

### github-env-injection (severity: high)

In setup-mpi.sh, multiple inherited process environment variables are written to $GITHUB_ENV and $GITHUB_PATH without the required sanitization step (`printf '%s' ... | tr -d '\n\r'`). A calling workflow can set these env vars to values containing newlines, enabling injection of arbitrary environment variables:
1. `setup-env-intel-oneapi()`: writes `${I_MPI_ROOT}/bin` to $GITHUB_PATH and `ONEAPI_ROOT`, `I_MPI_ROOT`, `FI_PROVIDER_PATH`, `LD_LIBRARY_PATH`, `PKG_CONFIG_PATH` to $GITHUB_ENV — all sourced from the inherited environment after `source setvars.sh`.
2. `setup-win-intel-oneapi-mpi-env()`: writes `${ONEAPI_ROOT}`, `${I_MPI_ROOT}`, `${I_MPI_OFI_LIBRARY_INTERNAL}`, `${mpibindir}`, `${ofibindir}` to $GITHUB_ENV/$GITHUB_PATH without sanitization.
3. Final `echo "mpi=${MPI}" >> "$GITHUB_OUTPUT"`: writes `$MPI` (derived from `inputs.mpi` via `$1`) to $GITHUB_OUTPUT without sanitization.

Locations:

- `setup-mpi.sh:63`
- `setup-mpi.sh:64`
- `setup-mpi.sh:65`
- `setup-mpi.sh:66`
- `setup-mpi.sh:67`
- `setup-mpi.sh:68`
- `setup-mpi.sh:107`
- `setup-mpi.sh:108`
- `setup-mpi.sh:109`
- `setup-mpi.sh:110`
- `setup-mpi.sh:111`
- `setup-mpi.sh:196`

## Iteration Notes

### Iteration 1

**Fixes applied:** unpinned-uses, script-injection, github-env-injection

**Notes:**

Fixed all three findings:

1. unpinned-uses: Pinned all action references in .github/workflows/ci.yml to full 40-char SHAs: actions/checkout@3d3c42e5aac5ba805825da76410c181273ba90b1 (v7), actions/setup-python@ece7cb06caefa5fff74198d8649806c4678c61a1 (v6), step-security/harden-runner@05e31511f85b41b11d1cf0ef85d0992719546e2c (v2).

2. script-injection: Moved all ${{ steps.*.outputs.mpi }} expressions out of run: blocks into env: blocks, referencing them as $SETUP1_MPI, $SETUP2_MPI, etc. The ci-status job's run: ${{ !(contains(needs.*.result, 'failure')) }} was replaced with an env var NEEDS_RESULT and a proper shell conditional.

3. github-env-injection: In setup-mpi.sh, all writes to $GITHUB_ENV, $GITHUB_PATH, and $GITHUB_OUTPUT now use printf '%s' "$VAR" | tr -d '\n\r' to strip newlines before writing, preventing injection of arbitrary environment variables.

