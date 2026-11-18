<!-- markdownlint-disable -->

# Hardening Report: mpi4py--setup-mpi/v1.4.3

> This file was generated automatically by the hardening agent.

**Policy SHA:** `374911c4be584df24e6da57af2d990570ef0f859`

**Test Policy SHA:** `843adf9e4b8f85d0c08b27b9d0b09dd094b54702`

**Harden Agent Version:** `2`

Action **mpi4py--setup-mpi/v1.4.3** was hardened automatically. 3 finding(s) were identified and resolved across 1 iteration(s).

## Findings Fixed

### github-env-injection (severity: high)

setup-mpi.sh setup-env-intel-oneapi() (run by the composite step in action.yml) writes values to $GITHUB_ENV and $GITHUB_PATH without first stripping newlines. The values come from the environment the action inherits: LD_LIBRARY_PATH, PKG_CONFIG_PATH, FI_PROVIDER_PATH, ONEAPI_ROOT and I_MPI_ROOT. Intel's setvars.sh adds to these but keeps whatever the calling workflow set. Example lines: `echo "LD_LIBRARY_PATH=${LD_LIBRARY_PATH}" >> "$GITHUB_ENV"`, `echo "PKG_CONFIG_PATH=${PKG_CONFIG_PATH}" >> "$GITHUB_ENV"`, `echo "${I_MPI_ROOT}/bin" >> "$GITHUB_PATH"`. If one of these values contains a newline, it can add extra environment or PATH entries. Fix: run each value through `printf '%s' "$VAR" | tr -d '\n\r'` before writing it.

Locations:

- `setup-mpi.sh:76`
- `setup-mpi.sh:77`
- `setup-mpi.sh:80`
- `setup-mpi.sh:81`

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
index 9ebc30a..ff940fe 100755
--- a/setup-mpi.sh
+++ b/setup-mpi.sh
@@ -71,12 +71,18 @@ setup-env-intel-oneapi () {
     set +u
     source /opt/intel/oneapi/setvars.sh
     set -u
-    echo "${I_MPI_ROOT}/bin" >> "$GITHUB_PATH"
-    echo "ONEAPI_ROOT=${ONEAPI_ROOT}" >> "$GITHUB_ENV"
-    echo "I_MPI_ROOT=${I_MPI_ROOT}" >> "$GITHUB_ENV"
-    echo "FI_PROVIDER_PATH=${FI_PROVIDER_PATH}" >> "$GITHUB_ENV"
-    echo "LD_LIBRARY_PATH=${LD_LIBRARY_PATH}" >> "$GITHUB_ENV"
-    echo "PKG_CONFIG_PATH=${PKG_CONFIG_PATH}" >> "$GITHUB_ENV"
+    safe_impi_root_bin=$(printf '%s' "${I_MPI_ROOT}/bin" | tr -d '\n\r')
+    safe_oneapi_root=$(printf '%s' "${ONEAPI_ROOT}" | tr -d '\n\r')
+    safe_impi_root=$(printf '%s' "${I_MPI_ROOT}" | tr -d '\n\r')
+    safe_fi_provider_path=$(printf '%s' "${FI_PROVIDER_PATH}" | tr -d '\n\r')
+    safe_ld_library_path=$(printf '%s' "${LD_LIBRARY_PATH}" | tr -d '\n\r')
+    safe_pkg_config_path=$(printf '%s' "${PKG_CONFIG_PATH}" | tr -d '\n\r')
+    echo "${safe_impi_root_bin}" >> "$GITHUB_PATH"
+    echo "ONEAPI_ROOT=${safe_oneapi_root}" >> "$GITHUB_ENV"
+    echo "I_MPI_ROOT=${safe_impi_root}" >> "$GITHUB_ENV"
+    echo "FI_PROVIDER_PATH=${safe_fi_provider_path}" >> "$GITHUB_ENV"
+    echo "LD_LIBRARY_PATH=${safe_ld_library_path}" >> "$GITHUB_ENV"
+    echo "PKG_CONFIG_PATH=${safe_pkg_config_path}" >> "$GITHUB_ENV"
 }
 
 setup-win-intel-oneapi-mpi () {
@@ -207,7 +213,8 @@ case "$(uname)" in
         ;;
 esac
 
-echo "mpi=${MPI}" >> "$GITHUB_OUTPUT"
+safe_mpi=$(printf '%s' "${MPI}" | tr -d '\n\r')
+echo "mpi=${safe_mpi}" >> "$GITHUB_OUTPUT"
 
 case "$MPI" in
     mpich)

```

Locations:

- `setup-mpi.sh`

## Iteration Notes

### Iteration 1

**Fixes applied:** github-env-injection, prior-fix

**Notes:**

I fixed both findings in setup-mpi.sh. action.yml needed no change.

- **github-env-injection:** in setup-env-intel-oneapi(), each value is now passed through `printf '%s' ... | tr -d '\n\r'` before it is written to $GITHUB_PATH or $GITHUB_ENV. That covers I_MPI_ROOT/bin, ONEAPI_ROOT, I_MPI_ROOT, FI_PROVIDER_PATH, LD_LIBRARY_PATH and PKG_CONFIG_PATH.
- **prior-fix (setup-mpi.sh):** I re-applied the previous harden's changes. Besides the lines above, that also strips newlines from the mpi value before it goes to $GITHUB_OUTPUT. The script still stops if one of these variables is unset (`set -u`), as upstream does.
- **prior-fix (action.yml):** the file already passes inputs.mpi to the script through the step's env block as $MPI. The rest of the previous patch was formatting (the `---` line and comment spacing), so I left the file alone.

Nothing was run or tested in this pass.

