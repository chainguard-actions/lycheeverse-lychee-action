<!-- markdownlint-disable -->

# Hardening Report: lycheeverse--lychee-action/v2.8.0

> This file was generated automatically by the hardening agent.

**Policy SHA:** `374911c4be584df24e6da57af2d990570ef0f859`

**Test Policy SHA:** `843adf9e4b8f85d0c08b27b9d0b09dd094b54702`

**Harden Agent Version:** `2`

Action **lycheeverse--lychee-action/v2.8.0** was hardened automatically. 7 finding(s) were identified and resolved across 2 iteration(s).

## Findings Fixed

### script-injection (severity: high)

Sub-rule (a): the 'Install lychee' step puts a step output expression directly into the shell script: `install -t "$HOME/.local/bin" -D "${{ steps.lychee-setup.outputs.temp_dir }}/lychee"`. Pass it through an env var instead (e.g. TEMP_DIR: ${{ steps.lychee-setup.outputs.temp_dir }}) and use "$TEMP_DIR".

Locations:

- `action.yml:104`

### script-injection (severity: high)

Sub-rule (a): the 'Run Lychee' step's run line is `run: ${{ github.action_path }}/entrypoint.sh`, so the expression is interpolated straight into the shell command. Use the env var instead: `"$GITHUB_ACTION_PATH/entrypoint.sh"`.

Locations:

- `action.yml:109`

### script-injection (severity: high)

Sub-rule (b): entrypoint.sh, which the action runs, passes caller-controlled input to eval unquoted: `eval lychee ${CHECKBOX} ${FORMAT} --output ${LYCHEE_TMP} ${ARGS}`. ARGS comes from INPUT_ARGS (inputs.args) and FORMAT comes from INPUT_FORMAT (inputs.format). eval re-parses both as shell code, so metacharacters such as `;`, `$(...)` and backticks in those inputs run as commands. Build an argument array instead of using eval.

Locations:

- `entrypoint.sh:58`

### github-env-injection (severity: high)

The 'Download and extract lychee' step writes `echo "temp_dir=${TEMP_DIR}" >> $GITHUB_OUTPUT`. TEMP_DIR is built from the inherited env var RUNNER_TEMP, which this script does not set. There is no `printf '%s' ... | tr -d '\n\r'` before the write, and $GITHUB_OUTPUT is not quoted.

Locations:

- `action.yml:96`

### github-env-injection (severity: high)

The 'Set up environment' step writes `echo "$HOME/.local/bin" >> "$GITHUB_PATH"`. $HOME is inherited from the process environment and is not set in this script. It goes into GITHUB_PATH without the newline-stripping step (`printf '%s' ... | tr -d '\n\r'`).

Locations:

- `action.yml:57`

### prior-fix (severity: high)

Carried forward from this version's previous harden, which fixed it; fix it again unless the code no longer has the problem. The previous harden changed action.yml without recording a finding for it, and this analysis reported nothing there. Re-apply its security changes unless the code no longer needs them. The previous patch:
```diff
diff --git a/action.yml b/action.yml
index 1fd768b..16ec17f 100644
--- a/action.yml
+++ b/action.yml
@@ -52,6 +52,23 @@ outputs:
 runs:
   using: "composite"
   steps:
     - name: Set up environment
       run: |
         echo "$HOME/.local/bin" >> "$GITHUB_PATH"
@@ -98,15 +115,17 @@ runs:
         LYCHEE_VERSION: ${{ inputs.lycheeVersion }}
       shell: bash
     - name: Install lychee
+      env:
+        TEMP_DIR: ${{ steps.lychee-setup.outputs.temp_dir }}
       run: |
         # Install lychee from the temporary directory
-        install -t "$HOME/.local/bin" -D "${{ steps.lychee-setup.outputs.temp_dir }}/lychee"
+        install -t "$HOME/.local/bin" -D "${TEMP_DIR}/lychee"
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
@@ -118,6 +137,7 @@ runs:
         INPUT_CHECKBOX: ${{ inputs.CHECKBOX }}
         INPUT_OUTPUT: ${{ inputs.OUTPUT }}
         SUMMARY_URL: ${{ github.server_url }}/${{ github.repository }}/actions/runs/${{ github.run_id }}#summary-${{ job.check_run_id }}
+      run: "${ACTION_PATH}/entrypoint.sh"
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
index 2829212..03ab90c 100755
--- a/entrypoint.sh
+++ b/entrypoint.sh
@@ -55,7 +55,21 @@ if [ "${INPUT_CHECKBOX}" = true ]; then
 fi
 
 # Execute lychee
-eval lychee ${CHECKBOX} ${FORMAT} --output ${LYCHEE_TMP} ${ARGS} 
+lychee_args=()
+if [ -n "${CHECKBOX}" ]; then
+  while IFS= read -r -d '' t; do lychee_args+=("$t"); done \
+    < <(printf '%s' "${CHECKBOX}" | xargs printf '%s\0')
+fi
+if [ -n "${FORMAT}" ]; then
+  while IFS= read -r -d '' t; do lychee_args+=("$t"); done \
+    < <(printf '%s' "${FORMAT}" | xargs printf '%s\0')
+fi
+lychee_args+=(--output "${LYCHEE_TMP}")
+if [ -n "${ARGS}" ]; then
+  while IFS= read -r -d '' t; do lychee_args+=("$t"); done \
+    < <(printf '%s' "${ARGS}" | xargs printf '%s\0')
+fi
+lychee "${lychee_args[@]}"
 LYCHEE_EXIT_CODE=$?
 
 # If no links were found and `failIfEmpty` is set to `true` (and it is by default),

```

Locations:

- `entrypoint.sh`

## Iteration Notes

### Iteration 1

**Fixes applied:** script-injection, github-env-injection, prior-fix

**Notes:**

All findings are fixed in action.yml and entrypoint.sh. Nothing has been run or tested yet.

action.yml:
- The step output temp_dir now reaches the 'Install lychee' step through an env var (TEMP_DIR) instead of being pasted into the script.
- The 'Run Lychee' step now runs "$GITHUB_ACTION_PATH/entrypoint.sh" instead of building the path from an inline expression.
- Newlines are now stripped from the temp_dir and $HOME/.local/bin values before they are written to $GITHUB_OUTPUT and $GITHUB_PATH. $GITHUB_OUTPUT is now quoted.

entrypoint.sh:
- Replaced the eval with an argument array. The checkbox option adds `--mode task`, format adds `--format "$INPUT_FORMAT"`, and the output file is passed as `--output "$LYCHEE_TMP"`.
- The `args` input is split with a quote-aware xargs split, so quotes are still honoured the way eval honoured them. The split is guarded so an empty value adds nothing. An unmatched quote now stops the step with an error instead of silently dropping arguments.

Three behaviour changes compared with the old eval:
- Shell variables written inside `args` (for example `$GITHUB_WORKSPACE`) are no longer expanded. They reach lychee as literal text.
- Unquoted globs in `args` are no longer expanded by the shell; lychee now gets the pattern and expands it itself. The default patterns were already quoted, so this was already how they worked.
- Under eval, a value like `markdown extra` in `format` would have become two arguments; it is now passed as a single value.

### Iteration 2

**Fixes applied:** script-injection, invalid-yaml

**Notes:**

Install lychee step: steps.lychee-setup.outputs.temp_dir now goes in through env TEMP_DIR. Run Lychee step rebuilt with id: run-lychee, working-directory: ${{ inputs.workingDirectory }} and a block-scalar run: "$GITHUB_ACTION_PATH/entrypoint.sh". The leftover broken lines are removed, which fixes the YAML parse error.

