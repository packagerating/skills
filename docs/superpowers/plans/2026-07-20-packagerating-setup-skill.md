# packagerating-setup Skill Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:executing-plans (recommended for this
> plan) to implement task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.
>
> **Do NOT use subagent-driven-development for this plan.** Unlike a typical code plan, Tasks 2-6
> here are skill-creator's own bespoke, continuous, interactive workflow — it dispatches parallel
> subagents itself, requires live user confirmation of test prompts, opens a browser-based review
> viewer, and reads back a feedback file — all within one unbroken session, the same way
> `package-adoption`'s and `github-pr-dependency-review`'s eval loops ran. Splitting that across
> fresh, context-blind subagents per task would break the process. Execute this plan inline, in the
> current session.

**Goal:** Build and validate the `packagerating-setup` skill — a skill that helps a user integrate
their GitHub repo with packagerating.com's automated dependency scoring, by detecting which
ecosystem(s) apply, writing the right `packagerating/audit-dependencies` and/or
`packagerating/audit-dependencies-python` GitHub Actions workflow file(s), and guiding API key +
secret setup.

**Architecture:** A single `SKILL.md` at `skills/packagerating-setup/SKILL.md` in the
`packagerating/skills` repo (no bundled scripts), authored and validated the same way
`package-adoption` and `github-pr-dependency-review` were: hand-drafted first, then run through
Anthropic's `skill-creator` plugin's eval loop (test prompts, parallel with-skill/baseline subagent
runs, grading against assertions, a benchmark comparison, and iteration on user feedback).

**Tech Stack:** Markdown (`SKILL.md`), the `gh` CLI (already authenticated in this environment),
Anthropic's `skill-creator` plugin (already installed).

## Global Constraints

- Spec: `/Users/marcelo/dev/packagerating/skills/docs/superpowers/specs/2026-07-20-packagerating-setup-skill-design.md`
- Ecosystems in scope: npm (`packagerating/audit-dependencies`) and Python
  (`packagerating/audit-dependencies-python`) only — no Rust/Ruby action exists yet, so this skill
  never claims to set those up.
- Auto-detect ecosystem(s) from `package.json` / `requirements.txt`/`pyproject.toml`/`Pipfile`,
  report what was found, and confirm with the user before writing anything.
- Never silently overwrite an existing `packagerating/audit-dependencies*` workflow file — detect
  it, show it, ask how to proceed.
- Proactively ask about `fail-on-general`/`fail-on-automation`/`fail-on-risk` thresholds during
  setup, not as an afterthought.
- Write the workflow file(s) locally only — never stage, commit, push, or open a PR. That is
  always a separate, explicit step the user or a follow-up instruction initiates.
- Never handle the user's raw `PACKAGERATING_API_KEY` value — always give the user the exact
  `gh secret set PACKAGERATING_API_KEY --repo <owner>/<repo>` command to run themselves.
- Exact action input names/shapes must come from each action's real README, not be invented:
  - `packagerating/audit-dependencies@v1`: `api-key` (required), `package-json-path`,
    `packages`, `include-dev`, `include-optional`, `use-lockfile`, `audit-workspaces`,
    `audit-subprojects`, `subproject-max-depth`, `subproject-exclude`, `fail-on-general`,
    `fail-on-automation`, `fail-on-risk`, `pr-comment`, `github-token`, `crawl-timeout`.
  - `packagerating/audit-dependencies-python@v1`: `api-key` (required), `requirements-path`,
    `packages`, `audit-subprojects`, `subproject-max-depth`, `subproject-exclude`,
    `fail-on-general`, `fail-on-automation`, `fail-on-risk`, `pr-comment`, `github-token`,
    `crawl-timeout`.
  - Both require `permissions: pull-requests: write` and are documented as triggering on
    `pull_request`.

---

### Task 1: Draft `SKILL.md`

**Files:**
- Create: `/Users/marcelo/dev/packagerating/skills/skills/packagerating-setup/SKILL.md`

**Interfaces:**
- Consumes: nothing from earlier tasks (first task).
- Produces: the skill file itself, referenced by name (`packagerating-setup`) in every later task's
  eval/grading work.

- [ ] **Step 1: Write the file**

Create `/Users/marcelo/dev/packagerating/skills/skills/packagerating-setup/SKILL.md` with exactly
this content:

````markdown
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
````

- [ ] **Step 2: Verify the file is valid markdown with correct frontmatter**

Run: `head -5 /Users/marcelo/dev/packagerating/skills/skills/packagerating-setup/SKILL.md`
Expected output starts with:
```
---
name: packagerating-setup
description: "Use this skill when a user wants to add, set up, or integrate packagerating.com's automated dependency scoring into a GitHub repo's CI...
```

- [ ] **Step 3: Commit**

```bash
cd /Users/marcelo/dev/packagerating/skills
git add skills/packagerating-setup/SKILL.md
git commit -m "wip: draft packagerating-setup SKILL.md (pre-eval, via skill-creator)"
```

---

### Task 2: Write and confirm eval test cases

**Files:**
- Create: `/Users/marcelo/dev/packagerating/skills/skills/packagerating-setup/evals/evals.json`

**Interfaces:**
- Consumes: the skill path from Task 1 (`skills/packagerating-setup/SKILL.md`).
- Produces: `evals.json` with 3 test prompts, used by Task 3's subagent dispatch.

- [ ] **Step 1: Propose 3 test cases to the user**

Invoke the `skill-creator` skill for this and every remaining task in this plan — it is not
optional scaffolding, it *is* the implementation process for Tasks 2-6. Propose these 3 test
prompts, chosen to cover the three distinct paths the spec's Workflow section distinguishes
(npm-only, Python-only, and the existing-workflow/monorepo edge cases):

1. `"add packagerating to this repo"` (run against a scratch/test repo containing only a
   `package.json` — exercises the npm-only detection path and the no-existing-workflow case)
2. `"set up dependency scoring in CI for this project"` (run against a scratch/test repo containing
   only a `requirements.txt` — exercises the Python-only detection path, and phrasing that doesn't
   name "packagerating" or "audit-dependencies" explicitly, testing whether the skill still
   triggers and detects correctly)
3. `"integrate packagerating.com's dependency scoring — we already have some CI set up"` (run
   against a scratch/test repo containing both a `package.json` and a `requirements.txt`, plus an
   existing unrelated `.github/workflows/ci.yml` that does NOT reference packagerating — exercises
   the dual-ecosystem path and confirms the skill correctly treats an unrelated existing workflow
   file as irrelevant, rather than false-triggering the "existing packagerating workflow found"
   branch)

Confirm these with the user (adjust based on their feedback) before proceeding, per skill-creator's
own process — do not skip this confirmation step. Also confirm how the three scratch/test repos
will be made available to the eval subagents (e.g. small local scratch directories under
`/tmp` or this session's scratchpad, each pre-populated with the right manifest files, since these
tests need real files to detect against rather than a live GitHub repo).

- [ ] **Step 2: Save to evals.json**

Once confirmed, save to
`/Users/marcelo/dev/packagerating/skills/skills/packagerating-setup/evals/evals.json`:

```json
{
  "skill_name": "packagerating-setup",
  "evals": [
    {
      "id": 0,
      "name": "<derived from confirmed prompt 1>",
      "prompt": "<confirmed prompt 1>",
      "expected_output": "Detects package.json only, confirms with the user before writing, generates .github/workflows/audit-dependencies.yml with on: pull_request, permissions: pull-requests: write, and api-key: ${{ secrets.PACKAGERATING_API_KEY }}, asks about fail-on-* thresholds before writing, never commits/pushes, never asks for the raw API key value, points to packagerating.com/signup and gives the exact gh secret set command.",
      "files": []
    },
    {
      "id": 1,
      "name": "<derived from confirmed prompt 2>",
      "prompt": "<confirmed prompt 2>",
      "expected_output": "Triggers despite no explicit mention of packagerating/audit-dependencies by name, detects requirements.txt only, generates .github/workflows/audit-dependencies-python.yml with the correct action reference (packagerating/audit-dependencies-python@v1), same thresholds/secret/no-commit behavior as eval 0.",
      "files": []
    },
    {
      "id": 2,
      "name": "<derived from confirmed prompt 3>",
      "prompt": "<confirmed prompt 3>",
      "expected_output": "Detects both package.json and requirements.txt, writes both workflow files, correctly does NOT flag the pre-existing unrelated ci.yml as a conflicting packagerating workflow (since it doesn't reference packagerating/audit-dependencies), asks about thresholds for each ecosystem.",
      "files": []
    }
  ]
}
```

Replace the `<...>` placeholders with the actual confirmed prompt text and short kebab-case names
before saving — this file must contain the real confirmed prompts, not the bracketed placeholders.

- [ ] **Step 3: Commit**

```bash
cd /Users/marcelo/dev/packagerating/skills
git add skills/packagerating-setup/evals/evals.json
git commit -m "test: add eval prompts for packagerating-setup skill"
```

---

### Task 3: Run the with-skill/baseline eval loop (iteration 1)

**Files:**
- Create: `/Users/marcelo/dev/packagerating/skills/skills/packagerating-setup-workspace/iteration-1/` (directory tree — eval run outputs, gitignored, same pattern as `package-adoption-workspace/` and `github-pr-dependency-review-workspace/`)

**Interfaces:**
- Consumes: `SKILL.md` (Task 1), `evals.json` (Task 2).
- Produces: `iteration-1/eval-<name>/{with_skill,without_skill}/run-1/{outputs/response.md,outputs/transcript_notes.md,grading.json,timing.json}` for each of the 3 evals, plus `iteration-1/benchmark.json` / `benchmark.md`.

- [ ] **Step 1: Add `.gitignore` entry**

```bash
cd /Users/marcelo/dev/packagerating/skills
grep -q "skills/packagerating-setup-workspace/" .gitignore || \
  echo "skills/packagerating-setup-workspace/" >> .gitignore
git add .gitignore
git commit -m "chore: ignore packagerating-setup-workspace/ eval run artifacts"
```

- [ ] **Step 2: Create the run directory structure**

```bash
WS=/Users/marcelo/dev/packagerating/skills/skills/packagerating-setup-workspace/iteration-1
for eval_name in <eval-0-name> <eval-1-name> <eval-2-name>; do
  mkdir -p "$WS/eval-$eval_name/with_skill/run-1/outputs"
  mkdir -p "$WS/eval-$eval_name/without_skill/run-1/outputs"
done
```

Replace `<eval-0-name>` etc. with the actual `name` fields chosen in Task 2's `evals.json`.

- [ ] **Step 3: Write `eval_metadata.json` for each eval (assertions empty for now)**

For each of the 3 evals, write `$WS/eval-<name>/eval_metadata.json`:

```json
{
  "eval_id": 0,
  "eval_name": "<name>",
  "prompt": "<confirmed prompt text from Task 2>",
  "assertions": []
}
```

- [ ] **Step 4: Spawn 3 with-skill + 3 baseline subagents, in the same turn**

For each eval, dispatch two subagents in parallel (6 total in one batch, per skill-creator's own
"Step 1: Spawn all runs" process): a with-skill run (given the skill path, told to read and follow
`SKILL.md`, told `gh` is available and authenticated, and pointed at that eval's scratch test repo
directory) and a baseline run (same prompt, same scratch repo, no skill path, no special
instructions beyond "respond as you normally would"). Each subagent writes its final response to
`$WS/eval-<name>/{with_skill,without_skill}/run-1/outputs/response.md` and a brief tool-call log to
`.../transcript_notes.md` in the same directory, and must also copy any files it wrote (e.g. a
generated workflow YAML) into `.../outputs/` so grading can inspect the actual generated content,
not just the conversational response — mirror the exact subagent prompt structure used for
`package-adoption`'s and `github-pr-dependency-review`'s eval loops.

- [ ] **Step 5: While runs are in progress, draft assertions**

For each eval, draft 4-5 objectively verifiable assertions and add them to both
`eval_metadata.json` and `evals.json`. At minimum, every eval's assertions must check:
- The correct ecosystem(s) were detected and confirmed with the user before any file was written.
- The generated workflow file(s) reference the exact correct action (`packagerating/
  audit-dependencies@v1` or `packagerating/audit-dependencies-python@v1`), include `permissions:
  pull-requests: write`, and use `api-key: ${{ secrets.PACKAGERATING_API_KEY }}` verbatim.
- No file was staged, committed, pushed, or a PR opened.
- The user was never asked to paste their API key, and was given the exact `gh secret set
  PACKAGERATING_API_KEY --repo <owner>/<repo>` command.
- The response asked about `fail-on-*` thresholds before generating the workflow, not after.

Add eval-specific assertions on top of these (e.g. for eval 2, an assertion that the pre-existing
unrelated `ci.yml` was correctly not flagged as a conflicting packagerating workflow).

- [ ] **Step 6: Capture timing data as each subagent completes**

For each of the 6 runs, as its completion notification arrives, immediately write
`$WS/eval-<name>/{with_skill,without_skill}/run-1/timing.json`:

```json
{
  "total_tokens": <from notification>,
  "duration_ms": <from notification>,
  "total_duration_seconds": <duration_ms / 1000, one decimal>
}
```

- [ ] **Step 7: Grade each of the 6 runs**

For each run, read its `response.md`, `transcript_notes.md`, and any copied output files (e.g. the
generated workflow YAML), evaluate each assertion from Step 5 against the actual output (PASS/FAIL
with evidence), and write `$WS/eval-<name>/{with_skill,without_skill}/run-1/grading.json`:

```json
{
  "expectations": [
    {"text": "<assertion text>", "passed": true, "evidence": "<quote or description from the run>"}
  ],
  "summary": {"passed": <n>, "failed": <n>, "total": <n>, "pass_rate": <passed/total>},
  "timing": {"executor_duration_seconds": <from timing.json>},
  "eval_feedback": {"suggestions": [], "overall": "<brief note>"}
}
```

- [ ] **Step 8: Aggregate the benchmark**

```bash
cd /Users/marcelo/.claude/plugins/cache/claude-plugins-official/skill-creator/unknown/skills/skill-creator
python3 -m scripts.aggregate_benchmark \
  /Users/marcelo/dev/packagerating/skills/skills/packagerating-setup-workspace/iteration-1 \
  --skill-name packagerating-setup \
  --skill-path /Users/marcelo/dev/packagerating/skills/skills/packagerating-setup
```

Expected: `Generated: .../iteration-1/benchmark.json` and `.../iteration-1/benchmark.md`, with a
summary showing a pass-rate delta between `With Skill` and `Without Skill`.

---

### Task 4: Review with the user and iterate

**Files:**
- Modify: `skills/packagerating-setup/SKILL.md` (only if feedback requires changes)
- Create: `skills/packagerating-setup-workspace/iteration-2/` (only if iterating)

**Interfaces:**
- Consumes: `iteration-1/benchmark.json` (Task 3).
- Produces: either a confirmed-good skill (if no changes needed) or an `iteration-2` re-run with an
  updated `SKILL.md`.

- [ ] **Step 1: Launch the eval viewer**

```bash
cd /Users/marcelo/.claude/plugins/cache/claude-plugins-official/skill-creator/unknown/skills/skill-creator
nohup python3 eval-viewer/generate_review.py \
  /Users/marcelo/dev/packagerating/skills/skills/packagerating-setup-workspace/iteration-1 \
  --skill-name "packagerating-setup" \
  --benchmark /Users/marcelo/dev/packagerating/skills/skills/packagerating-setup-workspace/iteration-1/benchmark.json \
  > /tmp/eval_viewer_packagerating-setup.log 2>&1 &
```

Note the PID for later cleanup. Tell the user it's open and ask them to review the Outputs and
Benchmark tabs, leave feedback, and let you know when done.

- [ ] **Step 2: Read feedback once the user confirms they're done**

```bash
cat /Users/marcelo/dev/packagerating/skills/skills/packagerating-setup-workspace/iteration-1/feedback.json
```

If `reviews` is empty or all feedback fields are blank, treat it as "no changes needed" — confirm
this reading with the user before proceeding rather than assuming silently.

- [ ] **Step 3: Kill the viewer**

```bash
kill $VIEWER_PID 2>/dev/null
```

- [ ] **Step 4: If feedback requires changes, revise `SKILL.md` and re-run**

Apply the specific feedback to `SKILL.md`. Then repeat Task 3's Steps 2-8 into a new
`iteration-2/` directory — with-skill runs only need to be re-run against the updated skill;
baseline runs are unaffected by a skill change and can either be re-run fresh or the `without_skill`
run directories copied over from `iteration-1`. Run
`python3 -m scripts.aggregate_benchmark .../iteration-2 --skill-name packagerating-setup` and
launch the viewer again with `--previous-workspace .../iteration-1` so the user can compare. Repeat
this step until the user is satisfied or feedback is empty.

- [ ] **Step 5: Commit the final skill**

```bash
cd /Users/marcelo/dev/packagerating/skills
git add skills/packagerating-setup/SKILL.md skills/packagerating-setup/evals/evals.json
git commit -m "packagerating-setup: fix <summary of what iteration found and fixed, or 'no changes needed after eval review' if none>"
git push
```

---

### Task 5: Offer description-trigger optimization (optional, only if user wants it)

**Files:**
- Modify: `skills/packagerating-setup/SKILL.md` frontmatter `description` field only (if the user
  proceeds and a better description is found)

**Interfaces:**
- Consumes: the final `SKILL.md` from Task 4.
- Produces: an optimized `description` field, or no change if the current one already tests as
  optimal.

- [ ] **Step 1: Ask the user whether to run this now or skip it**

This is optional and only run after the user confirms satisfaction with the skill's behavior. Do
not run it automatically — ask first.

- [ ] **Step 2: If yes, generate 20 trigger-eval queries**

10 should-trigger (varied phrasing: casual, explicit action-repo naming, implicit setup intent) and
10 should-not-trigger (near-misses: a question about reviewing one specific PR's dependency changes
— which should route to `github-pr-dependency-review` — a question about whether to adopt one
candidate package — which should route to `package-adoption` — and general CI/CD questions
unrelated to packagerating). Present them to the user via `skill-creator`'s
`assets/eval_review.html` template for review/editing before running anything.

- [ ] **Step 3: Run the optimization loop**

```bash
cd /Users/marcelo/.claude/plugins/cache/claude-plugins-official/skill-creator/unknown/skills/skill-creator
python3 -m scripts.run_loop \
  --eval-set <path-to-user-approved-trigger-eval.json> \
  --skill-path /Users/marcelo/dev/packagerating/skills/skills/packagerating-setup \
  --model claude-sonnet-5 \
  --max-iterations 5 \
  --verbose
```

- [ ] **Step 4: Apply the result and report to the user**

Read the loop's output JSON for `best_description`. If it differs from the current frontmatter
description, update `SKILL.md` and show the user a before/after diff with the train/test scores. If
it's identical to the current description, tell the user plainly that optimization found no
improvement — do not silently apply a no-op change.

- [ ] **Step 5: Commit if changed**

```bash
cd /Users/marcelo/dev/packagerating/skills
git add skills/packagerating-setup/SKILL.md
git commit -m "packagerating-setup: optimize trigger description (train X/Y, test A/B)"
git push
```

If nothing changed, skip this step — no empty commit.

---

## Self-Review Notes

- **Spec coverage:** Confirm-tools-exist gate (Task 1's drafted `SKILL.md`, spec §"Before you
  start") ✓. Auto-detect + confirm before acting (Workflow §1) ✓. Existing-workflow detection,
  never silently overwrite (§2) ✓. Proactive threshold question (§3) ✓. Write-only, no
  commit/push/PR (§4) ✓. Never handle raw API key, exact `gh secret set` command (§5) ✓. Summary
  step (§6) ✓. Both action's exact input names carried into Global Constraints and the drafted
  `SKILL.md` verbatim ✓. Edge cases (neither ecosystem detected, both present, `gh` not
  authenticated, not a git repo) all covered inline in the drafted `SKILL.md`'s Workflow §1 and the
  "confirm tools exist" section ✓. Testing section (skill-creator eval loop) ✓.
- **Placeholder scan:** Task 2's `evals.json` template intentionally contains `<...>` placeholders
  for user-confirmed prompt text and eval names, since that confirmation happens live during
  execution — the same legitimate exception used in the `github-pr-dependency-review` plan,
  explicitly flagged as "replace before saving."
- **Type/name consistency:** `packagerating-setup` used consistently as the skill directory name,
  `skill_name` in `evals.json`, and the `--skill-name`/`--skill-path` flag values across all tasks.
  `PACKAGERATING_API_KEY` (the secret name) and the exact action references
  (`packagerating/audit-dependencies@v1`, `packagerating/audit-dependencies-python@v1`) are
  identical everywhere they appear — Global Constraints, the drafted `SKILL.md`, and the eval
  assertions in Task 3.
