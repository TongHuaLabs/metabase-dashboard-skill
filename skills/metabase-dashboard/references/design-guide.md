# Dashboard Design Guide

## Table of Contents
1. [Dashboard Anatomy](#dashboard-anatomy)
2. [KPI Selection](#kpi-selection)
3. [Chart Type Selection](#chart-type-selection)
4. [Layout Patterns](#layout-patterns)
5. [Filter Design](#filter-design)
6. [Naming & Descriptions](#naming--descriptions)
7. [Common Dashboard Templates](#common-dashboard-templates)

---

## Dashboard Anatomy

Every good dashboard has a clear structure:

```
┌─────────────────────────────────────┐
│  KPI Row — 4–6 key numbers at a     │  ← Always first
│  glance (Total Revenue, Orders, etc) │    Row height: 2–3
├──────────────────┬──────────────────┤
│  Trend Chart     │  Breakdown Chart  │  ← Primary insights
│  (line/bar)      │  (bar/pie)        │    Row height: 4–6
├──────────────────┴──────────────────┤
│  Secondary Charts / Distributions   │  ← Supporting context
│                                     │    Row height: 4–6
├─────────────────────────────────────┤
│  Detail Table (sortable, full width) │  ← Drill-down last
└─────────────────────────────────────┘    Row height: 5–8
```

**Key principle**: Top = summary (answer "how are we doing?"), bottom = detail (answer "why?").

---

## KPI Selection

### What makes a good KPI card
- A single number that answers a key business question
- Comparable to a benchmark (period over period, target, industry average)
- Actionable — something someone can act on

### Common KPI categories by domain

**E-commerce / Sales:**
- Total Revenue, Total Orders, Avg Order Value, Revenue Today/This Month
- Return Rate, Discount Rate, Gross Margin
- New Customers, Repeat Customer Rate, Customer Lifetime Value

**Product / SaaS:**
- Active Users (DAU/MAU), New Signups, Churn Rate, MRR/ARR
- Feature Adoption Rate, Session Duration, Retention

**Operations:**
- Tickets Open/Resolved, Avg Resolution Time, SLA Compliance
- Inventory Levels, Fulfillment Rate, Defect Rate

**Marketing:**
- Impressions, Clicks, CTR, Conversion Rate, Cost per Lead
- Campaign ROI, Email Open Rate

### KPI row layout
Use equal-width cards. For 4 KPIs: `size_x=6` each. For 3: `size_x=8`. For 6: `size_x=4`.

---

## Chart Type Selection

| Data pattern | Best chart | Metabase display |
|---|---|---|
| Single aggregate value | Number/Scalar | `scalar` |
| Trend over time | Line chart | `line` |
| Comparison of categories | Bar chart (horizontal for many items) | `bar` or `row` |
| Part-to-whole (≤6 categories) | Pie/Donut | `pie` |
| Distribution / histogram | Bar chart | `bar` |
| Two metrics over time | Line (dual axis) | `line` |
| Ranked list with value | Bar or Table | `row` or `table` |
| Detailed records / top N | Table | `table` |
| Geographic data | Map (if lat/lon or region) | `map` |
| Progress toward goal | Progress bar | `progress` |

### When NOT to use pie charts
- More than 5–6 categories
- Values are close to each other (hard to compare slices)
- Showing change over time

### Line vs Bar for time series
- **Line**: Continuous trend, emphasizes trajectory (monthly revenue over a year)
- **Bar**: Discrete comparisons, emphasizes individual periods (sales per quarter)

---

## Layout Patterns

### 24-column grid reference

| Cards per row | size_x each | Use case |
|---|---|---|
| 1 | 24 | Full-width table, hero chart |
| 2 | 12 | Even split — two related charts |
| 2 (asymmetric) | 10 + 14 | Small chart + wide chart |
| 2 (asymmetric) | 8 + 16 | Small metric + wide detail |
| 3 | 8 | Three equal charts |
| 4 | 6 | KPI row (4 metrics) |
| 6 | 4 | Dense KPI row (6 metrics) |

### Grouping sections visually
Place related charts in the same row. Users read left-to-right, top-to-bottom:
- Row 1: "How are we doing overall?" (KPIs)
- Row 2: "What does the trend look like?" (time series)
- Row 3: "What's driving it?" (breakdowns, distributions)
- Row 4+: "Who/what specifically?" (tables, rankings)

### Row heights
- `size_y: 2` — Scalar KPI cards
- `size_y: 3` — KPI with a sparkline
- `size_y: 4` — Compact charts
- `size_y: 5` — Standard charts
- `size_y: 6` — Charts needing more vertical space (maps, distributions)
- `size_y: 7–8` — Large tables

---

## Filter Design

### Which filters to add

**Always useful:**
- Date range on the primary time column (orders, events, signups)

**Add when the dimension is high-value:**
- Category / segment (product category, customer tier, region)
- Status (order status, user status)
- Specific entity (product ID, user ID — use as a drill-down filter)

### Filter placement
Filters appear at the top of the dashboard. Use descriptive names:
- "Order Date" not "date"
- "Product Category" not "category"
- "Customer Region" not "region"

### Which cards to wire
Only connect a filter to cards that actually query the filtered table. Wiring a filter to an unrelated card causes SQL errors or unexpected empty results:
- Date filter on ORDERS → only wire to cards querying ORDERS
- If a card queries PEOPLE for demographics → do NOT wire the Orders date filter to it

---

## Naming & Descriptions

### Dashboard names
Descriptive, audience-aware:
- "E-Commerce Overview" — for ops/exec team
- "Customer Intelligence & Sales Behavior" — for sales/marketing
- "Product Performance" — for product team

### Card names
State the insight clearly:
- "Monthly Revenue" not "revenue_by_month"
- "Top 10 Products by Revenue" not "products"
- "Repeat Customer Rate (%)" — include unit when ambiguous
- "Avg Order Value (USD)" — include currency if multi-currency system

### Avoid
- SQL column names as card names (TOTAL_REVENUE_SUM)
- Abbreviations your audience won't know (AOV is fine for e-commerce, not for HR)
- Redundant context that's obvious from the dashboard name

---

## Common Dashboard Templates

### E-Commerce Overview
- KPIs: Total Revenue, Orders, AOV, New Customers
- Charts: Revenue over time, Revenue by category, Top products (table), Returns rate
- Filters: Order date

### Customer Intelligence
- KPIs: Repeat rate, Avg orders/customer, Total discounts, % discounted orders
- Charts: Geographic revenue map, Age distribution, Order frequency histogram, Discounted vs regular AOV over time
- Filters: Order date

### SaaS / Product
- KPIs: MAU, New signups, Churn rate, MRR
- Charts: DAU/MAU ratio over time, Feature adoption by cohort, Retention curve, Top power users (table)
- Filters: Signup date, Plan type

### Operations / Support
- KPIs: Open tickets, Avg resolution time, SLA compliance %, Customer satisfaction
- Charts: Tickets by category, Volume over time, Agent performance (table), P1/P2 breakdown
- Filters: Created date, Priority, Assignee
