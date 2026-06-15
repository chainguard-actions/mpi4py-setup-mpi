<!-- markdownlint-disable -->

# Hardening Report: mpi4py--setup-mpi/v1.4.3

> This file was generated automatically by the hardening agent.

**Policy SHA:** `d636be7e43ef829af6e853da6b3c7566db9f72fe`

**Test Policy SHA:** `843adf9e4b8f85d0c08b27b9d0b09dd094b54702`

**Harden Agent Version:** `1`

Action **mpi4py--setup-mpi/v1.4.3** was hardened automatically. 2 finding(s) were identified and resolved across 1 iteration(s).

## Findings Fixed

### github-env-injection (severity: high)

In setup-mpi.sh, the `MPI` variable is derived from `inputs.mpi` (passed as positional argument `$1` from action.yml, where `MPI: ${{ inputs.mpi }}`). It is processed through `tr '[:upper:]' '[:lower:]' | tr -d '-'` which strips dashes and lowercases but does NOT strip newlines (\n/\r). The value is then written directly to $GITHUB_OUTPUT as `echo "mpi=${MPI}" >> "$GITHUB_OUTPUT"` without the required sanitization step (`printf '%s' "$MPI" | tr -d '\n\r'`). An attacker-controlled `inputs.mpi` value containing embedded newlines could inject additional key=value pairs into $GITHUB_OUTPUT.

Locations:

- `setup-mpi.sh:155`

### github-env-injection (severity: high)

In setup-mpi.sh, the `setup-env-intel-oneapi` function sources `/opt/intel/oneapi/setvars.sh` and then writes inherited process environment variables (`ONEAPI_ROOT`, `I_MPI_ROOT`, `FI_PROVIDER_PATH`, `LD_LIBRARY_PATH`, `PKG_CONFIG_PATH`) directly to $GITHUB_ENV and $GITHUB_PATH without sanitization. As a composite action, these env vars are inherited from the calling workflow and must be treated as untrusted. No `printf '%s' ... | tr -d '\n\r'` sanitization is applied before any of these writes.

Locations:

- `setup-mpi.sh:63`
- `setup-mpi.sh:64`
- `setup-mpi.sh:65`
- `setup-mpi.sh:66`
- `setup-mpi.sh:67`
- `setup-mpi.sh:68`

## Iteration Notes

### Iteration 1

**Fixes applied:** github-env-injection

**Notes:**

Fixed two github-env-injection findings in setup-mpi.sh:

1. Line ~155 (GITHUB_OUTPUT write): Added `safe_mpi=$(printf '%s' "${MPI}" | tr -d '\n\r')` and used `safe_mpi` in the `echo "mpi=${safe_mpi}" >> "$GITHUB_OUTPUT"` line to strip embedded newlines from the attacker-controlled `inputs.mpi` value.

2. Lines 63-68 (setup-env-intel-oneapi function): Added sanitization for all six environment variables written to $GITHUB_ENV and $GITHUB_PATH (`I_MPI_ROOT/bin`, `ONEAPI_ROOT`, `I_MPI_ROOT`, `FI_PROVIDER_PATH`, `LD_LIBRARY_PATH`, `PKG_CONFIG_PATH`) using `printf '%s' "${VAR}" | tr -d '\n\r'` before each write, preventing newline injection from environment variables sourced from `/opt/intel/oneapi/setvars.sh`.

