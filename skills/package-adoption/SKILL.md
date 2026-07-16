---
name: package-adoption
description: "Use this skill whenever a user is deciding whether to add a new npm or PyPI package to a project — evaluating a specific package, comparing it against alternatives, or asking things like \"should I use X\", \"is X or Y better\", \"what's a good library for...\", or when a new dependency shows up in a diff, package.json, or requirements.txt the user is working on. Trigger even if the user doesn't say \"packagerating\" or \"score\" by name — any dependency-adoption decision for a JavaScript/npm or Python/PyPI package qualifies. Requires the packagerating MCP server (list_packages, get_package, request_crawl tools) to be configured; if those tools aren't available, say so and point to https://github.com/packagerating/mcp-server instead of guessing from training data alone."
---

# Package Adoption

Help a developer decide whether to adopt a candidate npm or PyPI package by pulling its live
health/risk score from packagerating.com and comparing it against realistic alternatives — not by
relying on your own training data, which goes stale the moment a package's maintenance status,
security posture, or community changes.

## Before you start: confirm the tools exist

This skill depends on three MCP tools — `list_packages`, `get_package`, `request_crawl` — provided
by the `packagerating` MCP server. If they're not in your available tools, don't try to fake the
workflow from memory. Tell the user the packagerating MCP server isn't configured and point them
to https://github.com/packagerating/mcp-server to set it up, then stop.

## Workflow

### 1. Identify the candidate package

Usually obvious from what the user said, or from a dependency visible in the diff/`package.json`/
`requirements.txt` you're already looking at. If it's ambiguous (e.g., they said "a date library"
with no name), ask which specific package, or treat their description as the starting point for
step 2 instead of a single candidate.

### 2. Name realistic alternatives

Before calling any tool, think about 2-3 packages a developer would genuinely consider instead of
the candidate — from your own knowledge of the ecosystem, not from `list_packages` (it sorts and
filters what's already scored, it can't semantically search "alternatives to X"). Aim for
alternatives that solve the same problem in a comparably idiomatic way, not just anything
tangentially related. If the user already named alternatives themselves, use those instead of
guessing.

If you genuinely don't know of good alternatives (an unusual or highly specific package), it's
fine to evaluate the candidate alone and say so — don't invent alternatives just to fill the slot.

### 3. Look up scores

Call `get_package` for the candidate and each alternative. Pass `language: "javascript"` for npm
packages, `language: "python"` for PyPI packages — get this right, since a name collision across
registries would silently score the wrong package. A first-ever lookup on an obscure package can
take up to a minute (the tool waits out a live crawl internally) — that's normal, not a hang.

If `get_package` comes back `still_crawling` even after that wait, say so plainly and offer to
retry rather than presenting incomplete data as if it were final.

### 4. Present the comparison — data, not a verdict

Lay out the candidate and its alternatives side by side: the three composite scores (General,
Automation, Risk) and the six underlying dimensions (Liveness, Community, Security, Dependency,
Versioning, Dep-tree risk). A table works well for 2-4 packages. For each dimension you present,
ground it in the actual numbers you got back — don't round scores into vague labels like "good" or
"bad" without showing the number that led you there.

**Do not issue a recommend/avoid verdict or apply a hardcoded score threshold.** Different projects
weigh these dimensions differently — a hobby project might not care that a package's `versioning`
score is low, while a compliance-conscious team might care a great deal. Your job is to make the
tradeoffs legible, not to make the decision for the user. It's fine — encouraged, even — to point
out a specific fact worth their attention ("axios has more open advisories but ships weekly and has
10x the download count of the alternative"), as long as it's a factual observation tied to the
data, not a disguised verdict.

This applies just as much when the "alternative" isn't another scored package but a language
built-in (e.g. `String.prototype.padStart` instead of `left-pad`) — it's tempting to treat "just use
the built-in" as obviously correct and slide into "so don't add this dependency," but that's still a
verdict. Whether the convenience of a maintained package is worth one more dependency is exactly the
kind of judgment call this skill leaves to the user; present the tradeoff (dependency weight and
supply-chain surface vs. a few lines of code you'd otherwise own) and stop there.

### 5. Self-check for methodology drift (once per session)

The first time you use this skill in a session, call `get_package` for `axios` (language
`javascript`) as a reference point, and check that its `dimensions` object has exactly these six
keys: `liveness`, `community`, `security`, `dependency`, `versioning`, `dep_risk`. If any are
missing, renamed, or if new ones have appeared, packagerating.com's scoring methodology has
changed since this skill was written — tell the user plainly that this skill's understanding of the
dimensions may be out of date, rather than silently presenting data under stale dimension names.

You only need to do this once per session — don't repeat it on every subsequent package lookup in
the same conversation.

## Example

**User:** "I need a date library for a new Node project, was going to use moment"

1. Candidate: `moment` (javascript). Alternatives: `dayjs`, `date-fns`, `luxon` — all common
   moment replacements.
2. `get_package` for all four.
3. Present:

| Package | General | Automation | Risk | Liveness | Community | Security | Dependency | Versioning | Dep-tree |
|---|---|---|---|---|---|---|---|---|---|
| moment | 58.1 | 51.3 | 62.0 | 20 | 95 | 70 | 55 | 60 | 80 |
| dayjs | 89.8 | 91.2 | 92.2 | 100 | 88 | 90 | 95 | 85 | 100 |
| date-fns | 84.6 | 87.0 | 88.5 | 95 | 82 | 85 | 90 | 80 | 95 |
| luxon | 79.2 | 80.1 | 83.0 | 85 | 70 | 88 | 82 | 75 | 90 |

4. "`moment` scores notably lower on Liveness (20) — it's in maintenance mode, no new features,
   though it still gets security patches. `dayjs` and `date-fns` both score well across the board;
   `dayjs` has a smaller bundle size if that matters for your project, `date-fns` has a more
   functional/tree-shakeable API. Your call based on what you need."
