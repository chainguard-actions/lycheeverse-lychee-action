<!-- markdownlint-disable -->

# Hardening Report: lycheeverse--lychee-action/v2.8.0

> This file was generated automatically by the hardening agent.

**Policy SHA:** `d636be7e43ef829af6e853da6b3c7566db9f72fe`

**Test Policy SHA:** `843adf9e4b8f85d0c08b27b9d0b09dd094b54702`

**Harden Agent Version:** `1`

Action **lycheeverse--lychee-action/v2.8.0** was hardened automatically. 2 finding(s) were identified and resolved across 2 iteration(s).

## Findings Fixed

### script-injection (severity: high)

Sub-rule (a): A ${{ steps.lychee-setup.outputs.temp_dir }} expression (steps.*.outputs.* context) is directly interpolated inside a run: shell command string in the 'Install lychee' step. Before the shell executes, GitHub Actions performs YAML template substitution, so any attacker-controlled value in that output could inject shell metacharacters. The offending line is: install -t "$HOME/.local/bin" -D "${{ steps.lychee-setup.outputs.temp_dir }}/lychee". The fix is to pass the value via an env: variable and reference it as a quoted shell variable.

Locations:

- `action.yml:104`

### script-injection (severity: high)

Sub-rule (a): A ${{ github.action_path }} expression (github.* context) is directly interpolated as the entire run: command in the 'Run Lychee' step. The offending line is: run: ${{ github.action_path }}/entrypoint.sh. Even though github.action_path is typically GitHub-controlled, any ${{ ... }} expression inside a run: block undergoes YAML template substitution before shell quoting, making it a script-injection risk. The fix is to use the $GITHUB_ACTION_PATH environment variable instead.

Locations:

- `action.yml:109`

## Iteration Notes

### Iteration 1

**Fixes applied:** script-injection

**Notes:**

Fixed two script-injection findings in action.yml:
1. 'Install lychee' step (line 104): Moved `${{ steps.lychee-setup.outputs.temp_dir }}` into an `env:` block as `LYCHEE_TEMP_DIR` and referenced it as `$LYCHEE_TEMP_DIR` in the shell command.
2. 'Run Lychee' step (line 109): Replaced `run: ${{ github.action_path }}/entrypoint.sh` with `run: "$GITHUB_ACTION_PATH/entrypoint.sh"`, using the built-in environment variable instead of YAML template substitution.

### Iteration 2

**Fixes applied:** script-injection

**Notes:**

Fixed the script injection vulnerability in entrypoint.sh by replacing the dangerous `eval lychee ${CHECKBOX} ${FORMAT} --output ${LYCHEE_TMP} ${ARGS}` call with a safe direct invocation using bash arrays. Changes: (1) `FORMAT` string replaced with `FORMAT_ARGS` array `("--format" "${INPUT_FORMAT}")` so INPUT_FORMAT is always a single literal argument; (2) `CHECKBOX` string replaced with `CHECKBOX_ARGS` array `("--mode" "task")`; (3) `ARGS` is split into `ARGS_ARRAY` via `read -ra` (word-splitting on whitespace only, not shell metacharacters); (4) `eval` replaced with direct `lychee "${CHECKBOX_ARGS[@]}" "${FORMAT_ARGS[@]}" --output "${LYCHEE_TMP}" "${ARGS_ARRAY[@]}"`. Shell metacharacters like `;`, `|`, `$(...)`, and backticks in user-supplied inputs are now passed as literal arguments to lychee rather than being interpreted by the shell.

