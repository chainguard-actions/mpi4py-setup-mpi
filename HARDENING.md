<!-- markdownlint-disable -->

# Hardening Report: mpi4py--setup-mpi/v1.4.2

> This file was generated automatically by the hardening agent.

**Policy SHA:** `d636be7e43ef829af6e853da6b3c7566db9f72fe`

**Test Policy SHA:** `843adf9e4b8f85d0c08b27b9d0b09dd094b54702`

**Harden Agent Version:** `1`

Action **mpi4py--setup-mpi/v1.4.2** was hardened automatically. 2 finding(s) were identified and resolved across 1 iteration(s).

## Findings Fixed

### github-env-injection (severity: high)

In setup-mpi.sh, the `setup-env-intel-oneapi` function sources Intel's setvars.sh and then writes the resulting inherited environment variables (I_MPI_ROOT, ONEAPI_ROOT, FI_PROVIDER_PATH, LD_LIBRARY_PATH, PKG_CONFIG_PATH) directly to $GITHUB_PATH and $GITHUB_ENV without the required sanitization step (`printf '%s' "$VAR" | tr -d '\n\r'`). These variables are set by an external script and are workflow-controllable, making them untrusted for the purposes of this check. A malicious value containing newlines could inject arbitrary environment variables or PATH entries into subsequent steps.

Locations:

- `setup-mpi.sh:67`
- `setup-mpi.sh:68`
- `setup-mpi.sh:69`
- `setup-mpi.sh:70`
- `setup-mpi.sh:71`

### github-env-injection (severity: high)

In setup-mpi.sh, the final `echo "mpi=${MPI}" >> $GITHUB_OUTPUT` writes the MPI variable to $GITHUB_OUTPUT without the required sanitization step. MPI is derived from the user-supplied `inputs.mpi` value (passed as $1 from action.yml), and while it has been lowercased and had dashes removed, it has not been sanitized for newline characters (`tr -d '\n\r'`) before being written to the special environment file. An attacker could supply a value containing newlines to inject additional output variables.

Locations:

- `setup-mpi.sh:183`

## Iteration Notes

### Iteration 1

**Fixes applied:** github-env-injection

**Notes:**

Fixed two github-env-injection findings in setup-mpi.sh:
1. In setup-env-intel-oneapi(): Added sanitization for all 5 Intel OneAPI environment variables (I_MPI_ROOT, ONEAPI_ROOT, FI_PROVIDER_PATH, LD_LIBRARY_PATH, PKG_CONFIG_PATH) using `printf '%s' "${VAR}" | tr -d '\n\r'` before writing to $GITHUB_PATH and $GITHUB_ENV. Also quoted the special file variables.
2. At the final `echo "mpi=..." >> $GITHUB_OUTPUT` line: Added sanitization of the MPI variable using `printf '%s' "${MPI}" | tr -d '\n\r'` before writing to $GITHUB_OUTPUT. Also quoted the special file variable.

