<!-- markdownlint-disable -->

# Hardening Report: cycjimmy--semantic-release-action/v6.0.0

> This file was generated automatically by the hardening agent.

**Policy SHA:** `d636be7e43ef829af6e853da6b3c7566db9f72fe`

**Test Policy SHA:** `843adf9e4b8f85d0c08b27b9d0b09dd094b54702`

**Harden Agent Version:** `2`

Action **cycjimmy--semantic-release-action/v6.0.0** was hardened automatically. 3 finding(s) were identified and resolved across 1 iteration(s).

## Findings Fixed

### unpinned-uses (severity: high)

Multiple workflow files reference GitHub Actions using mutable version tags instead of full 40-character commit SHAs. This exposes the workflow to supply-chain attacks if the tag is moved to a malicious commit. Failing references: `actions/checkout@v5` (all three workflow files) and `actions/setup-node@v6` (release.yml). All should be pinned to a full SHA, e.g. `actions/checkout@11bd71901bbe5b1630ceea73d27597364c9af683 # v5`.

Locations:

- `.github/workflows/checkPullRequest.yml:13`
- `.github/workflows/release.yml:18`
- `.github/workflows/release.yml:47`
- `.github/workflows/testRelease.yml:19`
- `.github/workflows/testRelease.yml:62`

### script-injection (severity: high)

Rule (a): GitHub Actions expressions (`${{ ... }}`) are interpolated directly inside `run:` shell command strings. Before the shell executes the command, GitHub Actions substitutes the expression value as raw text, allowing an attacker who can influence the output value to inject arbitrary shell commands. Affected steps:

1. `.github/workflows/testRelease.yml` — "Test Outputs" step in `test-semantic-latest` job: `echo ${{ steps.semantic.outputs.new_release_version }}` (and three similar lines).
2. `.github/workflows/testRelease.yml` — "Test Outputs" step in `test-semantic-v15` job: same pattern.
3. `.github/workflows/release.yml` — "Push updates to branch for major version" step: `run: "git push ... HEAD:refs/heads/v${{steps.semantic.outputs.new_release_major_version}}"`.

Fix: move the values into `env:` variables and reference them as quoted shell variables, e.g. `echo "$NEW_RELEASE_VERSION"`.

Locations:

- `.github/workflows/testRelease.yml:52`
- `.github/workflows/testRelease.yml:82`
- `.github/workflows/release.yml:55`

### missing-permissions (severity: medium)

None of the three workflow files define a top-level `permissions:` block, and no individual job defines its own `permissions:` block. Without explicit permissions, workflows run with the repository's default token permissions, which may be overly broad (e.g. `write` access to contents, pull-requests, etc.). Each workflow should declare the minimal permissions required, for example `permissions: contents: read` at the top level and elevate only where needed.

Locations:

- `.github/workflows/checkPullRequest.yml:1`
- `.github/workflows/release.yml:1`
- `.github/workflows/testRelease.yml:1`

## Iteration Notes

### Iteration 1

**Fixes applied:** unpinned-uses, script-injection, missing-permissions

**Notes:**

Fixed all three findings across the three workflow files: (1) Pinned actions/checkout@v5 to SHA fbc6f3992d24b796d5a048ff273f7fcc4a7b6c09 in all three files, and actions/setup-node@v6 to SHA 249970729cb0ef3589644e2896645e5dc5ba9c38 in release.yml. (2) Fixed script injection in testRelease.yml (both Test Outputs steps) and release.yml (git push step) by moving ${{ }} expressions into env: blocks and referencing them as quoted shell variables. (3) Added minimal permissions blocks to all three files: contents:read for checkPullRequest.yml and testRelease.yml (dry-run workflows), and contents:write + packages:write for release.yml (which pushes branches and publishes packages).

