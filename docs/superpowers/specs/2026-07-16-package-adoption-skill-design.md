# Package Adoption Skill

## Problem

`packagerating/mcp-server` (merged 2026-07-16) gives Claude live tool access to packagerating.com
scores — `list_packages`, `get_package`, `request_crawl` — but nothing yet teaches Claude *when*
and *how* to reach for those tools during the single most common moment they'd matter: a developer
is about to add a new dependency and wants to know if it's a good choice, and how it stacks up
against realistic alternatives.

Without a skill, using the MCP tools well for this purpose depends on Claude reinventing the same
judgment call in every conversation — which package name to look up, which alternatives are worth
comparing, how to present the comparison usefully. A skill turns that judgment call into a
documented, repeatable workflow.

This is the first of the skills captured in `packagerating/mcp-server`'s design spec
(`docs/superpowers/specs/2026-07-15-mcp-server-design.md`, "Full Scope — Saved for Later" section)
as the roadmap the MCP server unlocks. It is explicitly distinct from the also-planned
**dependency-audit skill** (walking an *existing* repo's full manifest and scoring every current
dependency) — this skill is about a single *adoption decision*, before a package is added.

## Design

### Distribution

A new public repo, `packagerating/skills`, alongside `audit-dependencies`,
`audit-dependencies-python`, and `mcp-server` in the org — mirroring Anthropic's own
`anthropics/skills` repo (which holds many skills, e.g. `pptx`) rather than bundling this into
`mcp-server` itself. This gives a natural home for the also-planned dependency-audit skill (and any
future skills) without needing a new repo per skill, and keeps "this is an MCP server" and "this is
a skill collection" as separate concerns.

### Structure

A single `SKILL.md` at `skills/package-adoption/SKILL.md` (matching the `anthropics/skills`
layout — `skills/<name>/SKILL.md`), authored with Anthropic's own `skill-creator` skill
(already installed in this environment) rather than hand-written — `skill-creator` covers drafting
the frontmatter `description` for trigger accuracy, writing test prompts, and running qualitative/
quantitative evals to iterate, none of which the general-purpose subagent-driven-development
pipeline used for code in this org covers. No bundled scripts are needed — unlike `pptx`, all the
actual work here is calling three already-built MCP tools and presenting results, not local file
manipulation.

### Hard dependency on `packagerating/mcp-server`

This skill assumes `list_packages`/`get_package`/`request_crawl` are available as tools in the
calling session (i.e., the user has `packagerating/mcp-server` configured in their MCP client). If
they're not available, the skill should say so plainly and point at the MCP server's README/setup
instructions rather than silently failing or guessing at package quality from training data alone.

### Workflow

1. **Identify the candidate package** — from what the user names directly, or from what's visibly
   being added in a diff/`package.json`/`requirements.txt` in the current context.
2. **Name 2-3 realistic alternatives** using Claude's own knowledge (e.g., evaluating `moment` →
   knows to consider `dayjs`, `date-fns`, `luxon`). `list_packages` cannot semantically search for
   "alternatives to X" — it only sorts/filters what's already scored — so alternative-discovery is
   Claude's own judgment call, not a new tool capability. This works today with the MCP server
   exactly as built.
3. **Call `get_package`** for the candidate and each alternative, language-aware (npm vs. PyPI,
   matching the `language` parameter already on that tool).
4. **Present a comparison** — composite scores (General/Automation/Risk) and the six dimensions,
   laid out so differences are easy to scan.
5. **No baked-in verdict.** The skill presents data, not a recommend/avoid judgment with hardcoded
   thresholds — consistent with the MCP server's own "thin mirror, no curated opinions" design
   philosophy (see its spec's "Out of Scope" section). Any qualitative framing ("axios has more
   advisories but 10x the downloads") states facts the user should weigh, not a threshold-driven
   pass/fail. This was an explicit, deliberate choice during brainstorming, not a default.

### Self-check for methodology drift

Carried forward from `packagerating/mcp-server`'s design spec ("Self-aware / self-updating
components" section): once per session (not on every invocation — wasteful), the skill calls
`get_package` on a fixed reference package and compares the returned `dimensions` object's keys
against the six the skill's own instructions describe (`liveness`, `community`, `security`,
`dependency`, `versioning`, `dep_risk`). If they've diverged — packagerating.com added, removed, or
renamed a scoring dimension since this skill was written — the skill surfaces a clear warning that
its methodology understanding may be stale, rather than silently presenting data using outdated
dimension names as if they were still current.

## Out of Scope

- **The dependency-audit skill** (full-repo manifest scan) — separate, not-yet-brainstormed piece
  of the same roadmap.
- **A "should I adopt this" verdict/scoring threshold** — deliberately left as raw data
  presentation, not a policy decision baked into the skill.
- **Alternative-discovery via a new MCP tool or search capability** — relies entirely on Claude's
  own knowledge, per the brainstorming decision; no `search_similar_packages`-style tool is being
  added to `mcp-server` for this.
- **A GitHub-integration skill for reviewing PR dependency changes** — a separate, also-planned
  roadmap item (item 3 in `mcp-server`'s "Full Scope — Saved for Later"), not this skill.

## Testing

Per `skill-creator`'s own process (not the code-testing conventions used elsewhere in this org):
write a handful of test prompts (e.g. "should I add `moment` to my project?", "is `requests`
better than `httpx` for a new Python service?") and run them against a live session with
`packagerating/mcp-server` actually configured, evaluating qualitatively whether the skill
triggers correctly, names sensible alternatives, and presents a clear comparison — then iterate on
the `SKILL.md` and its trigger `description` using `skill-creator`'s eval-viewer/description-
optimizer tooling. This replaces the plan-and-subagent-implement flow used for the MCP server
itself, since there's no application code here to unit test in the same sense.

## Files Touched

New repository — everything is new:

| File | Purpose |
|---|---|
| `skills/package-adoption/SKILL.md` | The skill itself |
| `docs/superpowers/specs/2026-07-16-package-adoption-skill-design.md` | This spec |
