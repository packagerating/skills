---
name: packagerating-setup
description: "Use this skill when a user wants to add, set up, or integrate packagerating.com's automated dependency scoring into a GitHub repo's CI — e.g. \"add packagerating to this repo\", \"set up dependency scoring in CI\", \"integrate with packagerating.com\", \"add the audit-dependencies action\", or similar setup/integration intent, even if they don't name the action repos exactly. This is for getting the automated GitHub Action running in a repo for the first time — NOT for reviewing one specific PR's dependency changes (that's github-pr-dependency-review, which assumes the action is already running) and NOT for evaluating whether to adopt one candidate package (that's package-adoption). Requires the gh CLI, authenticated, and a git repo with a GitHub remote; if either is missing, say so and stop rather than guessing."
---

# packagerating Setup

Help a developer get packagerating.com's automated dependency scoring running in their repo's CI —
detecting which ecosystem(s) apply, writing the right GitHub Actions workflow file(s), and guiding
them through getting an API key and setting it as a repo secret. This is the "get from zero to the
automated Action existing" skill — `github-pr-dependency-review` and `package-adoption` both assume
that setup already happened.

Only two ecosystems have an action today: npm (`packagerating/audit-dependencies`) and Python
(`packagerating/audit-dependencies-python`). packagerating.com's scoring API also covers crates.io
and rubygems.org, but no GitHub Action exists yet for either — never claim to set those up.

## Before you start: confirm the tools exist

This skill needs the `gh` CLI, authenticated, and to be run from inside a git repository with a
GitHub remote. If `gh` isn't available or isn't authenticated, tell the user to run `gh auth login`
and stop. If this isn't a git repo, or has no GitHub remote, say so and stop — don't improvise a
manual alternative.

## Workflow

### 1. Detect ecosystem(s)

Look for `package.json` (npm) and `requirements.txt` / `pyproject.toml` / `Pipfile` (Python) at the
repo root. Report what you found and what you plan to set up before writing anything — e.g. "Found
package.json → I'll set up the npm action. No Python manifest found, skipping that one." — and wait
for confirmation. Each action's own `audit-workspaces`/`audit-subprojects` inputs already default to
`true` and handle monorepo/subproject discovery once the workflow is running, so this detection step
doesn't need to be exhaustive across every subdirectory — root-level detection is enough to decide
which action(s) to set up.

If nothing recognizable is found, ask the user directly which ecosystem(s)/action(s) they want
rather than concluding there's nothing to do.

### 2. Check for an existing workflow

Search `.github/workflows/*.yml` for any file already referencing `packagerating/
audit-dependencies` or `packagerating/audit-dependencies-python`. If found, show the user what's
there and ask whether to leave it, update it, or add the missing ecosystem alongside it. Never
silently overwrite an existing workflow file.

### 3. Ask about score thresholds

Both actions support optional inputs that fail the CI run if a scored package falls outside a
threshold:
- `fail-on-general` — fails if any package's `general_score` is below this (0–100)
- `fail-on-automation` — fails if any package's `automation_score` is below this (0–100)
- `fail-on-risk` — fails if any package's `risk_score` is *above* this (0–100; higher risk_score
  means riskier, so this one is inverted relative to the other two)

Ask the user directly whether they want any of these set, and at what values, before generating the
workflow. Explain that leaving them all unset makes the action purely informational — it posts a PR
comment with the scores but never fails the build — which is a legitimate choice, not just a
placeholder state.

### 4. Write the workflow file(s) locally

For npm, write `.github/workflows/audit-dependencies.yml`:

```yaml
name: Audit Dependencies

on:
  pull_request:
    branches: [main]

permissions:
  pull-requests: write

jobs:
  audit:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: packagerating/audit-dependencies@v1
        with:
          api-key: ${{ secrets.PACKAGERATING_API_KEY }}
```

For Python, write `.github/workflows/audit-dependencies-python.yml`:

```yaml
name: Audit Dependencies (Python)

on:
  pull_request:
    branches: [main]

permissions:
  pull-requests: write

jobs:
  audit:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: packagerating/audit-dependencies-python@v1
        with:
          api-key: ${{ secrets.PACKAGERATING_API_KEY }}
```

If the user chose thresholds in step 3, add the corresponding `fail-on-general`/
`fail-on-automation`/`fail-on-risk` lines under `with:` in the relevant file(s), using the exact
input names above — never invented names, never approximated values.

If the repo's default branch isn't `main`, adjust the `branches: [main]` line to match. Write both
files if both ecosystems were detected/confirmed in step 1 — this is the normal, expected case for a
monorepo with both a `package.json` and a `requirements.txt`, not a rare edge case needing special
handling.

Stop here — do not stage, commit, push, or open a PR. Tell the user the file(s) are ready to
review, and that committing/pushing is their call (or a separate explicit request to Claude).

### 5. Guide API key + secret setup

If the user doesn't already have a packagerating.com API key, point them to
`https://packagerating.com/signup` — free for open-source projects, 100 requests/day, no credit
card required. Once they have a key, give them this exact command to run themselves:

```
gh secret set PACKAGERATING_API_KEY --repo <owner>/<repo>
```

Let them choose how to supply the value (interactive prompt, `--body`, or stdin) — don't prescribe
one. **Never ask the user to paste their API key into the conversation, and never run `gh secret
set` on their behalf.** This is a permanent design decision for this skill, not a per-session
judgment call: the raw key value should never pass through Claude at any point in this workflow.

### 6. Summarize next steps

Confirm what was written (which file(s), which thresholds if any), remind the user that
committing/pushing the workflow file(s) is their call, and mention that threshold flags — if
skipped in step 3 — can be added later by editing the generated file directly.

## Example

**User:** "add packagerating to this repo"

1. Detects `package.json` at the repo root, no Python manifest. Reports: "Found package.json → I'll
   set up the npm action (packagerating/audit-dependencies). No Python manifest found, so I'll skip
   the Python action." User confirms.
2. Checks `.github/workflows/` — nothing existing references packagerating's actions.
3. Asks: "Want me to set score thresholds that fail the build (fail-on-general,
   fail-on-automation, fail-on-risk), or keep it informational-only for now (PR comment, never
   fails)?" User says informational-only for now.
4. Writes `.github/workflows/audit-dependencies.yml` with `on: pull_request`, `permissions:
   pull-requests: write`, and `api-key: ${{ secrets.PACKAGERATING_API_KEY }}` — no `fail-on-*`
   inputs.
5. "You'll need a packagerating.com API key. If you don't have one, sign up free at
   https://packagerating.com/signup (no card needed for open-source projects, 100 req/day). Once
   you have a key, run: `gh secret set PACKAGERATING_API_KEY --repo your-org/your-repo` — I won't
   ask you to paste the key here."
6. "Written `.github/workflows/audit-dependencies.yml`. Review it, then commit/push whenever
   you're ready — I won't do that automatically. You can add fail-on-* thresholds later by editing
   the file directly."
