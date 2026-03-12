---
name: databricks-dashboard
description: Use when creating, generating, or modifying Databricks AI/BI dashboard .lvdash.json files, or when the user asks to build a dashboard on Databricks, visualize data in Databricks, or convert queries into a Lakeview dashboard
---

# Databricks AI/BI Dashboard Generation

Generate production-ready `.lvdash.json` dashboard definitions for Databricks Lakeview (AI/BI) dashboards.

## Overview

Databricks dashboards are defined as `.lvdash.json` files containing datasets (SQL queries), pages, and widget layouts. This skill produces consistent, polished dashboards that avoid common pitfalls with filters, parameters, display names, and layout.

## Top-Level Structure

```json
{
  "datasets": [ ... ],
  "pages": [ ... ]
}
```

## ID Convention

All `name` fields (datasets, pages, widgets) use **8-character lowercase hex** IDs. Generate unique ones per entity.

```
"name": "a3f1b9c2"
```

## Datasets

Datasets define SQL queries that power widgets. **Critical rules:**

- Return all data — NO `WHERE` clauses for filtering (filters are client-side)
- Every categorical column used in any filter MUST appear in `SELECT` and `GROUP BY`
- Use human-readable column aliases (see Display Names section)

```json
{
  "name": "a1b2c3d4",
  "displayName": "Monthly Revenue by Region",
  "query": "SELECT date_trunc('month', order_date) AS order_month, region, SUM(revenue) AS total_revenue, COUNT(*) AS order_count FROM catalog.schema.orders GROUP BY date_trunc('month', order_date), region ORDER BY order_month"
}
```

## Display Names — Human-Readable Labels

**NEVER use raw column/variable names in user-facing labels.** Always transform to readable form.

| Raw Column | Display Name |
|------------|-------------|
| `pct_change_yoy` | `Year-over-Year Change (%)` |
| `avg_order_val` | `Average Order Value` |
| `num_active_users` | `Active Users` |
| `ts_created` | `Created Date` |
| `is_churned` | `Churned` |
| `revenue_usd` | `Revenue (USD)` |
| `cust_ltv` | `Customer Lifetime Value` |

**Rules:**
- Add units in parentheses: `(%)`, `(USD)`, `(ms)`, `(GB)`
- Expand abbreviations: `avg` → `Average`, `num` → number context, `pct` → context with `(%)`
- Use title case for display names
- Use SQL aliases to produce clean names: `SUM(revenue) AS total_revenue` then display as `Total Revenue`
- In widget `frame.title`, `encodings.*.displayName`, and table column `title` fields — always use the human-readable form
- When column names are ambiguous or abbreviated, use LLM judgment to infer the best readable name from context

## Pages and Layout

### Grid System

- Grid is **6 columns** wide
- Position: `{ "x": 0, "y": 0, "width": 3, "height": 4 }`

### Consistent Sizing Rules

**Same-type charts in a row MUST have equal dimensions:**

```
Row of 2 charts:  width=3, height=4 each
Row of 3 charts:  width=2, height=4 each
Full-width chart: width=6, height=4
```

**Counter metrics row:** Counters get equal small widths, filling the row evenly:

```
4 counters: width=1.5 each (but use width=1 with padding or width=2 for 3 counters)
3 counters: width=2 each
2 counters: width=3 each
```

**Mixed rows (counters + chart):** Size counters minimally, give remaining width to the chart:

```
2 counters (width=1 each) + 1 chart (width=4, height=4)
```

**Height consistency:** All charts in the same row use the same height. Standard heights:
- Counters: `height=2`
- Charts: `height=4`
- Tables: `height=5` or `height=6`
- Filters row: `height=1`

### Layout Template

```
y=0:  Filter widgets row (height=1)
y=1:  Counter metrics row (height=2)
y=3:  Primary charts row (height=4)
y=7:  Secondary charts row (height=4)
y=11: Table (height=5)
```

## Widget Spec Versions

| Widget Type | `spec.version` | `widgetType` |
|------------|----------------|--------------|
| Counter | 2 | `counter` |
| Table | 1 | `table` |
| Bar chart | 3 | `bar` |
| Line chart | 3 | `line` |
| Area chart | 3 | `area` |
| Pie chart | 3 | `pie` |
| Scatter | 3 | `scatter` |
| Forecast | 3 | `line` (with forecast encodings) |
| Choropleth | 3 | `choropleth` |
| Single-select filter | 2 | `filter-single-select` |
| Multi-select filter | 2 | `filter-multi-select` |
| Date range filter | 2 | `filter-date-range-picker` |

## Chart Widget Template

```json
{
  "position": { "x": 0, "y": 3, "width": 3, "height": 4 },
  "name": "c4d5e6f7",
  "widget": {
    "queries": [
      {
        "name": "main_query",
        "query": {
          "datasetName": "a1b2c3d4",
          "disaggregated": false,
          "fields": [
            { "expression": "`order_month`", "name": "order_month" },
            { "expression": "`total_revenue`", "name": "total_revenue" }
          ]
        }
      }
    ],
    "spec": {
      "version": 3,
      "widgetType": "line",
      "frame": {
        "showTitle": true,
        "title": "Monthly Revenue Trend"
      },
      "encodings": {
        "x": { "displayName": "Month", "fieldName": "order_month", "scale": { "type": "temporal" } },
        "y": { "displayName": "Total Revenue (USD)", "fieldName": "total_revenue", "scale": { "type": "quantitative" } }
      }
    }
  }
}
```

## Forecast Charts

**Use the built-in forecast visualization — do NOT plot forecast as separate series lines.**

For forecast data with actual vs. predicted values and confidence intervals, use the line chart with forecast-specific encodings:

**Dataset must include columns for:** actual value, forecast value, lower bound, upper bound, and a flag/date distinguishing actual from forecast periods.

```json
{
  "spec": {
    "version": 3,
    "widgetType": "line",
    "frame": { "showTitle": true, "title": "Revenue Forecast" },
    "encodings": {
      "x": {
        "displayName": "Date",
        "fieldName": "date",
        "scale": { "type": "temporal" }
      },
      "y": {
        "displayName": "Revenue (USD)",
        "fieldName": "actual_value",
        "scale": { "type": "quantitative" }
      },
      "yForecast": {
        "displayName": "Forecast",
        "fieldName": "forecast_value"
      },
      "yForecastLower": {
        "displayName": "Lower Bound",
        "fieldName": "lower_bound"
      },
      "yForecastUpper": {
        "displayName": "Upper Bound",
        "fieldName": "upper_bound"
      }
    }
  }
}
```

This renders actual values as a solid line, forecast as a dashed line, and confidence bands as a shaded area between upper and lower bounds.

**SQL pattern for forecast datasets:**

```sql
SELECT
  date,
  CASE WHEN date <= CURRENT_DATE() THEN value ELSE NULL END AS actual_value,
  CASE WHEN date > CURRENT_DATE() THEN predicted ELSE NULL END AS forecast_value,
  CASE WHEN date > CURRENT_DATE() THEN predicted - 1.96 * std_err ELSE NULL END AS lower_bound,
  CASE WHEN date > CURRENT_DATE() THEN predicted + 1.96 * std_err ELSE NULL END AS upper_bound
FROM catalog.schema.forecast_table
ORDER BY date
```

## Counter Widget

```json
{
  "spec": {
    "version": 2,
    "widgetType": "counter",
    "frame": { "showTitle": true, "title": "Total Revenue (USD)" },
    "encodings": {
      "value": { "displayName": "Total Revenue (USD)", "fieldName": "total_revenue" }
    }
  }
}
```

## Consistent Brand Colors

**Do NOT use Databricks default colors.** Define a consistent palette using the `mark` property:

```json
{
  "spec": {
    "version": 3,
    "widgetType": "bar",
    "mark": ["#2563EB", "#10B981", "#F59E0B", "#EF4444", "#8B5CF6", "#EC4899", "#06B6D4", "#84CC16"],
    "encodings": { ... }
  }
}
```

**Standard palette (8 colors):**
| Index | Hex | Use |
|-------|-----|-----|
| 1 | `#2563EB` | Primary (blue) |
| 2 | `#10B981` | Positive/success (green) |
| 3 | `#F59E0B` | Warning/attention (amber) |
| 4 | `#EF4444` | Negative/alert (red) |
| 5 | `#8B5CF6` | Tertiary (purple) |
| 6 | `#EC4899` | Accent (pink) |
| 7 | `#06B6D4` | Info (cyan) |
| 8 | `#84CC16` | Secondary positive (lime) |

Apply the same `mark` array across ALL chart widgets in the dashboard for visual consistency.

## Filters — Field-Based (NOT Parameter-Based)

**CRITICAL: Use field-based filters, NOT parameters. Parameter-based filters cause "visualization not shown" errors.**

### Why Parameters Break

When you use `parameters` array with `:param_name` in SQL:
- Creates text inputs instead of dropdowns
- Widgets show "visualization not shown" when parameter is empty or mismatched
- No auto-populated dropdown values

### Correct Filter Pattern

Filters reference datasets by field, not by parameter injection:

```json
{
  "position": { "x": 0, "y": 0, "width": 2, "height": 1 },
  "name": "f1a2b3c4",
  "widget": {
    "queries": [
      {
        "name": "filter_query_1",
        "query": {
          "datasetName": "a1b2c3d4",
          "fields": [
            { "expression": "`region`", "name": "region" }
          ]
        }
      },
      {
        "name": "filter_query_2",
        "query": {
          "datasetName": "e5f6a7b8",
          "fields": [
            { "expression": "`region`", "name": "region" }
          ]
        }
      }
    ],
    "spec": {
      "version": 2,
      "widgetType": "filter-multi-select",
      "frame": { "showTitle": true, "title": "Region" },
      "encodings": {
        "fields": [
          { "displayName": "Region", "fieldName": "region" }
        ]
      }
    }
  }
}
```

### Filter Checklist

- [ ] Every dataset that should respond to a filter MUST have a query entry in that filter's `queries` array
- [ ] The field name in the filter MUST exactly match the column alias in the dataset SQL
- [ ] The filter field MUST be in the dataset's `SELECT` and `GROUP BY`
- [ ] Do NOT use `parameters` array, `:param_name` syntax, or `parameterName` anywhere
- [ ] Use `filter-multi-select` for categorical fields (populates dropdown automatically)
- [ ] Use `filter-date-range-picker` for date fields
- [ ] One filter widget per categorical dimension visible on any chart

### Cross-Filtering

For filters to work across multiple charts using different datasets:
1. Add a query entry for EACH dataset in the filter widget's `queries` array
2. The field name must match across all datasets
3. Identify all categorical columns visible on any chart/table — each needs a filter widget

## Table Widget

```json
{
  "spec": {
    "version": 1,
    "widgetType": "table",
    "frame": { "showTitle": true, "title": "Order Details" },
    "encodings": {
      "columns": [
        {
          "fieldName": "region",
          "title": "Region",
          "type": "string",
          "displayAs": "string",
          "visible": true,
          "order": 0,
          "alignContent": "left",
          "booleanValues": ["false", "true"],
          "imageUrlTemplate": "{{ @ }}",
          "imageTitleTemplate": "{{ @ }}",
          "imageWidth": "",
          "imageHeight": "",
          "linkUrlTemplate": "{{ @ }}",
          "linkTextTemplate": "{{ @ }}",
          "linkTitleTemplate": "{{ @ }}",
          "linkOpenInNewTab": true,
          "numberFormat": ""
        }
      ]
    },
    "invisibleColumns": [],
    "allowHTMLByDefault": false,
    "paginationSize": 25
  }
}
```

Set `disaggregated: true` in the query for tables.

## Widget Field Expressions

**Only simple aggregations work in widget `fields[].expression`:**
- Supported: `` `column_name` ``, `SUM()`, `COUNT()`, `AVG()`, `MIN()`, `MAX()`
- **NOT supported:** `CASE WHEN`, `IF`, string functions, arithmetic
- Complex logic belongs in the dataset SQL, not widget expressions

## Avoiding "Visualization Not Shown"

Common causes and fixes:

| Cause | Fix |
|-------|-----|
| Parameter-based filters | Switch to field-based filters (see above) |
| Empty parameter value | Remove parameters entirely |
| Field name mismatch | Ensure widget field names exactly match dataset column aliases |
| Missing field in dataset | Add the column to dataset `SELECT` and `GROUP BY` |
| Complex widget expression | Move logic to dataset SQL, reference simple column in widget |
| Dataset returns no rows | Check SQL query independently, ensure data exists |

## Deployment

### Option 1: Lakeview API

```python
from databricks.sdk import WorkspaceClient
import json

w = WorkspaceClient()
dashboard_json = json.dumps(dashboard_definition)

# Create draft
draft = w.lakeview.create(
    display_name="My Dashboard",
    serialized_dashboard=dashboard_json,
    warehouse_id="your_warehouse_id"
)

# Publish
w.lakeview.publish(
    dashboard_id=draft.dashboard_id,
    embed_credentials=True,
    warehouse_id="your_warehouse_id"
)
```

### Option 2: Workspace Import

```python
import base64, json

content = base64.b64encode(json.dumps(dashboard_definition).encode()).decode()
w.workspace.import_(
    path="/Workspace/Dashboards/my_dashboard.lvdash.json",
    content=content,
    format="AUTO",
    overwrite=True
)
```

### Option 3: Databricks Asset Bundles

```yaml
resources:
  dashboards:
    my_dashboard:
      display_name: "My Dashboard"
      file_path: ../src/my_dashboard.lvdash.json
      warehouse_id: ${var.warehouse_id}
```

Deploy with `databricks bundle deploy`.

## Common Mistakes

| Mistake | Fix |
|---------|-----|
| Using raw column names as labels | Always use human-readable display names with units |
| Plotting forecast as separate series | Use forecast encodings (`yForecast`, `yForecastLower`, `yForecastUpper`) |
| Inconsistent chart sizes in a row | All charts in the same row must have equal width and height |
| Using default Databricks colors | Define `mark` color array on every chart widget |
| Parameter-based filters | Use field-based filters with query entries per dataset |
| Filter not affecting a chart | Add that chart's dataset to the filter's `queries` array |
| `CASE WHEN` in widget expression | Move complex logic to dataset SQL |
| Missing `disaggregated: true` on table | Set in the query object for table widgets |
| Forgetting filter field in GROUP BY | Every filter field must be in SELECT and GROUP BY |
