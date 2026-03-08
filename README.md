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

### Option A — Via Claude Code plugin system

Add this repo as a marketplace, then install the skill:

```
/plugin marketplace add TongHuaLabs/metabase-dashboard-skill
/plugin install metabase-dashboard@metabase-dashboard-skill
```

### Option B — Manual

Copy the skill files into your project:

```bash
mkdir -p .claude/skills/metabase-dashboard/references
curl -fsSL https://raw.githubusercontent.com/TongHuaLabs/metabase-dashboard-skill/main/skills/metabase-dashboard/SKILL.md \
  -o .claude/skills/metabase-dashboard/SKILL.md
curl -fsSL https://raw.githubusercontent.com/TongHuaLabs/metabase-dashboard-skill/main/skills/metabase-dashboard/references/design-guide.md \
  -o .claude/skills/metabase-dashboard/references/design-guide.md
curl -fsSL https://raw.githubusercontent.com/TongHuaLabs/metabase-dashboard-skill/main/skills/metabase-dashboard/references/sql-patterns.md \
  -o .claude/skills/metabase-dashboard/references/sql-patterns.md
```

## Usage

Trigger the skill by asking Claude to build a Metabase dashboard:

```
/metabase-dashboard
```

Or just describe what you want:
> "Build me an e-commerce dashboard in Metabase showing revenue KPIs and trends"

## Files

- `skills/metabase-dashboard/SKILL.md` — Main skill instructions loaded by Claude Code
- `skills/metabase-dashboard/references/design-guide.md` — Chart type selection, layout rules, KPI best practices
- `skills/metabase-dashboard/references/sql-patterns.md` — SQL dialect quirks (H2/PostgreSQL/MySQL), filter syntax, JOIN patterns
- `.claude-plugin/marketplace.json` — Plugin marketplace metadata

## License

MIT
