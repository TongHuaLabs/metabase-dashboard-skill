# SQL Patterns for Metabase

## Table of Contents
1. [dataset_query Format: pMBQL vs Legacy](#dataset_query-format-pmbql-vs-legacy)
2. [Identify Your SQL Dialect](#identify-your-sql-dialect)
3. [Filter Syntax](#filter-syntax)
4. [Common Metric Patterns](#common-metric-patterns)
5. [H2 Specific Patterns](#h2-specific-patterns)
6. [PostgreSQL Patterns](#postgresql-patterns)
7. [MySQL / MariaDB Patterns](#mysql--mariadb-patterns)
8. [JOIN Patterns with Filters](#join-patterns-with-filters)
9. [Common Pitfalls](#common-pitfalls)

---

## dataset_query Format: pMBQL vs Legacy

Metabase v0.58+ uses **pMBQL** internally. Passing the old legacy format to `create_card` or `update_card` causes the query to silently save as `{}`, breaking the card with no error.

### How to detect which format to use

Before creating any card, call `get_card` on one existing card and inspect its `dataset_query`:

- If it has a `"stages"` key → **pMBQL** (v0.58+)
- If it has a `"native"` key at the top level → **legacy**

Always match the format of the existing cards in the instance.

### Legacy format (pre-v0.58)

```json
{
  "type": "native",
  "database": 1,
  "native": {
    "query": "SELECT COUNT(*) FROM ORDERS",
    "template-tags": {}
  }
}
```

### pMBQL format (v0.58+)

```json
{
  "lib/type": "mbql/query",
  "database": 1,
  "stages": [
    {
      "lib/type": "mbql.stage/native",
      "native": "SELECT COUNT(*) FROM ORDERS",
      "template-tags": {}
    }
  ]
}
```

### pMBQL with a filter template-tag

```json
{
  "lib/type": "mbql/query",
  "database": 1,
  "stages": [
    {
      "lib/type": "mbql.stage/native",
      "native": "SELECT COUNT(*) FROM ORDERS WHERE 1=1 [[AND {{order_date}}]]",
      "template-tags": {
        "order_date": {
          "id": "<uuid>",
          "name": "order_date",
          "display-name": "Order Date",
          "type": "dimension",
          "dimension": ["field", {"base-type": "type/DateTime", "lib/uuid": "<new-uuid>", "effective-type": "type/DateTime"}, 13],
          "widget-type": "date/all-options"
        }
      }
    }
  ]
}
```

### Finding the integer field ID for template-tag dimension

The third element of `dimension` (e.g. `13`) is Metabase's internal integer field ID — **not** a SQL column ID. It must be real; `null` fails validation.

To find it:
1. Call `get_card` on any card that queries the same table and has been executed (has `result_metadata`)
2. Check `dataset_query.stages[0].template-tags.<tag>.dimension[2]` for the field ID
3. Or check `field_ref` in `result_metadata` of an executed card

Known field IDs for the Metabase Sample Database:
| Table | Column | Field ID |
|---|---|---|
| ORDERS | CREATED_AT | 13 |

---

## Identify Your SQL Dialect

```python
list_databases()  # → check "engine" field per database
```

| Engine value | Dialect | Notes |
|---|---|---|
| `h2` | H2 (Java embedded) | Metabase sample database |
| `postgres` | PostgreSQL | Most common production DB |
| `mysql` | MySQL / MariaDB | Common in web apps |
| `bigquery` | BigQuery | Google Cloud DWH |
| `snowflake` | Snowflake | Cloud DWH |
| `redshift` | Amazon Redshift | AWS DWH |

---

## Filter Syntax

### Optional filter clause
Wrap the filter variable in `[[ ]]` so the query returns all data when no filter is selected:

```sql
SELECT COUNT(*) FROM ORDERS
WHERE 1=1 [[AND {{order_date}}]]
```

`WHERE 1=1` is required as a base condition so `[[AND ...]]` can be appended safely.

### Template tag definition (in dataset_query.native.template-tags)

```json
{
  "order_date": {
    "id": "<uuid>",
    "name": "order_date",
    "display-name": "Order Date",
    "type": "dimension",
    "dimension": ["field", <field_id>, null],
    "widget-type": "date/all-options"
  }
}
```

### Filter widget types

| widget-type | UI control | Use for |
|---|---|---|
| `date/all-options` | Date picker (relative + absolute + range) | Date columns |
| `category` | Dropdown with values | Low-cardinality text columns |
| `string/=` | Text input | String equality |
| `number/=` | Number input | Numeric equality |
| `number/between` | Range input | Numeric range |

### Multiple filters in one query
Each filter gets its own template tag and its own `[[AND ...]]` clause:
```sql
WHERE 1=1
  [[AND {{order_date}}]]
  [[AND {{category}}]]
```

---

## Common Metric Patterns

### Aggregates
```sql
-- Total revenue
SELECT ROUND(SUM(TOTAL), 2) AS total_revenue FROM ORDERS

-- Order count
SELECT COUNT(*) AS total_orders FROM ORDERS

-- Average order value
SELECT ROUND(AVG(TOTAL), 2) AS avg_order_value FROM ORDERS

-- Distinct count
SELECT COUNT(DISTINCT USER_ID) AS unique_customers FROM ORDERS
```

### Rates and percentages
```sql
-- Percentage: CAST to float before dividing to avoid integer truncation
SELECT ROUND(
  CAST(COUNT(DISTINCT USER_ID) AS FLOAT) / (SELECT COUNT(*) FROM PEOPLE) * 100,
  1
) AS customer_rate FROM ORDERS

-- Conditional rate (% orders with discount)
SELECT ROUND(
  CAST(SUM(CASE WHEN DISCOUNT > 0 THEN 1 ELSE 0 END) AS FLOAT) / COUNT(*) * 100,
  1
) AS pct_discounted FROM ORDERS
```

### Time series (monthly)
Group by formatted date to get ordered monthly buckets:
```sql
-- PostgreSQL / MySQL
SELECT DATE_TRUNC('month', CREATED_AT) AS order_month,
       ROUND(SUM(TOTAL), 2) AS revenue
FROM ORDERS
GROUP BY 1
ORDER BY 1

-- H2
SELECT FORMATDATETIME(CREATED_AT, 'yyyy-MM') AS order_month,
       ROUND(SUM(TOTAL), 2) AS revenue
FROM ORDERS
GROUP BY order_month
ORDER BY order_month
```

### Top N ranking
```sql
SELECT CATEGORY, ROUND(SUM(TOTAL), 2) AS revenue
FROM ORDERS o JOIN PRODUCTS p ON o.PRODUCT_ID = p.ID
GROUP BY CATEGORY
ORDER BY revenue DESC
LIMIT 10
```

### Cohort / subgroup comparison
```sql
SELECT
  CASE WHEN DISCOUNT > 0 THEN 'Discounted' ELSE 'No Discount' END AS segment,
  COUNT(*) AS orders,
  ROUND(AVG(TOTAL), 2) AS avg_value
FROM ORDERS
GROUP BY segment
```

---

## H2 Specific Patterns

H2 is the engine for Metabase's built-in sample database.

### Date formatting
```sql
-- Format date as year-month string
FORMATDATETIME(CREATED_AT, 'yyyy-MM')

-- Format date as year
FORMATDATETIME(CREATED_AT, 'yyyy')

-- Format date as full date
FORMATDATETIME(CREATED_AT, 'yyyy-MM-dd')
```

H2 does NOT support `DATE_FORMAT()` (MySQL) or `TO_CHAR()` (PostgreSQL).

### Date arithmetic
```sql
-- Days between two dates
DATEDIFF('DAY', CREATED_AT, CURRENT_DATE)

-- Subtract N days from today
DATEADD('DAY', -30, CURRENT_DATE)

-- Current timestamp
CURRENT_TIMESTAMP
```

### Type casting
```sql
CAST(value AS FLOAT)
CAST(value AS VARCHAR)
CAST(value AS INT)
```

### Reserved words as aliases
H2 rejects SQL reserved words as column aliases. Avoid:
- `month` → use `order_month`
- `year` → use `order_year`
- `date` → use `order_date`
- `time` → use `order_time`
- `value` → use `metric_value`

### INFORMATION_SCHEMA (H2)
```sql
-- List all tables
SELECT TABLE_NAME FROM INFORMATION_SCHEMA.TABLES
WHERE TABLE_SCHEMA = 'PUBLIC'

-- List columns in a table
SELECT COLUMN_NAME, DATA_TYPE
FROM INFORMATION_SCHEMA.COLUMNS
WHERE TABLE_NAME = 'ORDERS'
```

---

## PostgreSQL Patterns

### Date formatting
```sql
-- Format as year-month
TO_CHAR(created_at, 'YYYY-MM')

-- Truncate to month
DATE_TRUNC('month', created_at)

-- Extract year
EXTRACT(YEAR FROM created_at)
```

### Date arithmetic
```sql
-- Subtract interval
created_at - INTERVAL '30 days'
NOW() - INTERVAL '1 month'

-- Age / difference
AGE(NOW(), created_at)
```

### Type casting
```sql
value::float
value::text
value::int
CAST(value AS NUMERIC(10,2))
```

### Window functions (not available in H2)
```sql
-- Running total
SUM(total) OVER (ORDER BY created_at)

-- Rank within group
RANK() OVER (PARTITION BY category ORDER BY revenue DESC)

-- Month-over-month change
total - LAG(total, 1) OVER (ORDER BY month)
```

---

## MySQL / MariaDB Patterns

### Date formatting
```sql
DATE_FORMAT(created_at, '%Y-%m')   -- year-month
DATE_FORMAT(created_at, '%Y')      -- year
YEAR(created_at)                   -- extract year
MONTH(created_at)                  -- extract month
```

### Date arithmetic
```sql
DATE_SUB(NOW(), INTERVAL 30 DAY)
DATEDIFF(NOW(), created_at)        -- days between
```

### Type casting
```sql
CAST(value AS DECIMAL(10,2))
CAST(value AS CHAR)
CONVERT(value, SIGNED)
```

---

## JOIN Patterns with Filters

### Problem: Ambiguous column names
When two joined tables both have a `CREATED_AT` column and you use a field filter `{{order_date}}`, H2 cannot determine which `CREATED_AT` the filter refers to and throws an error.

### Solution: Subquery pattern
Wrap the filtered table in a subquery before joining:

```sql
-- ❌ Will fail if both ORDERS and PRODUCTS have CREATED_AT
SELECT p.CATEGORY, SUM(o.TOTAL)
FROM ORDERS o
JOIN PRODUCTS p ON o.PRODUCT_ID = p.ID
WHERE 1=1 [[AND {{order_date}}]]
GROUP BY p.CATEGORY

-- ✅ Subquery isolates the filter to ORDERS only
SELECT p.CATEGORY, SUM(o.TOTAL)
FROM (SELECT * FROM ORDERS WHERE 1=1 [[AND {{order_date}}]]) o
JOIN PRODUCTS p ON o.PRODUCT_ID = p.ID
GROUP BY p.CATEGORY
```

This pattern works across all SQL dialects and is the safest approach whenever joining two tables that might share column names.

### Multi-table JOIN with filter
```sql
SELECT
  p.CATEGORY,
  u.STATE,
  ROUND(SUM(o.TOTAL), 2) AS revenue
FROM (SELECT * FROM ORDERS WHERE 1=1 [[AND {{order_date}}]]) o
JOIN PRODUCTS p ON o.PRODUCT_ID = p.ID
JOIN PEOPLE u ON o.USER_ID = u.ID
GROUP BY p.CATEGORY, u.STATE
ORDER BY revenue DESC
LIMIT 20
```

---

## Common Pitfalls

| Pitfall | Symptom | Fix |
|---|---|---|
| Using legacy `dataset_query` format on v0.58+ | Card saves with `{}` query, silently broken | Call `get_card` on existing card first, use pMBQL format if `stages` key present |
| `null` field ID in template-tag dimension | Validation error on card create/update | Discover real integer field ID from `get_card` result_metadata, never use `null` |
| Integer division | Rate returns 0 | `CAST(numerator AS FLOAT)` before dividing |
| Reserved word alias | SQL parse error | Rename alias (e.g., `month` → `order_month`) |
| Missing `WHERE 1=1` | Filter clause syntax error | Always prefix optional filter with `WHERE 1=1` |
| Ambiguous column in JOIN | "Column X not found" with filter | Wrap filtered table in subquery |
| Filter on non-ORDERS card | Wrong/empty results | Only wire filter to cards that query that table |
| Wrong date function | "Function not found" | Check dialect — H2 uses `FORMATDATETIME`, PG uses `TO_CHAR` |
| NULL in aggregates | Unexpected totals | Use `COALESCE(column, 0)` or `NULLIF` appropriately |
| Case-sensitive table names | "Table not found" | H2 is case-insensitive; PostgreSQL is lowercase by default |
