# GitHub PR Dependency Review Skill

## Problem

`packagerating/audit-dependencies` and `packagerating/audit-dependencies-python` already score every
dependency in a PR's final manifest automatically, on every push, posting a PR comment. That's the
right tool for continuous, no-human-in-the-loop governance — but it doesn't help someone who's
already *in* a Claude session and wants to ask, conversationally, "what did this PR actually change
in its dependencies, and is any of it risky?" — without leaving the conversation, and without
duplicating what the Action already reports.

This is roadmap item 3 from `packagerating/mcp-server`'s design spec
(`docs/superpowers/specs/2026-07-15-mcp-server-design.md`, "Full Scope — Saved for Later"):
"A GitHub-integration skill for ad hoc, interactive review... composing this MCP server with the
`gh` CLI... explicitly complementary to, not a duplicate of, the two existing GitHub Actions."

## Design

### Distribution

Lives alongside `package-adoption` in `packagerating/skills`, at
`skills/github-pr-dependency-review/SKILL.md` — same repo, same `skill-creator`-authored workflow,
same eval-loop process already established for this org's skills.

### Scope: diff only, not a full manifest audit

The single distinguishing decision that keeps this complementary rather than redundant: this skill
looks only at what a PR's diff actually **changed** in its dependency manifests — added packages,
removed packages, version bumps — not a full audit of every dependency in the PR's final state.
A full-manifest, on-demand re-run already exists in the form of the Action itself (triggered by any
push); this skill answers a different, narrower question a human reviewer actually has: "what's new
or different here."

Manifest files only — `package.json`, `requirements.txt`, `pyproject.toml`, `Pipfile`, and other
formats already recognized by `audit-dependencies`/`audit-dependencies-python` as manifests.
Lockfile-only changes (a transitive dependency bumped with no manifest change) are explicitly out of
scope — lockfile diff formats vary too much across npm/yarn/pnpm/poetry to parse reliably from raw
diff text without a bundled parser, which this skill deliberately doesn't carry (see below).
`devDependencies` and dev-only Python dependencies are skipped by default, matching
`audit-dependencies`' own `include-dev: false` default — the skill can still discuss them if the
user explicitly asks, just doesn't surface them unprompted.

### Ecosystem scope

Same as `package-adoption` and the underlying MCP server: npm (`language: javascript`) and PyPI
(`language: python`) only — whatever `get_package` can actually score. A manifest change for an
unsupported ecosystem simply isn't picked up.

### Workflow

1. **Resolve the target PR.** If the user names one (number, URL, or clearly means "this PR" in
   context), use that. Otherwise fall back to `gh pr view` for whatever repo/branch the user is
   currently in. If neither resolves to a PR, say so and ask.
2. **Pull the diff.** `gh pr diff <PR>`, filtered to recognized manifest file paths.
3. **Read the diff directly — no bundled parser.** Claude reads the `+`/`-` hunks itself to
   identify added dependencies, removed dependencies, and version bumps (old version → new version
   for the same package name). This mirrors `package-adoption`'s no-bundled-scripts simplicity;
   manifest diff hunks for `package.json`/`requirements.txt`-style files are regular enough
   (one dependency per line, in most formats) to read reliably without custom tooling. A bundled
   parser was considered and rejected for a first version — see "Approaches Considered" in the
   brainstorming transcript for the full reasoning; the short version is it adds real build/
   maintenance surface for edge cases (reordered deps, multi-line entries) that a first version can
   defer.
4. **Score what changed.** For each added or version-bumped dependency: `get_package` with the
   correct `language`. For a version bump, look up both the old and new version (via `get_package`'s
   `version` param) when both have been crawled, so the report shows what actually moved, not just
   the new version's score in isolation. Removed dependencies are acknowledged by name, not scored
   — there's nothing to look up.
5. **Present — no verdict, no pass/fail.** Same convention as `package-adoption`: composite scores
   plus all six dimensions, grounded in the actual numbers. Added dependencies get a normal
   comparison-style table row. Version bumps show a before/after delta on whichever dimensions
   moved. This skill does not apply a hardcoded threshold or tell the user to merge/block — same
   reasoning as `package-adoption`: different teams weigh risk differently, and that's not a
   judgment call this skill should make for them.
6. **Methodology self-check, once per session.** Inherits `package-adoption`'s drift check: the
   first time this skill runs in a session, call `get_package` for `axios` (language `javascript`)
   and confirm the `dimensions` object still has exactly the six expected keys. Both skills call the
   same MCP tools against the same live API, so they carry the same staleness risk if
   packagerating.com's scoring methodology changes — if one skill already ran the check this
   session, no need to repeat it, same as today.

### Output: conversational only, never posts to GitHub

This skill never creates or updates a PR comment. Findings are reported directly in the
conversation. Two reasons: posting to GitHub is a side-effectful, visible-to-others action that
shouldn't be a skill default without explicit per-use confirmation (consistent with how this
environment treats posting/commenting generally), and a second automated commenter would compete
with the existing Action's own PR comment rather than complementing it. If a user wants the findings
posted somewhere, they can ask for that separately and explicitly in the moment — not something this
skill's instructions bake in.

### Error handling

- `gh` not installed or not authenticated → clear message pointing to `gh auth login`, not a raw CLI
  error.
- No PR resolvable (nothing named, not on a PR branch) → say so, ask which PR.
- PR has no manifest file changes at all → say so plainly; don't force a report out of nothing.
- A changed dependency's name can't be resolved via `get_package` (private/scoped package not on the
  public registry, typo, etc.) → note it by name and move on; don't let one unresolvable package
  block the rest of the report.
- Same `get_package` crawl-timeout handling as `package-adoption`: if a lookup comes back
  `still_crawling`, say so plainly rather than presenting partial data as final.

## Out of Scope (this build)

- **Lockfile-only changes.** Explicitly deferred — see "Scope" above.
- **Posting PR comments.** Explicitly deferred — see "Output" above.
- **A bundled diff-parsing script.** Explicitly deferred — see "Workflow" step 3 above.
- **Full-manifest audit mode.** Considered and rejected during brainstorming (option: "diff-only,
  but let the user ask for a full audit when they want it") — the user's explicit call was diff-only
  only, keeping the skill's one job clear rather than growing it back toward what the Action already
  does.
