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
- Send `"query"` (single string) when creating; Databricks stores it internally as `"queryLines"` (array of strings). Both formats accepted on update.

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

- **REST API (`POST/PATCH /api/2.0/lakeview/dashboards`):** Grid is **12 columns** wide
- **`.lvdash.json` file format:** Grid is **6 columns** wide
- Position: `{ "x": 0, "y": 0, "width": 6, "height": 4 }`
- Height is in grid units

### Consistent Sizing Rules

**Same-type charts in a row MUST have equal dimensions:**

For 6-column grid (.lvdash.json):
```
Row of 2 charts:  width=3, height=4 each
Row of 3 charts:  width=2, height=4 each
Full-width chart: width=6, height=4
```

For 12-column grid (REST API):
```
Row of 2 charts:  width=6, height=5 each
Row of 3 charts:  width=4, height=5 each
Row of 6 charts:  width=2, height=8 each
Full-width chart: width=12, height=6
```

**Counter metrics row:** Counters get equal small widths, filling the row evenly:

```
4 counters (6-col): width=1.5 each
4 counters (12-col): width=3 each
3 counters: width=2 (6-col) or width=4 (12-col)
```

**Height consistency:** All charts in the same row use the same height. Standard heights:
- Counters: `height=2`
- Charts: `height=4` to `height=8` (taller for narrow charts)
- Tables: `height=5` or `height=6`
- Filters row: `height=1`

### Layout Template (6-column grid)

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

**CRITICAL: The `y` encoding MUST use `"fields"` (array), NOT `"fieldName"` (string).** Using `"fieldName"` for `y` will render "No data" or an empty chart.

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
          "disaggregated": true,
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
        "y": {
          "fields": [
            { "fieldName": "total_revenue", "displayName": "Total Revenue (USD)" }
          ],
          "scale": { "type": "quantitative" }
        }
      }
    }
  }
}
```

## Multi-Line Charts

There are TWO approaches for showing multiple lines. Choose the right one:

### Approach 1: Multiple Y-Fields (PREFERRED for related metrics)

Use when plotting two related metrics from the same dataset (e.g., current value vs baseline, actual vs forecast, scored % vs rolling average). Each field becomes a separate line. **NO color encoding needed.**

**Dataset:** Return separate columns for each line.

```sql
SELECT metric_date,
       ROUND(value, 2) AS current_value,
       ROUND(rolling_14d_avg, 2) AS baseline
FROM baselines_table
WHERE metric_date >= CURRENT_DATE - INTERVAL 30 DAYS
ORDER BY metric_date
```

**Widget:** List multiple fields in the `y.fields` array. All fields must also appear in `query.fields`.

```json
{
  "queries": [{
    "name": "main_query",
    "query": {
      "datasetName": "ds_baseline",
      "fields": [
        { "name": "metric_date", "expression": "`metric_date`" },
        { "name": "current_value", "expression": "`current_value`" },
        { "name": "baseline", "expression": "`baseline`" }
      ],
      "disaggregated": true
    }
  }],
  "spec": {
    "version": 3,
    "widgetType": "line",
    "encodings": {
      "x": { "fieldName": "metric_date", "scale": { "type": "temporal" }, "displayName": "Date" },
      "y": {
        "fields": [
          { "fieldName": "current_value", "displayName": "Current" },
          { "fieldName": "baseline", "displayName": "Baseline (14d avg)" }
        ],
        "scale": { "type": "quantitative" }
      }
    },
    "frame": { "showTitle": true, "title": "Current vs Baseline" }
  }
}
```

### Approach 2: Color Encoding (for categorical splits)

Use when data has a category column (e.g., iOS vs Android, region A vs B). Dataset uses GROUP BY or UNION ALL to create rows per category.

```json
{
  "encodings": {
    "x": { "fieldName": "date", "scale": { "type": "temporal" } },
    "y": {
      "fields": [{ "fieldName": "value", "displayName": "Value" }],
      "scale": { "type": "quantitative" }
    },
    "color": {
      "fieldName": "platform",
      "scale": { "type": "categorical" },
      "displayName": "Platform"
    }
  }
}
```

Add the color field to `query.fields`:
```json
{ "name": "platform", "expression": "`platform`" }
```

### WARNING: DO NOT use UNION ALL + color encoding for current vs baseline

Using `UNION ALL` to combine current and baseline into rows with a `series` column (e.g., series='Current' / series='Baseline') does **NOT** work in Lakeview. One series renders correctly but the other appears as a flat line or doesn't render at all. **Always use Approach 1 (multiple y-fields)** for plotting two related metrics from the same dataset.

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
        "fields": [{ "fieldName": "actual_value", "displayName": "Revenue (USD)" }],
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

**WARNING: Table widgets created via the REST API frequently show "Visualization has no fields selected" even with correct format. This is a known Lakeview bug (tested with 9+ format variants including v1/v3, typed/untyped columns, queryLines, empty encodings — ALL fail). Tables work reliably only when created manually through the dashboard "Edit draft" UI.**

If you must include tables via API, use version 1 with full column properties:

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
          "displayName": "Region",
          "type": "string",
          "displayAs": "string",
          "visible": true,
          "order": 0,
          "alignContent": "left",
          "allowHTML": false,
          "allowSearch": false,
          "highlightLinks": false,
          "useMonospaceFont": false,
          "preserveWhitespace": false,
          "booleanValues": ["false", "true"],
          "linkOpenInNewTab": true,
          "imageUrlTemplate": "{{ @ }}",
          "imageTitleTemplate": "{{ @ }}",
          "imageWidth": "",
          "imageHeight": "",
          "linkUrlTemplate": "{{ @ }}",
          "linkTextTemplate": "{{ @ }}",
          "linkTitleTemplate": "{{ @ }}"
        }
      ]
    },
    "allowHTMLByDefault": false,
    "itemsPerPage": 25,
    "paginationSize": "default",
    "condensed": true,
    "withRowNumber": false
  }
}
```

Set `disaggregated: true` in the query for tables.

**Recommended workaround:** Replace API-deployed tables with bar charts, counters, or text widgets showing the SQL query. Add tables manually through the UI after deployment.

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
| Line chart `y` uses `fieldName` | MUST use `y.fields` array, not `y.fieldName` |
| UNION ALL for current vs baseline | Use multiple y-fields instead (see Multi-Line Charts) |
| Table via REST API | Known bug — add tables manually via Edit draft UI |
| Field name mismatch | Ensure widget field names exactly match dataset column aliases |
| Missing field in dataset | Add the column to dataset `SELECT` and `GROUP BY` |
| Complex widget expression | Move logic to dataset SQL, reference simple column in widget |
| Dataset returns no rows | Check SQL query independently, ensure data exists |

## Deployment

### Option 1: REST API (PATCH existing dashboard)

**CRITICAL:** After PATCH, you MUST re-publish. The dashboard won't update for viewers until published.

```python
import json, os, urllib.request

# In Databricks job context, ctx.apiToken() throws NoSuchElementException.
# Use environment variable instead:
token = os.environ.get("DATABRICKS_TOKEN")
host = spark.conf.get("spark.databricks.workspaceUrl", "your-workspace.cloud.databricks.com")

dashboard_config = json.dumps({"datasets": datasets, "pages": pages})
payload = json.dumps({
    "display_name": "My Dashboard",
    "warehouse_id": "your_warehouse_id",
    "serialized_dashboard": dashboard_config
}).encode()

# PATCH (update existing)
req = urllib.request.Request(
    f"https://{host}/api/2.0/lakeview/dashboards/{dashboard_id}",
    data=payload, method="PATCH",
    headers={"Authorization": f"Bearer {token}", "Content-Type": "application/json"}
)
resp = urllib.request.urlopen(req)

# MUST re-publish after PATCH
pub_payload = json.dumps({"warehouse_id": "your_warehouse_id", "embed_credentials": True}).encode()
pub_req = urllib.request.Request(
    f"https://{host}/api/2.0/lakeview/dashboards/{dashboard_id}/published",
    data=pub_payload, method="POST",
    headers={"Authorization": f"Bearer {token}", "Content-Type": "application/json"}
)
urllib.request.urlopen(pub_req)
```

**Note:** Use `urllib.request` instead of `requests` library — `requests` may fail in Databricks job context.

### Option 2: Lakeview SDK

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

### Option 3: Workspace Import

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

### Option 4: Databricks Asset Bundles

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
| Using `y.fieldName` instead of `y.fields` array | Line/bar/area charts MUST use `y: { "fields": [...], "scale": {...} }` |
| UNION ALL + color for current vs baseline | Use multiple y-fields approach instead — UNION ALL renders one series as flat line |
| Table widgets via REST API | Known rendering bug — add tables via Edit draft UI instead |
| Using raw column names as labels | Always use human-readable display names with units |
| Plotting forecast as separate series | Use forecast encodings (`yForecast`, `yForecastLower`, `yForecastUpper`) |
| Inconsistent chart sizes in a row | All charts in the same row must have equal width and height |
| Using default Databricks colors | Define `mark` color array on every chart widget |
| Parameter-based filters | Use field-based filters with query entries per dataset |
| Filter not affecting a chart | Add that chart's dataset to the filter's `queries` array |
| `CASE WHEN` in widget expression | Move complex logic to dataset SQL |
| Missing `disaggregated: true` on table | Set in the query object for table widgets |
| Forgetting filter field in GROUP BY | Every filter field must be in SELECT and GROUP BY |
| Not re-publishing after PATCH | MUST `POST /lakeview/dashboards/{id}/published` after every PATCH |
| Using `ctx.apiToken()` in jobs | Throws `NoSuchElementException` — use `os.environ.get("DATABRICKS_TOKEN")` |
| Using `requests` library in jobs | May fail — use `urllib.request` for HTTP calls in job-context notebooks |
