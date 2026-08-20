<!-- markdownlint-disable -->

# Hardening Report: lycheeverse--lychee-action/v2.8.0

> This file was generated automatically by the hardening agent.

**Policy SHA:** `d636be7e43ef829af6e853da6b3c7566db9f72fe`

**Test Policy SHA:** `843adf9e4b8f85d0c08b27b9d0b09dd094b54702`

**Harden Agent Version:** `2`

Action **lycheeverse--lychee-action/v2.8.0** was hardened automatically. 3 finding(s) were identified and resolved across 2 iteration(s).

## Findings Fixed

### script-injection (severity: high)

Sub-rule (a): Direct ${{ ... }} expression interpolation inside run: blocks. In action.yml, the 'Install lychee' step interpolates ${{ steps.lychee-setup.outputs.temp_dir }} directly in the shell command string, and the 'Run Lychee' step uses ${{ github.action_path }} as the run: value itself. In lychee-version.yml, the 'compare-versions' step interpolates ${{ steps.get-action-lychee-version.outputs.result }} and ${{ steps.get-lychee-release.outputs.release_version }} directly inside a run: block. In test.yml, multiple run: blocks interpolate ${{ github.workspace }}, ${{ env.CUSTOM_OUTPUT_RELATIVE_PATH }}, ${{ env.CUSTOM_OUTPUT_ABSOLUTE_PATH }}, ${{ env.CUSTOM_OUTPUT_DUMP_PATH }}, and ${{ steps.lychee_exit_code_test.outputs.exit_code }} directly in shell command strings.

Locations:

- `action.yml:108`
- `action.yml:113`
- `.github/workflows/lychee-version.yml:48`
- `.github/workflows/lychee-version.yml:49`
- `.github/workflows/test.yml:47`
- `.github/workflows/test.yml:57`
- `.github/workflows/test.yml:68`
- `.github/workflows/test.yml:79`
- `.github/workflows/test.yml:155`
- `.github/workflows/test.yml:168`
- `.github/workflows/test.yml:179`
- `.github/workflows/test.yml:190`
- `.github/workflows/test.yml:218`
- `.github/workflows/test.yml:226`
- `.github/workflows/test.yml:234`
- `.github/workflows/test.yml:265`
- `.github/workflows/test.yml:270`
- `.github/workflows/test.yml:275`

### unpinned-uses (severity: high)

All uses: references across workflow files and action.yml use mutable tags or branch names instead of pinned 40-character SHA commit hashes. Failing references include: actions/checkout@v6, ahmadnassri/action-dependabot-auto-merge@v2.6.6, peter-evans/create-issue-from-file@v6, mikefarah/yq@master, peter-evans/create-pull-request@v8, actions/cache@v5, lycheeverse/lychee-action@v2, Actions-R-Us/actions-tagger@v2.

Locations:

- `.github/workflows/auto-merge.yml:10`
- `.github/workflows/auto-merge.yml:11`
- `.github/workflows/links.yml:14`
- `.github/workflows/links.yml:22`
- `.github/workflows/lychee-version.yml:20`
- `.github/workflows/lychee-version.yml:30`
- `.github/workflows/lychee-version.yml:55`
- `.github/workflows/lychee-version.yml:63`
- `.github/workflows/lychee-version.yml:67`
- `.github/workflows/test.yml:18`
- `.github/workflows/test.yml:246`
- `.github/workflows/test_cache.yml:16`
- `.github/workflows/test_cache.yml:19`
- `.github/workflows/test_cache.yml:22`
- `.github/workflows/versioning.yml:11`

### missing-permissions (severity: medium)

The following workflow files have no top-level permissions: block and no job-level permissions: blocks on any job, meaning they run with the default (potentially broad) token permissions: auto-merge.yml, test.yml, test_cache.yml, versioning.yml.

Locations:

- `.github/workflows/auto-merge.yml:1`
- `.github/workflows/test.yml:1`
- `.github/workflows/test_cache.yml:1`
- `.github/workflows/versioning.yml:1`

## Iteration Notes

### Iteration 1

**Fixes applied:** script-injection, unpinned-uses, missing-permissions

**Notes:**

Fixed all three findings across action.yml and .github/workflows/*.yml files:

1. script-injection: Moved all ${{ }} expressions out of run: blocks into step-level env: blocks in action.yml (Install lychee step uses TEMP_DIR env var, Run Lychee step uses ACTION_PATH env var), lychee-version.yml (compare-versions step uses ACTION_LYCHEE_VERSION and RELEASE_VERSION env vars), and test.yml (github.workspace, env.CUSTOM_OUTPUT_* paths, and steps.lychee_exit_code_test.outputs.exit_code all moved to env: blocks).

2. unpinned-uses: Pinned all 8 action references to full 40-character SHA hashes with tag comments: actions/checkout@v6→d23441a4, ahmadnassri/action-dependabot-auto-merge@v2.6.6→45fc124d, peter-evans/create-issue-from-file@v6→fca9117c, mikefarah/yq@master→ce7026e3, peter-evans/create-pull-request@v8→5f6978fa, actions/cache@v5→caa29612, lycheeverse/lychee-action@v2→e7477775, Actions-R-Us/actions-tagger@v2→330ddfac.

3. missing-permissions: Added top-level permissions: {} and appropriate minimal job-level permissions to auto-merge.yml (contents: write, pull-requests: write), test.yml (contents: read), test_cache.yml (contents: read), and versioning.yml (contents: write).

### Iteration 2

**Fixes applied:** script-injection, missing-permissions

**Notes:**

1. entrypoint.sh: Replaced `eval lychee ${CHECKBOX} ${FORMAT} --output ${LYCHEE_TMP} ${ARGS}` with a safe array-based invocation. CHECKBOX and FORMAT (single values) are tokenized via xargs into a lychee_args array with empty-value guards. LYCHEE_TMP is double-quoted as a path. ARGS (a list of user-controlled arguments) is tokenized via xargs with the NUL-delimited read loop pattern and an empty-value guard. eval is completely eliminated. 2. .github/workflows/lychee-version.yml: Added `permissions: contents: read` to the `check-lychee-version` job to restrict it to the minimum required permissions (read-only repository access for checkout).

