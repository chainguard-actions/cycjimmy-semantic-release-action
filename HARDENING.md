<!-- markdownlint-disable -->

# Hardening Report: cycjimmy--semantic-release-action/v2.7.0

> This file was generated automatically by the hardening agent.

**Policy SHA:** `d636be7e43ef829af6e853da6b3c7566db9f72fe`

**Test Policy SHA:** `843adf9e4b8f85d0c08b27b9d0b09dd094b54702`

**Harden Agent Version:** `2`

Action **cycjimmy--semantic-release-action/v2.7.0** was hardened automatically. 3 finding(s) were identified and resolved across 2 iteration(s).

## Findings Fixed

### unpinned-uses (severity: high)

All three workflow files reference external actions using mutable version tags instead of pinned full-length SHA commit hashes. This exposes the workflow to supply-chain attacks if a tag is moved or an action is compromised.

Failing references:
- checkPullRequest.yml: `uses: actions/checkout@v2`
- release.yml: `uses: actions/checkout@v2`, `uses: cycjimmy/semantic-release-action@v2`, `uses: actions/setup-node@v2`
- testRelease.yml: `uses: actions/checkout@v2` (appears in both jobs)

All should be pinned to a full 40-character commit SHA, e.g. `uses: actions/checkout@11bd71901bbe5b1630ceea73d27597364c9af683 # v2`.

Locations:

- `.github/workflows/checkPullRequest.yml:14`
- `.github/workflows/release.yml:21`
- `.github/workflows/release.yml:23`
- `.github/workflows/release.yml:35`
- `.github/workflows/testRelease.yml:25`
- `.github/workflows/testRelease.yml:63`

### script-injection (severity: high)

GitHub Actions expressions (`${{ ... }}`) are interpolated directly inside `run:` shell command strings, violating sub-rule (a). Before the shell executes the command, GitHub substitutes the expression value as raw text, allowing an attacker who controls the value to inject arbitrary shell commands.

**release.yml, line 47** — `steps.semantic.outputs.new_release_major_version` is interpolated directly into a `git push` URL:
```
run: "git push https://x-access-token:${GITHUB_TOKEN}@github.com/${GITHUB_REPOSITORY}.git HEAD:refs/heads/v${{steps.semantic.outputs.new_release_major_version}}"
```

**testRelease.yml, lines 53–56 and 88–91** — `steps.semantic.outputs.*` values are interpolated directly into `echo` commands:
```
run: |
  echo ${{ steps.semantic.outputs.new_release_version }}
  echo ${{ steps.semantic.outputs.new_release_major_version }}
  echo ${{ steps.semantic.outputs.new_release_minor_version }}
  echo ${{ steps.semantic.outputs.new_release_patch_version }}
```

Fix: move the values into `env:` variables and reference them as `"$ENV_VAR"` in the shell script.

Locations:

- `.github/workflows/release.yml:47`
- `.github/workflows/testRelease.yml:53`
- `.github/workflows/testRelease.yml:88`

### missing-permissions (severity: medium)

None of the three workflow files define a `permissions:` key at the top level or at the job level. Without explicit permissions, workflows inherit the default repository permissions (which may be `write-all` depending on repository settings), granting jobs broader access than necessary. Each workflow should declare the minimal required permissions, e.g.:
```yaml
permissions:
  contents: read
```

Locations:

- `.github/workflows/checkPullRequest.yml:1`
- `.github/workflows/release.yml:1`
- `.github/workflows/testRelease.yml:1`

## Iteration Notes

### Iteration 1

**Fixes applied:** unpinned-uses, script-injection, missing-permissions

**Notes:**

Fixed all three findings across checkPullRequest.yml, release.yml, and testRelease.yml:

1. unpinned-uses: Pinned all action references to full 40-char SHAs — actions/checkout@v2 → @0717577d45739eb3c851188b29f50ed6c0b2194e, cycjimmy/semantic-release-action@v2 → @5982a02995853159735cb838992248c4f0f16166, actions/setup-node@v2 → @7c12f8017d5436eb855f1ed4399f037a36fbd9e8. Original tags preserved as inline comments.

2. script-injection: Moved all ${{ steps.semantic.outputs.* }} expressions out of run: blocks into env: variables. In release.yml the git push command now uses ${NEW_RELEASE_MAJOR_VERSION}. In testRelease.yml both 'Test Outputs' steps now reference env vars ($NEW_RELEASE_VERSION, etc.) instead of inline expressions.

3. missing-permissions: Added top-level permissions blocks — checkPullRequest.yml and testRelease.yml get 'contents: read' (dry-run workflows), release.yml gets 'contents: write' (pushes branches) and 'packages: write' (publishes npm packages to GitHub Package Registry).

### Iteration 1

**Fixes applied:** script-injection

**Notes:**

Fixed the script injection vulnerability in `.github/workflows/release.yml` at the 'Push updates to branch for major version' step. The original `run:` was a single YAML double-quoted string, leaving `${NEW_RELEASE_MAJOR_VERSION}` unquoted at the shell level. Restructured to a multi-line `run: |` block where both the URL and refspec arguments are individually double-quoted at the shell level, preventing shell metacharacter injection from the attacker-controllable `new_release_major_version` step output.

