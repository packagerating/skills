# GitHub PR Dependency Review Skill Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:executing-plans (recommended for this
> plan) to implement task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.
>
> **Do NOT use subagent-driven-development for this plan.** Unlike a typical code plan, Tasks 2-6
> here are skill-creator's own bespoke, continuous, interactive workflow — it dispatches parallel
> subagents itself, requires live user confirmation of test prompts, opens a browser-based review
> viewer, and reads back a feedback file — all within one unbroken session, the same way
> `package-adoption`'s eval loop ran. Splitting that across fresh, context-blind subagents per task
> would break the process. Execute this plan inline, in the current session.

**Goal:** Build and validate the `github-pr-dependency-review` skill — an interactive, conversational
skill that reviews what a specific GitHub PR's diff changed in its npm/PyPI dependencies, using the
`gh` CLI and the packagerating MCP server's `get_package` tool.

**Architecture:** A single `SKILL.md` at `skills/github-pr-dependency-review/SKILL.md` in the
`packagerating/skills` repo (no bundled scripts — see spec's rejected-alternatives section),
authored and validated the same way `package-adoption` was: hand-drafted first, then run through
Anthropic's `skill-creator` plugin's eval loop (test prompts, parallel with-skill/baseline subagent
runs, grading against assertions, a benchmark comparison, and iteration on user feedback) rather than
the standard code-writing pipeline.

**Tech Stack:** Markdown (`SKILL.md`), the `gh` CLI (already authenticated in this environment), the
`packagerating` MCP server (`get_package`, `list_packages`, `request_crawl` — already registered and
connected in this environment), Anthropic's `skill-creator` plugin (already installed).

## Global Constraints

- Spec: `/Users/marcelo/dev/packagerating/skills/docs/superpowers/specs/2026-07-17-github-pr-dependency-review-skill-design.md`
- Diff-only scope: never a full manifest audit — only added/removed/version-bumped dependencies
  visible in `gh pr diff`.
- Manifest files only (`package.json`, `requirements.txt`, `pyproject.toml`, `Pipfile`, etc.) — never
  lockfiles.
- `devDependencies` / dev-only Python deps are skipped by default.
- Ecosystems: npm (`language: javascript`) and PyPI (`language: python`) only — whatever
  `get_package` can score.
- No bundled parser script — the skill reads diff hunks directly.
- Conversational output only — this skill never posts a PR comment or otherwise writes to GitHub.
- No recommend/avoid verdict, no hardcoded score threshold — same convention as `package-adoption`.
- Inherits `package-adoption`'s once-per-session `axios` methodology self-check (six dimension keys:
  `liveness`, `community`, `security`, `dependency`, `versioning`, `dep_risk`).

---

### Task 1: Draft `SKILL.md`

**Files:**
- Create: `/Users/marcelo/dev/packagerating/skills/skills/github-pr-dependency-review/SKILL.md`

**Interfaces:**
- Consumes: nothing from earlier tasks (first task).
- Produces: the skill file itself, referenced by name (`github-pr-dependency-review`) in every
  later task's eval/grading work.

- [ ] **Step 1: Write the file**

Create `/Users/marcelo/dev/packagerating/skills/skills/github-pr-dependency-review/SKILL.md` with
exactly this content:

````markdown
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
````

- [ ] **Step 2: Verify the file is valid markdown with correct frontmatter**

Run: `head -5 /Users/marcelo/dev/packagerating/skills/skills/github-pr-dependency-review/SKILL.md`
Expected output starts with:
```
---
name: github-pr-dependency-review
description: "Use this skill when a user asks Claude to review, check, or audit a specific GitHub pull request's dependency changes interactively...
```

- [ ] **Step 3: Commit**

```bash
cd /Users/marcelo/dev/packagerating/skills
git add skills/github-pr-dependency-review/SKILL.md
git commit -m "wip: draft github-pr-dependency-review SKILL.md (pre-eval, via skill-creator)"
```

---

### Task 2: Write and confirm eval test cases

**Files:**
- Create: `/Users/marcelo/dev/packagerating/skills/skills/github-pr-dependency-review/evals/evals.json`

**Interfaces:**
- Consumes: the skill path from Task 1 (`skills/github-pr-dependency-review/SKILL.md`).
- Produces: `evals.json` with 3 test prompts, used by Task 3's subagent dispatch.

- [ ] **Step 1: Propose 3 test cases to the user**

Use skill-creator's own process (invoke the `skill-creator` skill for this and every remaining task
in this plan — it is not optional scaffolding, it *is* the implementation process for Tasks 2-6).
Propose these 3 test prompts, chosen to cover: an added dependency, a version bump, and a mix of
added/removed/bumped in one PR — the three cases the spec's Workflow section distinguishes:

1. `"check this PR's dependency changes"` (no PR named — exercises `gh pr view` fallback to current
   branch context)
2. `"review the dependency changes in PR #<N>"` (explicit PR number — exercises direct targeting;
   substitute a real open PR number from a real repo with a manifest-touching diff when running
   this, e.g. an open PR in `packagerating/mcp-server` or `package-rating` itself if one exists at
   execution time, or construct/use a small scratch repo+PR for a controlled test)
3. `"did this PR remove any dependencies, and is the replacement any good?"` (a phrasing that
   implies both a removal and an addition/bump in the same PR, without naming a PR — exercises the
   removed+added/bumped combination and the current-branch fallback together)

Confirm these with the user (adjust based on their feedback) before proceeding, per skill-creator's
own process — do not skip this confirmation step.

- [ ] **Step 2: Save to evals.json**

Once confirmed, save to `/Users/marcelo/dev/packagerating/skills/skills/github-pr-dependency-review/evals/evals.json`:

```json
{
  "skill_name": "github-pr-dependency-review",
  "evals": [
    {
      "id": 0,
      "name": "<derived from confirmed prompt 1>",
      "prompt": "<confirmed prompt 1>",
      "expected_output": "Resolves the PR from gh pr view (no PR named), pulls gh pr diff filtered to manifest files, identifies changed dependencies, scores added/bumped ones via get_package, presents a no-verdict report.",
      "files": []
    },
    {
      "id": 1,
      "name": "<derived from confirmed prompt 2>",
      "prompt": "<confirmed prompt 2>",
      "expected_output": "Uses the explicitly named PR rather than gh pr view, shows a version-bump before/after comparison for whichever dependency changed version, no verdict issued.",
      "files": []
    },
    {
      "id": 2,
      "name": "<derived from confirmed prompt 3>",
      "prompt": "<confirmed prompt 3>",
      "expected_output": "Correctly separates removed dependencies (acknowledged by name, not scored) from added/bumped ones (scored via get_package), no verdict issued.",
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
git add skills/github-pr-dependency-review/evals/evals.json
git commit -m "test: add eval prompts for github-pr-dependency-review skill"
```

---

### Task 3: Run the with-skill/baseline eval loop (iteration 1)

**Files:**
- Create: `/Users/marcelo/dev/packagerating/skills/skills/github-pr-dependency-review-workspace/iteration-1/` (directory tree — eval run outputs, gitignored, same pattern as `package-adoption-workspace/`)

**Interfaces:**
- Consumes: `SKILL.md` (Task 1), `evals.json` (Task 2).
- Produces: `iteration-1/eval-<name>/{with_skill,without_skill}/run-1/{outputs/response.md,outputs/transcript_notes.md,grading.json,timing.json}` for each of the 3 evals, plus `iteration-1/benchmark.json` / `benchmark.md`.

- [ ] **Step 1: Add `.gitignore` entry (if not already present)**

Check whether `/Users/marcelo/dev/packagerating/skills/.gitignore` already ignores
`skills/package-adoption-workspace/`. If it does, add the equivalent line for this skill:

```bash
cd /Users/marcelo/dev/packagerating/skills
grep -q "skills/github-pr-dependency-review-workspace/" .gitignore || \
  echo "skills/github-pr-dependency-review-workspace/" >> .gitignore
git add .gitignore
git commit -m "chore: ignore github-pr-dependency-review-workspace/ eval run artifacts"
```

- [ ] **Step 2: Create the run directory structure**

```bash
WS=/Users/marcelo/dev/packagerating/skills/skills/github-pr-dependency-review-workspace/iteration-1
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
"Step 1: Spawn all runs" process) — a with-skill run (told the skill path, told to read and follow
`SKILL.md`, and that `gh`/the packagerating MCP tools are available) and a baseline run (same
prompt, no skill path, no special instructions beyond "respond as you normally would"). Each
subagent writes its final response to
`$WS/eval-<name>/{with_skill,without_skill}/run-1/outputs/response.md` and a brief tool-call log to
`.../transcript_notes.md` in the same directory — mirror the exact subagent prompt structure used
for `package-adoption`'s eval loop (see that skill's session history / this org's established
convention if unclear).

- [ ] **Step 5: While runs are in progress, draft assertions**

For each eval, draft 4-5 objectively verifiable assertions and add them to both
`eval_metadata.json` and `evals.json`. At minimum, every eval's assertions must check:
- The correct manifest-filtered `gh pr diff` was used (not a full-repo audit).
- `get_package` was called with the correct `language` for every added/bumped dependency.
- Removed dependencies are named but not scored.
- No recommend/avoid verdict or hardcoded threshold appears in the response.
- The once-per-session `axios` self-check ran (dimensions: `liveness`, `community`, `security`,
  `dependency`, `versioning`, `dep_risk`).

Add eval-specific assertions on top of these (e.g. for the version-bump eval, an assertion that
both old and new version were looked up and a before/after delta is shown).

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

For each run, read its `response.md` and `transcript_notes.md`, evaluate each assertion from Step 5
against the actual output (PASS/FAIL with evidence), and write
`$WS/eval-<name>/{with_skill,without_skill}/run-1/grading.json`:

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
  /Users/marcelo/dev/packagerating/skills/skills/github-pr-dependency-review-workspace/iteration-1 \
  --skill-name github-pr-dependency-review \
  --skill-path /Users/marcelo/dev/packagerating/skills/skills/github-pr-dependency-review
```

Expected: `Generated: .../iteration-1/benchmark.json` and `.../iteration-1/benchmark.md`, with a
summary showing a pass-rate delta between `With Skill` and `Without Skill`.

---

### Task 4: Review with the user and iterate

**Files:**
- Modify: `skills/github-pr-dependency-review/SKILL.md` (only if feedback requires changes)
- Create: `skills/github-pr-dependency-review-workspace/iteration-2/` (only if iterating)

**Interfaces:**
- Consumes: `iteration-1/benchmark.json` (Task 3).
- Produces: either a confirmed-good skill (if no changes needed) or an `iteration-2` re-run with an
  updated `SKILL.md`.

- [ ] **Step 1: Launch the eval viewer**

```bash
cd /Users/marcelo/.claude/plugins/cache/claude-plugins-official/skill-creator/unknown/skills/skill-creator
nohup python3 eval-viewer/generate_review.py \
  /Users/marcelo/dev/packagerating/skills/skills/github-pr-dependency-review-workspace/iteration-1 \
  --skill-name "github-pr-dependency-review" \
  --benchmark /Users/marcelo/dev/packagerating/skills/skills/github-pr-dependency-review-workspace/iteration-1/benchmark.json \
  > /tmp/eval_viewer_github-pr-dependency-review.log 2>&1 &
```

Note the PID for later cleanup. Tell the user it's open and ask them to review the Outputs and
Benchmark tabs, leave feedback, and let you know when done.

- [ ] **Step 2: Read feedback once the user confirms they're done**

```bash
cat /Users/marcelo/dev/packagerating/skills/skills/github-pr-dependency-review-workspace/iteration-1/feedback.json
```

If `reviews` is empty or all feedback fields are blank, treat it as "no changes needed" — confirm
this reading with the user before proceeding (do not assume silently, per this org's established
practice of checking rather than guessing when a review file could mean either "nothing submitted"
or "everything looked fine").

- [ ] **Step 3: Kill the viewer**

```bash
kill $VIEWER_PID 2>/dev/null
```

- [ ] **Step 4: If feedback requires changes, revise `SKILL.md` and re-run**

Apply the specific feedback to `SKILL.md`. Then repeat Task 3's Steps 2-8 into a new
`iteration-2/` directory — with-skill runs only need to be re-run against the updated skill;
baseline runs are unaffected by a skill change and can either be re-run fresh or the `without_skill`
run directories copied over from `iteration-1` (matching how `package-adoption`'s iteration 2
carried baselines forward unchanged). Run
`python3 -m scripts.aggregate_benchmark .../iteration-2 --skill-name github-pr-dependency-review`
and launch the viewer again with `--previous-workspace .../iteration-1` so the user can compare.
Repeat this step until the user is satisfied or feedback is empty.

- [ ] **Step 5: Commit the final skill**

```bash
cd /Users/marcelo/dev/packagerating/skills
git add skills/github-pr-dependency-review/SKILL.md skills/github-pr-dependency-review/evals/evals.json
git commit -m "github-pr-dependency-review: fix <summary of what iteration found and fixed, or 'no changes needed after eval review' if none>"
git push
```

---

### Task 5: Offer description-trigger optimization (optional, only if user wants it)

**Files:**
- Modify: `skills/github-pr-dependency-review/SKILL.md` frontmatter `description` field only
  (if the user proceeds and a better description is found)

**Interfaces:**
- Consumes: the final `SKILL.md` from Task 4.
- Produces: an optimized `description` field, or no change if the current one already tests as
  optimal (this is a realistic outcome — it's exactly what happened for `package-adoption`, whose
  description survived unchanged after a full optimization run).

- [ ] **Step 1: Ask the user whether to run this now or skip it**

This is optional and was explicitly the last step in `package-adoption`'s process, run only after
the user confirmed satisfaction with the skill's behavior. Do not run it automatically — ask first.

- [ ] **Step 2: If yes, generate 20 trigger-eval queries**

10 should-trigger (varied phrasing: casual, explicit PR reference, implicit "this PR" from context,
competing with adjacent domains like general code review) and 10 should-not-trigger (near-misses:
questions about a PR's non-dependency changes, questions about dependency *adoption* — which should
route to `package-adoption` instead — full-repo audit requests that belong to the GitHub Actions,
etc.). Present them to the user via `skill-creator`'s `assets/eval_review.html` template (same
process as `package-adoption`'s optimization step) for review/editing before running anything.

- [ ] **Step 3: Run the optimization loop**

```bash
cd /Users/marcelo/.claude/plugins/cache/claude-plugins-official/skill-creator/unknown/skills/skill-creator
python3 -m scripts.run_loop \
  --eval-set <path-to-user-approved-trigger-eval.json> \
  --skill-path /Users/marcelo/dev/packagerating/skills/skills/github-pr-dependency-review \
  --model claude-sonnet-5 \
  --max-iterations 5 \
  --verbose
```

- [ ] **Step 4: Apply the result and report to the user**

Read the loop's output JSON for `best_description`. If it differs from the current frontmatter
description, update `SKILL.md` and show the user a before/after diff with the train/test scores. If
it's identical to the current description (ties default to the original), tell the user plainly
that optimization found no improvement — do not silently apply a no-op change.

- [ ] **Step 5: Commit if changed**

```bash
cd /Users/marcelo/dev/packagerating/skills
git add skills/github-pr-dependency-review/SKILL.md
git commit -m "github-pr-dependency-review: optimize trigger description (train X/Y, test A/B)"
git push
```

If nothing changed, skip this step — no empty commit.

---

## Self-Review Notes

- **Spec coverage:** PR resolution (Task 1 Step 1, Workflow §1) ✓. Manifest-filtered diff extraction
  (§2) ✓. No-bundled-parser diff reading (§3) ✓. Scoring with old/new version lookup for bumps (§4)
  ✓. No-verdict presentation (§5) ✓. Once-per-session self-check (§6) ✓. Conversational-only output
  (Output section) ✓. All four Error Handling bullets are covered inline in the drafted `SKILL.md`
  (gh missing/unauthenticated, no PR resolvable, no manifest changes, unresolvable package name,
  crawl-timeout handling) ✓. Ecosystem/dev-dependency/lockfile scope constraints are stated in
  Global Constraints and reflected in the drafted `SKILL.md`'s Workflow §2-3 ✓.
- **Placeholder scan:** Task 2's `evals.json` template intentionally contains `<...>` placeholders
  for the user-confirmed prompt text, since that confirmation happens live during execution and
  can't be predetermined in the plan — this is the one legitimate exception, explicitly flagged as
  "replace before saving," not a plan gap.
- **Type/name consistency:** `github-pr-dependency-review` used consistently as the skill directory
  name, `skill_name` in `evals.json`, and the `--skill-name` flag value across all tasks.
