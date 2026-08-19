<!-- markdownlint-disable -->

# Hardening Report: lycheeverse--lychee-action/v2.9.0

> This file was generated automatically by the hardening agent.

**Policy SHA:** `d636be7e43ef829af6e853da6b3c7566db9f72fe`

**Test Policy SHA:** `843adf9e4b8f85d0c08b27b9d0b09dd094b54702`

**Harden Agent Version:** `2`

Action **lycheeverse--lychee-action/v2.9.0** was hardened automatically. 3 finding(s) were identified and resolved across 2 iteration(s).

## Findings Fixed

### script-injection (severity: high)

Multiple run: blocks directly interpolate ${{ ... }} expressions inside shell commands, violating rule (a). In action.yml, the 'Install lychee' step embeds `${{ steps.lychee-setup.outputs.temp_dir }}` directly in a shell command (`install -t ... -D "${{ steps.lychee-setup.outputs.temp_dir }}/lychee"`), and the 'Run Lychee' step uses `run: ${{ github.action_path }}/entrypoint.sh` — both allow expression values to be interpreted by the shell before quoting. In lychee-version.yml, the 'Compare versions' step assigns `action_lychee_version="${{ steps.get-action-lychee-version.outputs.result }}"` and `release_version="${{ steps.get-lychee-release.outputs.release_version }}"` directly inside a run: block. In test.yml, multiple steps embed `${{ github.workspace }}`, `${{ env.CUSTOM_OUTPUT_RELATIVE_PATH }}`, `${{ env.CUSTOM_OUTPUT_ABSOLUTE_PATH }}`, `${{ env.CUSTOM_OUTPUT_DUMP_PATH }}`, and `${{ steps.lychee_exit_code_test.outputs.exit_code }}` directly inside run: shell commands.

Locations:

- `action.yml:112`
- `action.yml:118`
- `.github/workflows/lychee-version.yml:57`
- `.github/workflows/lychee-version.yml:58`
- `.github/workflows/test.yml:52`
- `.github/workflows/test.yml:63`
- `.github/workflows/test.yml:64`
- `.github/workflows/test.yml:75`
- `.github/workflows/test.yml:76`
- `.github/workflows/test.yml:87`
- `.github/workflows/test.yml:88`
- `.github/workflows/test.yml:248`
- `.github/workflows/test.yml:249`
- `.github/workflows/test.yml:251`

### unpinned-uses (severity: high)

Multiple workflow files reference actions using mutable tags or branch names instead of immutable 40-character commit SHAs, making them vulnerable to supply-chain attacks. Unpinned references found: auto-merge.yml: `actions/checkout@v7`, `ahmadnassri/action-dependabot-auto-merge@v2.6.6`; links.yml: `actions/checkout@v7`, `peter-evans/create-issue-from-file@v6`; lychee-version.yml: `actions/checkout@v7` (×2), `mikefarah/yq@master` (×2), `peter-evans/create-pull-request@v8`; test.yml: `actions/checkout@v7` (×2); test_cache.yml: `actions/cache@v6`, `actions/checkout@v7`, `lycheeverse/lychee-action@v2`; versioning.yml: `Actions-R-Us/actions-tagger@v2`.

Locations:

- `.github/workflows/auto-merge.yml:9`
- `.github/workflows/auto-merge.yml:10`
- `.github/workflows/links.yml:14`
- `.github/workflows/links.yml:24`
- `.github/workflows/lychee-version.yml:20`
- `.github/workflows/lychee-version.yml:33`
- `.github/workflows/lychee-version.yml:37`
- `.github/workflows/lychee-version.yml:96`
- `.github/workflows/lychee-version.yml:100`
- `.github/workflows/test.yml:16`
- `.github/workflows/test.yml:258`
- `.github/workflows/test_cache.yml:14`
- `.github/workflows/test_cache.yml:19`
- `.github/workflows/test_cache.yml:21`
- `.github/workflows/versioning.yml:10`

### missing-permissions (severity: medium)

Several workflow files have no top-level `permissions:` key and at least one job also lacks a job-level `permissions:` block, meaning the GITHUB_TOKEN is granted its default (often broad) permissions. Affected files: `auto-merge.yml` (no top-level permissions, `auto-merge` job has none); `test.yml` (no top-level permissions, both `lychee-action` and `lychee-action-arm` jobs have none); `test_cache.yml` (no top-level permissions, `lychee-action` job has none); `versioning.yml` (no top-level permissions, `actions-tagger` job has none); `lychee-version.yml` (no top-level permissions, `check-lychee-version` job has none — only the `create-pr` job has explicit permissions).

Locations:

- `.github/workflows/auto-merge.yml:1`
- `.github/workflows/test.yml:1`
- `.github/workflows/test_cache.yml:1`
- `.github/workflows/versioning.yml:1`
- `.github/workflows/lychee-version.yml:10`

## Iteration Notes

### Iteration 1

**Fixes applied:** script-injection, unpinned-uses, missing-permissions

**Notes:**

Fixed all three findings across action.yml and .github/workflows/*.yml files:

**script-injection**: Moved all ${{ }} expressions out of run: blocks into env: blocks in action.yml (Install lychee step: LYCHEE_TEMP_DIR; Run Lychee step: ACTION_PATH), lychee-version.yml (Compare versions step: ACTION_LYCHEE_VERSION, RELEASE_VERSION), and test.yml (multiple steps: GITHUB_WORKSPACE_PATH, CUSTOM_OUTPUT_RELATIVE_PATH, CUSTOM_OUTPUT_ABSOLUTE_PATH, CUSTOM_OUTPUT_DUMP_PATH, LYCHEE_EXIT_CODE).

**unpinned-uses**: Pinned all 8 action references to full 40-character commit SHAs with tag comments: actions/checkout@v7→3d3c42e5, ahmadnassri/action-dependabot-auto-merge@v2.6.6→45fc124d, peter-evans/create-issue-from-file@v6→fca9117c, mikefarah/yq@master→7862131c (×2), peter-evans/create-pull-request@v8→5f6978fa, actions/cache@v6→55cc8345, lycheeverse/lychee-action@v2→e7477775, Actions-R-Us/actions-tagger@v2→330ddfac.

**missing-permissions**: Added top-level `permissions: {}` to auto-merge.yml, test.yml, test_cache.yml, and versioning.yml. Added job-level permissions to all affected jobs: auto-merge (contents: write, pull-requests: write), lychee-action/lychee-action-arm in test.yml (contents: read), lychee-action in test_cache.yml (contents: read), actions-tagger in versioning.yml (contents: write), check-lychee-version in lychee-version.yml (contents: read).

### Iteration 2

**Fixes applied:** script-injection

**Notes:**

Fixed the script injection vulnerability in entrypoint.sh by removing `eval` and replacing unquoted variable expansions with proper bash arrays. FORMAT (--format flag+value) is now FORMAT_ARGS array with two elements. CHECKBOX (--mode task) is now CHECKBOX_ARGS array with two elements. The user-controlled ARGS list input is tokenized into ARGS_ARRAY using xargs with the required non-empty guard, honoring quotes and backslashes. The final lychee invocation uses double-quoted array expansions throughout, preventing any shell metacharacter injection.

