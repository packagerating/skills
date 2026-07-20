# packagerating-setup Skill Design

## Overview

A new Claude skill, `packagerating-setup`, that helps a user integrate their GitHub repo with
packagerating.com's automated dependency scoring — adding the `packagerating/audit-dependencies`
(npm) and/or `packagerating/audit-dependencies-python` GitHub Action(s), configuring the API key
secret, and optionally setting score thresholds that fail the build.

This fills a real gap: `github-pr-dependency-review` explicitly assumes the automated GitHub
Action is already running in the repo ("not a substitute for the automated
audit-dependencies/audit-dependencies-python GitHub Actions, which already score every dependency
in a PR's final manifest on every push"). Nothing currently helps a user get from zero to that
automated setup existing. `package-adoption` is also unrelated — it's about evaluating one
candidate package, not repo-wide CI setup.

Only npm and Python are in scope, matching what actually exists today: `packagerating/
audit-dependencies` (npm) and `packagerating/audit-dependencies-python` (requirements.txt/Poetry/
Pipenv/uv). packagerating.com's scoring API also covers crates.io and rubygems.org, but no GitHub
Action exists yet for either — this skill only sets up what can actually be set up.

## Trigger

The user asks to set up, add, or integrate packagerating's dependency scoring/CI into a repo —
phrases like "add packagerating to this repo," "set up dependency scoring in CI," "integrate with
packagerating.com," or "add the audit-dependencies action." Should trigger even if the user doesn't
name the action repos explicitly, the same way the other two skills trigger on intent rather than
exact terminology.

## Flow

### 1. Confirm tools exist

Requires the `gh` CLI, authenticated, and to be running inside a git repository with a GitHub
remote. If either is missing, say so plainly and stop — do not guess or improvise a manual
alternative, matching `package-adoption`'s and `github-pr-dependency-review`'s existing
"confirm the tools exist" pattern.

### 2. Detect ecosystem(s)

Look for `package.json` (npm) and `requirements.txt`/`pyproject.toml`/`Pipfile` (Python) at the
repo root (and, loosely, in common subdirectories — monorepo detection doesn't need to be
exhaustive here, since each action's own `audit-workspaces`/`audit-subprojects` defaults already
handle monorepo discovery once the workflow is running). Report what was found and what's planned
("Found package.json → I'll set up the npm action. No Python manifest found, skipping that one.")
and wait for confirmation before writing anything. If detection finds nothing recognizable, ask the
user directly rather than guessing.

### 3. Check for an existing workflow

Search `.github/workflows/*.yml` for any file already referencing `packagerating/
audit-dependencies` or `packagerating/audit-dependencies-python`. If found, show the user what's
there and ask whether to leave it as-is, update it, or add the missing ecosystem alongside it —
never silently overwrite an existing workflow file.

### 4. Ask about score thresholds

Both actions support optional `fail-on-general` / `fail-on-automation` / `fail-on-risk` inputs that
fail the CI run if a scored package falls outside the given threshold (0–100 for general/
automation; risk is inverted — higher risk_score means riskier, so `fail-on-risk` fails if a
package's risk score is *above* the given value). Proactively ask the user whether they want these
set, and at what values, before generating the workflow — don't default to skipping this
conversation. Explain that leaving them unset makes the action purely informational (PR comment
only, never fails the build), which is a legitimate choice too.

### 5. Write the workflow file(s) locally

Generate `.github/workflows/audit-dependencies.yml` and/or `.github/workflows/
audit-dependencies-python.yml` (using separate files when both ecosystems apply, matching this
project's own convention of one workflow file per concern) with:
- `on: pull_request` (matching both actions' documented usage)
- `permissions: pull-requests: write` (required for the PR comment feature)
- `api-key: ${{ secrets.PACKAGERATING_API_KEY }}`
- Any `fail-on-*` inputs the user chose in step 4
- The exact input names/shapes from each action's real README — never invented or approximated
  input names

Stop here. Do not stage, commit, push, or open a PR — that's a separate, explicit step the user
(or a follow-up instruction to Claude) initiates. Tell the user the file(s) are ready to review.

### 6. Guide API key + secret setup

If the user doesn't already have a packagerating.com API key, point them to
`https://packagerating.com/signup` (free for open-source projects, 100 requests/day, no credit
card). Once they have a key, give them the exact command to set it themselves:

```
gh secret set PACKAGERATING_API_KEY --repo <owner>/<repo>
```

(prompts for the value interactively, or accepts `--body`/stdin — let the user choose how they
want to supply it). The raw key value never passes through Claude at any point in this skill —
this is a deliberate, permanent design decision (see Rejected Alternative below), not a
per-session judgment call.

### 7. Summarize next steps

Confirm what was written, remind the user committing/pushing is their call, and mention that
threshold flags (if skipped in step 4) can be added later by editing the generated workflow file
directly.

## Rejected Alternative: Claude runs `gh secret set` on the user's behalf

Considered having Claude prompt the user to paste their API key and run `gh secret set` directly,
mirroring how this session handled a one-off npm publish token. Rejected because a skill is reused
across many future sessions by many different users, not a single supervised interaction — the
npm-token case had a human directly watching one specific command execute once. Keeping the raw
key value out of Claude's hands entirely, every time, is the safer default for something this skill
will do repeatedly and unsupervised.

## Edge Cases

- **Neither ecosystem detected**: ask the user directly which language(s)/action(s) they want,
  rather than concluding there's nothing to do — a repo might use a build system or vendoring
  approach the skill's file-based detection doesn't recognize.
- **Both ecosystems present**: run through steps 3–6 for each independently (separate threshold
  questions, separate files) — a monorepo with both a `package.json` and a `requirements.txt` is a
  real, expected case, not a rare one.
- **`gh` not authenticated**: caught by step 1; stop and tell the user to run `gh auth login`.
- **Not a git repo / no GitHub remote**: caught by step 1; stop and explain why.

## Testing

Built the same way as `package-adoption` and `github-pr-dependency-review`: draft `SKILL.md`,
propose 2–3 realistic test prompts, run the skill-creator eval loop (with-skill vs. baseline
subagent dispatch) before considering it done.

## Self-Review

- **Placeholder scan**: none — every step has a concrete, decided behavior; no "TBD" or deferred
  decisions.
- **Internal consistency**: step 5 (no commit/push) and step 6 (never handle raw secret value) are
  both instances of the same underlying principle — this skill produces artifacts and instructions,
  not side effects on the user's repo or credentials — stated consistently rather than as two
  unrelated rules.
- **Scope check**: single skill, two ecosystems, appropriately sized for one implementation plan —
  not decomposed further. The two ecosystems share nearly all of their flow (steps 1, 3–7 are
  identical in shape for either), so one skill covering both is more coherent than two nearly
  duplicate skills.
- **Ambiguity check**: "auto-detect, confirm before acting" is made concrete in step 2 with an
  example of what the confirmation message actually says, rather than left as an abstract
  principle a future reader could interpret multiple ways.
