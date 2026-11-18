<!-- markdownlint-disable -->

# Hardening Report: mpi4py--setup-mpi/v1.4.2

> This file was generated automatically by the hardening agent.

**Policy SHA:** `374911c4be584df24e6da57af2d990570ef0f859`

**Test Policy SHA:** `843adf9e4b8f85d0c08b27b9d0b09dd094b54702`

**Harden Agent Version:** `2`

Action **mpi4py--setup-mpi/v1.4.2** was hardened automatically. 2 finding(s) were identified and resolved across 1 iteration(s).

## Findings Fixed

### github-env-injection (severity: high)

Carried forward from this version's previous harden, which fixed it; fix it again unless the code no longer has the problem. In setup-mpi.sh, the composite action writes unsanitized values to special GitHub environment files without the required `printf '%s' ... | tr -d '\n\r'` sanitization step:

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

### prior-fix (severity: high)

Carried forward from this version's previous harden, which fixed it; fix it again unless the code no longer has the problem. The previous harden changed setup-mpi.sh without recording a finding for it, and this analysis reported nothing there. Re-apply its security changes unless the code no longer needs them. The previous patch:
```diff
diff --git a/setup-mpi.sh b/setup-mpi.sh
index 3ee560d..94d88ae 100755
--- a/setup-mpi.sh
+++ b/setup-mpi.sh
@@ -70,12 +70,13 @@ setup-env-intel-oneapi () {
     set +u
     source /opt/intel/oneapi/setvars.sh
     set -u
-    echo "${I_MPI_ROOT}/bin" >> $GITHUB_PATH
-    echo "ONEAPI_ROOT=${ONEAPI_ROOT}" >> $GITHUB_ENV
-    echo "I_MPI_ROOT=${I_MPI_ROOT}" >> $GITHUB_ENV
-    echo "FI_PROVIDER_PATH=${FI_PROVIDER_PATH}" >> $GITHUB_ENV
-    echo "LD_LIBRARY_PATH=${LD_LIBRARY_PATH}" >> $GITHUB_ENV
-    echo "PKG_CONFIG_PATH=${PKG_CONFIG_PATH}" >> $GITHUB_ENV
+    printf '%s' "${I_MPI_ROOT}/bin" | tr -d '\n\r' >> $GITHUB_PATH
+    printf '\n' >> $GITHUB_PATH
+    printf 'ONEAPI_ROOT=%s\n' "$(printf '%s' "${ONEAPI_ROOT}" | tr -d '\n\r')" >> $GITHUB_ENV
+    printf 'I_MPI_ROOT=%s\n' "$(printf '%s' "${I_MPI_ROOT}" | tr -d '\n\r')" >> $GITHUB_ENV
+    printf 'FI_PROVIDER_PATH=%s\n' "$(printf '%s' "${FI_PROVIDER_PATH}" | tr -d '\n\r')" >> $GITHUB_ENV
+    printf 'LD_LIBRARY_PATH=%s\n' "$(printf '%s' "${LD_LIBRARY_PATH}" | tr -d '\n\r')" >> $GITHUB_ENV
+    printf 'PKG_CONFIG_PATH=%s\n' "$(printf '%s' "${PKG_CONFIG_PATH}" | tr -d '\n\r')" >> $GITHUB_ENV
 }
 
 setup-win-intel-oneapi-mpi () {
@@ -118,11 +119,11 @@ setup-win-intel-oneapi-mpi-env () {
     mpibindir="${I_MPI_ROOT}\bin"
     ofibindir="${I_MPI_ROOT}\opt\mpi\libfabric\bin"
 
-    echo "ONEAPI_ROOT=${ONEAPI_ROOT}" >> $GITHUB_ENV
-    echo "I_MPI_ROOT=${I_MPI_ROOT}" >> $GITHUB_ENV
-    echo "I_MPI_OFI_LIBRARY_INTERNAL=${I_MPI_OFI_LIBRARY_INTERNAL}" >> $GITHUB_ENV
-    echo "${mpibindir}" >> $GITHUB_PATH
-    echo "${ofibindir}" >> $GITHUB_PATH
+    printf 'ONEAPI_ROOT=%s\n' "$(printf '%s' "${ONEAPI_ROOT}" | tr -d '\n\r')" >> $GITHUB_ENV
+    printf 'I_MPI_ROOT=%s\n' "$(printf '%s' "${I_MPI_ROOT}" | tr -d '\n\r')" >> $GITHUB_ENV
+    printf 'I_MPI_OFI_LIBRARY_INTERNAL=%s\n' "$(printf '%s' "${I_MPI_OFI_LIBRARY_INTERNAL}" | tr -d '\n\r')" >> $GITHUB_ENV
+    printf '%s\n' "$(printf '%s' "${mpibindir}" | tr -d '\n\r')" >> $GITHUB_PATH
+    printf '%s\n' "$(printf '%s' "${ofibindir}" | tr -d '\n\r')" >> $GITHUB_PATH
 
     export PATH="$(cygpath -u "${mpibindir}"):$PATH"
     export PATH="$(cygpath -u "${ofibindir}"):$PATH"
@@ -204,7 +205,7 @@ case $(uname) in
         ;;
 esac
 
-echo "mpi=${MPI}" >> $GITHUB_OUTPUT
+printf 'mpi=%s\n' "$(printf '%s' "${MPI}" | tr -d '\n\r')" >> $GITHUB_OUTPUT
 
 case $MPI in
     mpich)

```

Locations:

- `setup-mpi.sh`

## Iteration Notes

### Iteration 1

**Fixes applied:** github-env-injection, prior-fix

**Notes:**

I re-applied the previous harden's fix to setup-mpi.sh. Every write to $GITHUB_ENV, $GITHUB_PATH and $GITHUB_OUTPUT now strips newlines with printf '%s' ... | tr -d '\n\r' before writing. That covers the Linux oneAPI setup (setup-env-intel-oneapi), the Windows Intel MPI setup (setup-win-intel-oneapi-mpi-env) and the mpi= output line. I also quoted the target file paths. The values written are the same as before except that any newline or carriage return is now removed. The command substitutions only wrap printf and tr, so set -e still stops the script on a real failure. action.yml already passes inputs.mpi through env:, so it needed no change. Nothing was run to test these edits.

