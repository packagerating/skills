---
name: github-pr-dependency-review
description: "Use this skill when a user asks Claude to review, check, or audit a specific GitHub pull request's dependency changes interactively — e.g. \"check this PR's dependency changes\", \"did this PR add anything risky\", \"review the deps in PR #42\", or when the user pastes a PR URL and asks about its dependencies. Trigger even without an explicit PR reference if the user is clearly discussing \"this PR\" from their current git branch/session context. This is for ad hoc, in-conversation review of ONE specific PR's diff — not a substitute for the automated audit-dependencies/audit-dependencies-python GitHub Actions, which already score every dependency in a PR's final manifest on every push; this skill instead looks only at what changed. Requires the gh CLI (authenticated) and the packagerating MCP server (get_package tool); if either is unavailable, say so and stop rather than guessing."
---

# GitHub PR Dependency Review

Help a developer understand what a specific pull request actually changed in its npm/PyPI
dependencies — not a full audit of every dependency (that's what the automated
`audit-dependencies`/`audit-dependencies-python` GitHub Actions already do on every push), but a
focused look at what's new, removed, or version-bumped in this one PR's diff.

## Before you start: confirm the tools exist

This skill depends on the `gh` CLI (authenticated) and the packagerating MCP server's `get_package`
tool. If `gh` isn't available or isn't authenticated, tell the user to run `gh auth login` and stop.
If the MCP tools aren't available, tell the user the packagerating MCP server isn't configured and
point them to https://github.com/packagerating/mcp-server, then stop.

## Workflow

### 1. Resolve the target PR

If the user names a PR (number, URL, or clearly means "this PR" from context), use that. Otherwise,
run `gh pr view` to resolve from the current branch/repo. If neither resolves to a PR, say so and
ask which PR to look at.

### 2. Pull the diff, filtered to manifest files

Run `gh pr diff <PR>` and look only at hunks touching recognized manifest files: `package.json`,
`requirements.txt`, `pyproject.toml`, `Pipfile`, and similar — not lockfiles (`package-lock.json`,
`yarn.lock`, `pnpm-lock.yaml`, `poetry.lock`, etc.), which are out of scope for this skill (their
diff format varies too much to parse reliably from raw text without dedicated tooling).

### 3. Read the diff directly to find what changed

No bundled parser — read the `+`/`-` lines yourself to identify:
- **Added dependencies**: a new line for a package that wasn't there before.
- **Removed dependencies**: a line removed for a package with no replacement line for the same
  name.
- **Version bumps**: the same package name appears on both a `-` and `+` line with a different
  version.

Skip `devDependencies` (and dev-only Python dependency groups/extras) by default — mention them only
if the user explicitly asks about dev dependencies specifically.

### 4. Score what changed

For each **added** or **version-bumped** dependency, call `get_package` with the correct
`language` (`javascript` for npm, `python` for PyPI). For a version bump, look up both the old and
new version via `get_package`'s `version` param when both have been crawled, so you can show what
actually moved — not just the new version's score in isolation. If a lookup comes back
`still_crawling`, say so plainly and offer to retry rather than presenting partial data as final.

**Removed** dependencies are acknowledged by name only — there's nothing to score.

If a changed dependency's name can't be resolved (private/scoped package not on the public
registry, a typo, etc.), note it by name and move on to the rest — don't let one unresolvable
package block the whole report.

### 5. Present — data, not a verdict

Same convention as the `package-adoption` skill: composite scores (General, Automation, Risk) plus
all six dimensions (Liveness, Community, Security, Dependency, Versioning, Dep-tree risk), grounded
in the actual numbers returned. Structure the report as:

- **Added dependencies**: one row per package in a comparison table, same format `package-adoption`
  uses.
- **Version bumps**: show the old and new version's numbers side by side, and call out which
  dimensions moved and by how much — this is usually the most useful part of the report.
- **Removed dependencies**: a short list by name, no scores.

**Do not issue a recommend/avoid verdict or apply a hardcoded score threshold.** Point out specific
factual observations tied to the data (e.g. "the version bump takes `security` from 40 to 100 — this
looks like it fixes a previously-open advisory") without turning that into "so this PR is safe to
merge." Different teams weigh these dimensions differently — leave the call to the user.

### 6. Self-check for methodology drift (once per session)

The first time either this skill or `package-adoption` runs in a session, call `get_package` for
`axios` (language `javascript`) and confirm its `dimensions` object has exactly these six keys:
`liveness`, `community`, `security`, `dependency`, `versioning`, `dep_risk`. If any are missing,
renamed, or new ones have appeared, say so plainly — this skill's understanding of the dimensions
may be out of date. Don't repeat this check if it already ran once this session (by this skill or
`package-adoption`).

## Output: conversational only

This skill never creates or updates a PR comment or otherwise posts to GitHub. Report findings
directly in the conversation. If the user wants findings posted somewhere, that's a separate,
explicit request in the moment — not something this skill does on its own.

## Example

**User:** "check this PR's dependency changes" (currently on a branch with an open PR)

1. `gh pr view` resolves PR #42.
2. `gh pr diff 42` shows `package.json` changed: `dayjs` added at `^1.11.10`, `moment` removed,
   `axios` bumped from `1.5.0` to `1.6.2`.
3. `get_package` for `dayjs@1.11.10`, `axios@1.5.0`, `axios@1.6.2` (all `language: javascript`).
   `moment` isn't scored since it's being removed.
4. Present:

**Added:**

| Package | General | Automation | Risk | Liveness | Community | Security | Dependency | Versioning | Dep-tree |
|---|---|---|---|---|---|---|---|---|---|
| dayjs@1.11.10 | 58.0 | 69.0 | 28.5 | 15 | 30 | 100 | 70 | 70 | 100 |

**Version bump — axios:**

| Version | General | Risk | Security | Dependency |
|---|---|---|---|---|
| 1.5.0 | 37.5 | 84.0 | 0 | 65 |
| 1.6.2 | 52.0 | 60.0 | 70 | 65 |

"The axios bump takes `security` from 0 to 70 and drops the risk score from 84 to 60 — looks like it
picks up fixes for whatever had audit findings open on 1.5.0."

**Removed:** `moment`
