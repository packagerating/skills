# packagerating/skills

A [Claude Code](https://claude.com/claude-code) plugin that gives Claude live, on-demand access to
[packagerating.com](https://packagerating.com) package health/risk scores — right inside your
coding session, not just in CI. Built on top of the
[packagerating MCP server](https://github.com/packagerating/mcp-server).

## Skills

| Skill | Use it when |
|---|---|
| [`package-adoption`](skills/package-adoption/SKILL.md) | Deciding whether to add a new npm or PyPI package — evaluating one candidate, comparing alternatives, or reacting to a new dependency in a diff. |
| [`github-pr-dependency-review`](skills/github-pr-dependency-review/SKILL.md) | Reviewing what a specific pull request changed in its dependencies — ad hoc, in-conversation, not a substitute for the automated `audit-dependencies`/`audit-dependencies-python` GitHub Actions, which already score every PR automatically. |
| [`packagerating-setup`](skills/packagerating-setup/SKILL.md) | Wiring up packagerating.com's automated dependency scoring into a repo's CI for the first time. |

Every skill pulls real, current data via the MCP server's tools — never from Claude's training
data, which goes stale the moment a package's maintenance status, security posture, or community
changes.

## Install

1. Get a free API key at [packagerating.com](https://packagerating.com).
2. Set up the [packagerating MCP server](https://github.com/packagerating/mcp-server#setup) — all
   three skills depend on its tools being available.
3. Add this plugin to Claude Code:

```
/plugin marketplace add packagerating/skills
/plugin install packagerating@packagerating-skills
```

That's it — the skills activate automatically whenever a task matches what they cover; you don't
need to invoke them by name.

## Related

- [`packagerating/mcp-server`](https://github.com/packagerating/mcp-server) — the MCP server these skills are built on
- [`packagerating/audit-dependencies`](https://github.com/packagerating/audit-dependencies) — GitHub Action, npm dependencies
- [`packagerating/audit-dependencies-python`](https://github.com/packagerating/audit-dependencies-python) — GitHub Action, Python dependencies
