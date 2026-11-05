<!-- markdownlint-disable -->

# Hardening Report: lycheeverse--lychee-action/v2.9.0

> This file was generated automatically by the hardening agent.

**Policy SHA:** `374911c4be584df24e6da57af2d990570ef0f859`

**Test Policy SHA:** `843adf9e4b8f85d0c08b27b9d0b09dd094b54702`

**Harden Agent Version:** `2`

Action **lycheeverse--lychee-action/v2.9.0** was hardened automatically. 6 finding(s) were identified and resolved across 2 iteration(s).

## Findings Fixed

### script-injection (severity: high)

Rule (a): the 'Install lychee' step puts a step output expression straight into the run: script: `install -t "$RUNNER_TEMP/lychee/bin" -D "${{ steps.lychee-setup.outputs.temp_dir }}/lychee"`. The value is substituted into the YAML template before the shell parses it. Pass it through an env: variable (e.g. TEMP_DIR) and use "$TEMP_DIR" instead.

Locations:

- `action.yml:112`

### script-injection (severity: high)

Rule (a): the 'Run Lychee' step puts a context expression straight into the run: line: `run: ${{ github.action_path }}/entrypoint.sh`. Use the env var instead, quoted: `"$GITHUB_ACTION_PATH/entrypoint.sh"`.

Locations:

- `action.yml:117`

### github-env-injection (severity: high)

The 'Download and extract lychee' step writes `echo "temp_dir=${TEMP_DIR}" >> $GITHUB_OUTPUT` without the sanitization step. TEMP_DIR comes from the inherited RUNNER_TEMP env var. It may also be replaced by a glob over the extracted archive's folder names, and that archive is picked by inputs.lycheeVersion. Strip newlines first, e.g. `safe=$(printf '%s' "$TEMP_DIR" | tr -d '\n\r')`, and quote "$GITHUB_OUTPUT".

Locations:

- `action.yml:105`

### github-env-injection (severity: high)

The 'Set up environment' step writes `echo "$RUNNER_TEMP/lychee/bin" >> "$GITHUB_PATH"`. RUNNER_TEMP is a process env var this script reads but does not set, so the composite-action rule treats it as untrusted. It is written to GITHUB_PATH without `printf '%s' ... | tr -d '\n\r'` sanitization.

Locations:

- `action.yml:61`

### prior-fix (severity: high)

Carried forward from this version's previous harden, which fixed it; fix it again unless the code no longer has the problem. The previous harden changed action.yml without recording a finding for it, and this analysis reported nothing there. Re-apply its security changes unless the code no longer needs them. The previous patch:
```diff
diff --git a/action.yml b/action.yml
index 217df7d..cb943b0 100644
--- a/action.yml
+++ b/action.yml
@@ -52,6 +52,23 @@ outputs:
 runs:
   using: "composite"
   steps:
     - name: Set up environment
       run: |
         # Install into $RUNNER_TEMP (not $HOME) so the install path is stable
@@ -107,15 +124,17 @@ runs:
         LYCHEE_VERSION: ${{ inputs.lycheeVersion }}
       shell: bash
     - name: Install lychee
+      env:
+        LYCHEE_TEMP_DIR: ${{ steps.lychee-setup.outputs.temp_dir }}
       run: |
         # Install lychee from the temporary directory
-        install -t "$RUNNER_TEMP/lychee/bin" -D "${{ steps.lychee-setup.outputs.temp_dir }}/lychee"
+        install -t "$RUNNER_TEMP/lychee/bin" -D "${LYCHEE_TEMP_DIR}/lychee"
       shell: bash
     - name: Run Lychee
       id: run-lychee
       working-directory: ${{ inputs.workingDirectory }}
-      run: ${{ github.action_path }}/entrypoint.sh
       env:
+        ACTION_PATH: ${{ github.action_path }}
         # https://github.com/actions/runner/issues/665
         INPUT_TOKEN: ${{ inputs.TOKEN }}
         INPUT_ARGS: ${{ inputs.ARGS }}
@@ -127,6 +146,7 @@ runs:
         INPUT_CHECKBOX: ${{ inputs.CHECKBOX }}
         INPUT_OUTPUT: ${{ inputs.OUTPUT }}
         SUMMARY_URL: ${{ github.server_url }}/${{ github.repository }}/actions/runs/${{ github.run_id }}#summary-${{ job.check_run_id }}
+      run: "$ACTION_PATH/entrypoint.sh"
       shell: bash
 branding:
   icon: "external-link"

```

Locations:

- `action.yml`

### prior-fix (severity: high)

Carried forward from this version's previous harden, which fixed it; fix it again unless the code no longer has the problem. The previous harden changed entrypoint.sh without recording a finding for it, and this analysis reported nothing there. Re-apply its security changes unless the code no longer needs them. The previous patch:
```diff
diff --git a/entrypoint.sh b/entrypoint.sh
index 2829212..b056767 100755
--- a/entrypoint.sh
+++ b/entrypoint.sh
@@ -23,10 +23,10 @@ if [ -n "${INPUT_TOKEN:-}" ]; then
 fi
 
 ARGS="${INPUT_ARGS}"
-FORMAT=""
+FORMAT_ARGS=()
 # Backwards compatibility:
 # If `format` occurs in args, ignore the value from `INPUT_FORMAT`
-[[ "$ARGS" =~ "--format " ]] || FORMAT="--format ${INPUT_FORMAT}"
+[[ "$ARGS" =~ "--format " ]] || FORMAT_ARGS=("--format" "${INPUT_FORMAT}")
 
 
 # If `output` occurs in args and `INPUT_OUTPUT` is set, exit with an error 
@@ -42,7 +42,7 @@ if [[ "$ARGS" =~ "--mode" ]] && [ -n "${INPUT_CHECKBOX:-}" ]; then
   exit 1
 fi
 
-CHECKBOX=""
+CHECKBOX_ARGS=()
 if [ "${INPUT_CHECKBOX}" = true ]; then
   # Warn if the version is lower than 0.18.1
   function version_lt() { test "$(echo "$@" | tr " " "\n" | sort -rV | head -n 1)" != "$1"; }
@@ -50,12 +50,20 @@ if [ "${INPUT_CHECKBOX}" = true ]; then
   if version_lt "$lychee_version" "0.18.1"; then
     echo "WARNING: 'checkbox' is not supported in lychee versions lower than 0.18.1 (current is $lychee_version). Continuing without 'checkbox'."
   else
-    CHECKBOX="--mode task"
+    CHECKBOX_ARGS=("--mode" "task")
   fi
 fi
 
+# Tokenize ARGS (a whitespace-separated list of arguments) into an array,
+# honoring quotes and backslashes the same way the shell would.
+ARGS_ARRAY=()
+if [ -n "$ARGS" ]; then
+  while IFS= read -r -d '' t; do ARGS_ARRAY+=("$t"); done \
+    < <(printf '%s' "$ARGS" | xargs printf '%s\0')
+fi
+
 # Execute lychee
-eval lychee ${CHECKBOX} ${FORMAT} --output ${LYCHEE_TMP} ${ARGS} 
+lychee "${CHECKBOX_ARGS[@]}" "${FORMAT_ARGS[@]}" --output "${LYCHEE_TMP}" "${ARGS_ARRAY[@]}"
 LYCHEE_EXIT_CODE=$?
 
 # If no links were found and `failIfEmpty` is set to `true` (and it is by default),

```

Locations:

- `entrypoint.sh`

## Iteration Notes

### Iteration 1

**Fixes applied:** script-injection, github-env-injection, prior-fix

**Notes:**

action.yml: steps.lychee-setup.outputs.temp_dir moved to env LYCHEE_TEMP_DIR; entrypoint now invoked as "$GITHUB_ACTION_PATH/entrypoint.sh"; GITHUB_PATH and GITHUB_OUTPUT writes are sanitized with tr -d '\n\r' and $GITHUB_OUTPUT is quoted. entrypoint.sh: removed eval; FORMAT/CHECKBOX built as arrays; INPUT_ARGS tokenized quote-aware via guarded xargs -r split (with an unmatched-quote check that exits with an error), arrays expanded in a set -u safe way.

### Iteration 1

**Fixes applied:** publish-gate

**Notes:**

Repaired two garbled lines in entrypoint.sh that stopped the script from parsing. (1) The checkbox warning `echo` was cut off mid-string and merged into the next line. I closed the string ("Continuing without 'checkbox'.") and put back the `else` branch that sets CHECKBOX_ARGS=(--mode task). That `else` structure is rebuilt from the surrounding logic, not copied from upstream. (2) The lychee command had leftover text after the args array (`_TMP} ${ARGS}`). I removed it, so lychee now gets only the args array built by the safe xargs split, with no unquoted ${ARGS} and no eval. Nothing else was changed. Not verified: I couldn't run the script through a parser here; I only re-read the edited section.

