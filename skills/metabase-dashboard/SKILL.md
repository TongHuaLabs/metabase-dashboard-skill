---
name: metabase-dashboard
description: Build fully functional Metabase dashboards end-to-end — schema exploration, dashboard design with best practices, card/chart creation, layout arrangement, and date/category filter wiring. Use when a user wants to create, design, or implement a dashboard in Metabase, turn database tables into visualizations, or add filters to an existing dashboard. Handles the complete workflow from raw database to polished interactive dashboard. Requires the metabase-mcp-server MCP server (github.com/CognitionAI/metabase-mcp-server) to be configured and connected.
---

# Metabase Dashboard Builder

Build data-rich Metabase dashboards end-to-end: explore schema → design → implement charts → wire filters.

## Workflow Overview

1. **Pre-flight** — verify MCP access and target collection
2. **Explore** — discover schema, relationships, key metrics
3. **Design** — propose dashboard layout and get approval before building
4. **Implement** — create cards, dashboard, and layout
5. **Filters** — add date/category filters and wire to cards

## Phase 1: Pre-flight

Confirm MCP tools are available: `list_databases`, `execute_query`, `create_card`, `create_dashboard`, `add_dashboard_filter`, `update_dashboard_cards`.

Check collection permissions:
```
list_collections → find target collection → confirm can_write: true
```

If `can_write: false`, ask the user to grant write access to the collection or choose a writable one.

Confirm target database ID via `list_databases`.

## Phase 2: Schema Exploration

Sample key tables to understand the data shape:
```sql
SELECT * FROM <table> LIMIT 3
```

Build a mental model before designing:
- **Fact tables**: orders, events, transactions — rows = individual occurrences
- **Dimension tables**: products, users, categories — rows = entities
- **Relationships**: foreign keys linking facts to dimensions
- **Time columns**: identify the primary date column for time-series and date filters

Validate every SQL query with `execute_query` before creating a card. This catches syntax errors, wrong column names, and empty results early.

See [sql-patterns.md](sql-patterns.md) for database-specific SQL syntax (H2, PostgreSQL, MySQL).

## Phase 3: Dashboard Design

**Always propose the full dashboard design before writing any code.** Present it in this format and wait for approval:

```
Dashboard: <Name>
Description: <one-line purpose>
Collection: <target collection>

Row 1 — KPI Scorecards (full 24 cols)
  [6 cols] Total Revenue       | [6 cols] Total Orders
  [6 cols] Avg Order Value     | [6 cols] Active Customers

Row 2 — Revenue Trends (24 cols)
  [12 cols] Monthly Revenue (line) | [12 cols] Revenue by Category (bar)

Row 3 — Customer Insights (24 cols)
  [10 cols] Age Distribution (bar) | [14 cols] Orders Over Time (line)

Row 4 — Detail Table (24 cols)
  [24 cols] Top 15 Products by Revenue (table)

Filters: Order Date (date range, applied to all ORDERS-based cards)
```

See [design-guide.md](design-guide.md) for chart type selection, layout rules, and KPI best practices.

## Phase 4: Card Implementation

### Creating Cards

For each card:
1. Write and test SQL with `execute_query`
2. Create with `create_card(name, dataset_query, display, visualization_settings)`

Common `display` values: `scalar`, `bar`, `line`, `pie`, `table`, `row`

**If filters will be added**, include the optional clause in every card that queries the filtered table:
```sql
WHERE 1=1 [[AND {{order_date}}]]
```

**For JOINs with filters**, wrap the filtered table in a subquery to avoid ambiguous column errors:
```sql
FROM (SELECT * FROM ORDERS WHERE 1=1 [[AND {{order_date}}]]) o
JOIN PRODUCTS p ON o.PRODUCT_ID = p.ID
```

See [sql-patterns.md](sql-patterns.md) for metric SQL patterns and dialect differences.

### Building the Dashboard

```python
create_dashboard(name, collection_id, description)
```

Add cards using `add_card_to_dashboard(dashboard_id, card_id, col, row, size_x, size_y)`.

**24-column grid rules:**
- Columns per row must sum to ≤ 24
- KPI row: 4–6 equal-width scalar cards (e.g., 4×6 cols or 3×8 cols)
- Chart pairs: 12+12 (even split) or 10+14, 8+16 (asymmetric)
- Full-width: 24 cols for wide tables or hero charts
- Row heights: scalars = 2–3; charts = 4–6; tables = 5–8

## Phase 5: Filter Implementation

### Step 1 — Discover the Field ID

Find the internal Metabase field ID for the column to filter on:
```sql
-- H2 (Metabase sample DB)
SELECT COLUMN_NAME FROM INFORMATION_SCHEMA.COLUMNS
WHERE TABLE_NAME = 'ORDERS'
```
Or inspect via `get_card` on an existing card — result_metadata includes field IDs for used columns.

### Step 2 — Update Each Card with the Template Tag

Use `update_card` to add `template-tags` to the `dataset_query.native` of every card that needs the filter. Generate one UUID and reuse it across all cards for the same filter variable:

```json
{
  "order_date": {
    "id": "<uuid-generate-once>",
    "name": "order_date",
    "display-name": "Order Date",
    "type": "dimension",
    "dimension": ["field", <field_id>, null],
    "widget-type": "date/all-options"
  }
}
```

Only update cards whose SQL contains `[[AND {{order_date}}]]`. Skip unrelated cards (e.g., a People-only card when filtering Orders).

### Step 3 — Add the Parameter to the Dashboard

```python
add_dashboard_filter(dashboard_id, parameters=[{
  "id": "order-date-filter-001",
  "name": "Order Date",
  "slug": "order_date",
  "type": "date/all-options",
  "sectionId": "date"
}])
```

For category/text filters use `"type": "string/="`. For numeric filters use `"type": "number/="`.

### Step 4 — Wire Cards to the Filter

Fetch current dashcard IDs and positions via `get_dashboard`, then call `update_dashboard_cards` with the **full** cards array (include every dashcard, even unfiltered ones with `parameter_mappings: []`):

```python
update_dashboard_cards(dashboard_id, cards=[
  {
    "id": <dashcard_id>,   # dashcard ID from get_dashboard, NOT card ID
    "card_id": <card_id>,
    "row": <row>, "col": <col>, "size_x": <w>, "size_y": <h>,
    "parameter_mappings": [{
      "parameter_id": "order-date-filter-001",
      "card_id": <card_id>,
      "target": ["dimension", ["template-tag", "order_date"]]
    }]
  },
  # ... all other dashcards with parameter_mappings: [] if not filtered
])
```

## Reference Files

- [design-guide.md](design-guide.md) — Chart type selection, layout patterns, KPI best practices, naming conventions, section design
- [sql-patterns.md](sql-patterns.md) — SQL dialect quirks (H2/PostgreSQL/MySQL), filter syntax, JOIN patterns, common metric formulas
