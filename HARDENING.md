<!-- markdownlint-disable -->

# Hardening Report: mpi4py--setup-mpi/v1.4.4

> This file was generated automatically by the hardening agent.

**Policy SHA:** `374911c4be584df24e6da57af2d990570ef0f859`

**Test Policy SHA:** `843adf9e4b8f85d0c08b27b9d0b09dd094b54702`

**Harden Agent Version:** `2`

Action **mpi4py--setup-mpi/v1.4.4** was hardened automatically. 3 finding(s) were identified and resolved across 1 iteration(s).

## Findings Fixed

### github-env-injection (severity: high)

Carried forward from this version's previous harden, which fixed it; fix it again unless the code no longer has the problem. In setup-mpi.sh, multiple inherited process environment variables are written to $GITHUB_ENV and $GITHUB_PATH without the required sanitization step (`printf '%s' ... | tr -d '\n\r'`). A calling workflow can set these env vars to values containing newlines, enabling injection of arbitrary environment variables:
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

### prior-fix (severity: high)

Carried forward from this version's previous harden, which fixed it; fix it again unless the code no longer has the problem. The previous harden changed action.yml without recording a finding for it, and this analysis reported nothing there. Re-apply its security changes unless the code no longer needs them. The previous patch:
```diff
diff --git a/action.yml b/action.yml
index 060d743..568f07a 100644
--- a/action.yml
+++ b/action.yml
@@ -1,4 +1,3 @@
----
 name: 'Setup MPI'
 description: 'Set up a specific MPI implementation.'
 author: 'Lisandro Dalcin'
@@ -6,7 +5,7 @@ inputs:
   mpi:
     description: "MPI implementation name."
     required: false
-    default: ''  # Linux/macOS: 'mpich', Windows: 'msmpi'
+    default: '' # Linux/macOS: 'mpich', Windows: 'msmpi'
 outputs:
   mpi:
     description: "The installed MPI implementation name."
@@ -14,6 +13,23 @@ outputs:
 runs:
   using: 'composite'
   steps:
     - id: setup-mpi
       run: ${GITHUB_ACTION_PATH}/setup-mpi.sh "$MPI"
       shell: bash

```

Locations:

- `action.yml`

### prior-fix (severity: high)

Carried forward from this version's previous harden, which fixed it; fix it again unless the code no longer has the problem. The previous harden changed setup-mpi.sh without recording a finding for it, and this analysis reported nothing there. Re-apply its security changes unless the code no longer needs them. The previous patch:
```diff
diff --git a/setup-mpi.sh b/setup-mpi.sh
index f948ecf..7606ffd 100755
--- a/setup-mpi.sh
+++ b/setup-mpi.sh
@@ -71,12 +71,13 @@ setup-env-intel-oneapi () {
     set +u
     source /opt/intel/oneapi/setvars.sh
     set -u
-    echo "${I_MPI_ROOT}/bin" >> "$GITHUB_PATH"
-    echo "ONEAPI_ROOT=${ONEAPI_ROOT}" >> "$GITHUB_ENV"
-    echo "I_MPI_ROOT=${I_MPI_ROOT}" >> "$GITHUB_ENV"
-    echo "FI_PROVIDER_PATH=${FI_PROVIDER_PATH}" >> "$GITHUB_ENV"
-    echo "LD_LIBRARY_PATH=${LD_LIBRARY_PATH}" >> "$GITHUB_ENV"
-    echo "PKG_CONFIG_PATH=${PKG_CONFIG_PATH}" >> "$GITHUB_ENV"
+    printf '%s' "${I_MPI_ROOT}/bin" | tr -d '\n\r' >> "$GITHUB_PATH"
+    printf '\n' >> "$GITHUB_PATH"
+    printf 'ONEAPI_ROOT=%s\n' "$(printf '%s' "${ONEAPI_ROOT}" | tr -d '\n\r')" >> "$GITHUB_ENV"
+    printf 'I_MPI_ROOT=%s\n' "$(printf '%s' "${I_MPI_ROOT}" | tr -d '\n\r')" >> "$GITHUB_ENV"
+    printf 'FI_PROVIDER_PATH=%s\n' "$(printf '%s' "${FI_PROVIDER_PATH}" | tr -d '\n\r')" >> "$GITHUB_ENV"
+    printf 'LD_LIBRARY_PATH=%s\n' "$(printf '%s' "${LD_LIBRARY_PATH}" | tr -d '\n\r')" >> "$GITHUB_ENV"
+    printf 'PKG_CONFIG_PATH=%s\n' "$(printf '%s' "${PKG_CONFIG_PATH}" | tr -d '\n\r')" >> "$GITHUB_ENV"
 }
 
 setup-win-intel-oneapi-mpi () {
@@ -121,11 +122,13 @@ setup-win-intel-oneapi-mpi-env () {
     mpibindir="${I_MPI_ROOT}\bin"
     ofibindir="${I_MPI_ROOT}\opt\mpi\libfabric\bin"
 
-    echo "ONEAPI_ROOT=${ONEAPI_ROOT}" >> "$GITHUB_ENV"
-    echo "I_MPI_ROOT=${I_MPI_ROOT}" >> "$GITHUB_ENV"
-    echo "I_MPI_OFI_LIBRARY_INTERNAL=${I_MPI_OFI_LIBRARY_INTERNAL}" >> "$GITHUB_ENV"
-    echo "${mpibindir}" >> "$GITHUB_PATH"
-    echo "${ofibindir}" >> "$GITHUB_PATH"
+    printf 'ONEAPI_ROOT=%s\n' "$(printf '%s' "${ONEAPI_ROOT}" | tr -d '\n\r')" >> "$GITHUB_ENV"
+    printf 'I_MPI_ROOT=%s\n' "$(printf '%s' "${I_MPI_ROOT}" | tr -d '\n\r')" >> "$GITHUB_ENV"
+    printf 'I_MPI_OFI_LIBRARY_INTERNAL=%s\n' "$(printf '%s' "${I_MPI_OFI_LIBRARY_INTERNAL}" | tr -d '\n\r')" >> "$GITHUB_ENV"
+    printf '%s' "${mpibindir}" | tr -d '\n\r' >> "$GITHUB_PATH"
+    printf '\n' >> "$GITHUB_PATH"
+    printf '%s' "${ofibindir}" | tr -d '\n\r' >> "$GITHUB_PATH"
+    printf '\n' >> "$GITHUB_PATH"
 
     PATH="$(cygpath -u "${mpibindir}"):$PATH"
     PATH="$(cygpath -u "${ofibindir}"):$PATH"
@@ -208,7 +211,7 @@ case "$(uname)" in
         ;;
 esac
 
-echo "mpi=${MPI}" >> "$GITHUB_OUTPUT"
+printf 'mpi=%s\n' "$(printf '%s' "${MPI}" | tr -d '\n\r')" >> "$GITHUB_OUTPUT"
 
 case "$MPI" in
     mpich)

```

Locations:

- `setup-mpi.sh`

## Iteration Notes

### Iteration 1

**Fixes applied:** github-env-injection, prior-fix

**Notes:**

I fixed the newline-injection finding in setup-mpi.sh again. Every value the script writes to GITHUB_ENV, GITHUB_PATH and GITHUB_OUTPUT now goes through printf '%s' "$VAR" | tr -d '\n\r' first. That covers the Intel oneAPI variables on Linux, the Intel MPI variables and bin directories on Windows, and the final mpi= output. The values are plain variables rather than command output, so the sanitizing step can't hide a failed command. I didn't change action.yml: it already passes inputs.mpi to the script through env (MPI) and quotes it as "$MPI" in run, which is the security part of the earlier patch. The rest of that earlier patch only removed the leading '---' line and changed comment spacing, so I left those out. Nothing was run or tested.

