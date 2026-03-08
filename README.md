# Metabase Dashboard Skill for Claude Code

A Claude Code skill for building fully functional Metabase dashboards end-to-end — from schema exploration to dashboard design, chart creation, layout arrangement, and filter wiring.

## What it does

When you invoke this skill, Claude will guide you through a structured workflow:

1. **Pre-flight** — verifies MCP access and target collection
2. **Schema Exploration** — discovers tables, relationships, and key metrics
3. **Dashboard Design** — proposes a full layout for your approval before building
4. **Card Implementation** — creates SQL-backed charts and KPI cards
5. **Filter Wiring** — adds date/category filters and connects them to the right cards

## Requirements

- [Claude Code](https://claude.ai/code) with the [Metabase MCP server](https://github.com/CognitionAI/metabase-mcp-server) configured
- A running Metabase instance

## Installation

Install via Claude Code:

```
/plugins install metabase-dashboard-skill
```

Or copy `SKILL.md` and the `references/` folder into your project's `.claude/skills/metabase-dashboard/` directory.

## Usage

Trigger the skill by asking Claude to build a Metabase dashboard:

```
/metabase-dashboard
```

Or just describe what you want:
> "Build me an e-commerce dashboard in Metabase showing revenue KPIs and trends"

## Files

- `SKILL.md` — Main skill instructions loaded by Claude Code
- `references/design-guide.md` — Chart type selection, layout rules, KPI best practices
- `references/sql-patterns.md` — SQL dialect quirks (H2/PostgreSQL/MySQL), filter syntax, JOIN patterns

## License

MIT
