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

**Authoritative source:** There is no official public spec for `serialized_dashboard` JSON.
The best reference is `databrickslabs/lsql` model.py (auto-generated from Databricks internal OpenAPI specs).

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
Row of 4 charts:  width=3, height=5 each
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

## Widget Spec Versions (from OpenAPI spec)

| Widget Type | `spec.version` | `widgetType` |
|------------|----------------|--------------|
| Counter | 2 | `counter` |
| Table (full-featured) | 1 | `table` |
| Table (simplified) | 2 | `table` |
| Bar chart | 3 | `bar` |
| Line chart | 3 | `line` |
| Area chart | 3 | `area` |
| Pie chart | 3 | `pie` |
| Scatter | 3 | `scatter` |
| Pivot table | 3 | `pivot` |
| Forecast | 3 | `line` (with forecast encodings) |
| Single-select filter | 2 | `filter-single-select` |
| Multi-select filter | 2 | `filter-multi-select` |
| Date range filter | 2 | `filter-date-range-picker` |
| Date picker filter | 2 | `filter-date-picker` |
| Text entry filter | 2 | `filter-text-entry` |
| Symbol map | 2 | `symbol-map` |
| Word cloud | 2 | `word-cloud` |

## Line Chart — y.fieldName vs y.fields (CRITICAL)

**Choose the right `y` encoding based on the chart pattern:**

| Scenario | `y` encoding | `disaggregated` |
|----------|-------------|-----------------|
| **Single series** (one line) | `y.fieldName` (string) | `true` |
| **One measure split by color** (e.g. by platform) | `y.fieldName` (string) | **`false`** |
| **Multiple distinct Y series** (e.g. current vs baseline) | `y.fields` (array) | `true` |

### Pattern A: Single-series line chart

One line, optionally with a custom Y-axis label.

```json
{
  "queries": [{
    "name": "main_query",
    "query": {
      "datasetName": "ds_data",
      "fields": [
        {"name": "metric_date", "expression": "`metric_date`"},
        {"name": "hidden_pct", "expression": "`hidden_pct`"}
      ],
      "disaggregated": true
    }
  }],
  "spec": {
    "version": 3,
    "widgetType": "line",
    "encodings": {
      "x": {
        "fieldName": "metric_date",
        "scale": {"type": "temporal"},
        "axis": {}
      },
      "y": {
        "fieldName": "hidden_pct",
        "displayName": "Hidden Pct",
        "axis": {"title": "%Hidden"},
        "scale": {"type": "quantitative"}
      }
    },
    "frame": {"showTitle": true, "title": "Hidden Trip % (30d)"}
  }
}
```

### Pattern B: One measure split by color (e.g. Trips by Platform)

**CRITICAL:** Use `disaggregated: false` and define an **aggregated query field** whose
`name` matches the aggregation expression pattern (e.g. `"sum(trips)"`).

```json
{
  "queries": [{
    "name": "main_query",
    "query": {
      "datasetName": "ds_trips_by_platform",
      "fields": [
        {"name": "metric_date", "expression": "`metric_date`"},
        {"name": "platform", "expression": "`platform`"},
        {"name": "sum(trips)", "expression": "SUM(`trips`)"}
      ],
      "disaggregated": false
    }
  }],
  "spec": {
    "version": 3,
    "widgetType": "line",
    "encodings": {
      "x": {
        "fieldName": "metric_date",
        "scale": {"type": "temporal"},
        "axis": {}
      },
      "y": {
        "fieldName": "sum(trips)",
        "axis": {"title": "#Trips"},
        "scale": {"type": "quantitative"}
      },
      "color": {
        "fieldName": "platform",
        "scale": {"type": "categorical"},
        "displayName": "Platform"
      }
    },
    "frame": {"showTitle": true, "title": "Trips by Platform"}
  }
}
```

**Why this matters:** Using `y.fields` array + `disaggregated: true` with raw column names
causes Y-axis to show no scale and series render as flat lines at zero. The widget doesn't
apply aggregation unless `disaggregated: false` with a properly named aggregation field.

### Pattern C: Multiple distinct Y series (current vs baseline)

Use `y.fields` array — each field becomes a separate line. NO color encoding needed.
Use `mark` (color array) to control series colors. Use `strokeDash` inside a field for dashed lines.

```json
{
  "queries": [{
    "name": "main_query",
    "query": {
      "datasetName": "ds_data",
      "fields": [
        {"name": "metric_date", "expression": "`metric_date`"},
        {"name": "current_value", "expression": "`current_value`"},
        {"name": "baseline", "expression": "`baseline`"}
      ],
      "disaggregated": true
    }
  }],
  "spec": {
    "version": 3,
    "widgetType": "line",
    "legend": {"orient": "bottom", "direction": "horizontal"},
    "mark": ["#2563EB", "#6b7280"],
    "encodings": {
      "x": {
        "fieldName": "metric_date",
        "scale": {"type": "temporal", "format": "dd, MMM"},
        "axis": {"showTitle": false, "title": ""}
      },
      "y": {
        "fields": [
          {"fieldName": "current_value", "displayName": "Current"},
          {"fieldName": "baseline", "displayName": "Baseline (14d avg)", "strokeDash": [4, 2]}
        ],
        "axis": {"showTitle": false, "title": ""},
        "scale": {"type": "quantitative"}
      }
    },
    "frame": {"showTitle": true, "title": "Current vs Baseline"}
  }
}
```

**Key details (validated against live dashboards):**
- `strokeDash` goes INSIDE `y.fields[i]` (per-field), NOT as a top-level encoding
- `mark` is a simple color array `["#hex1", "#hex2"]` — one per y-field, in order
- `legend` is a top-level spec property: `{"orient": "bottom", "direction": "horizontal"}`

### WARNING: DO NOT use UNION ALL + color encoding for current vs baseline

Using `UNION ALL` to combine current and baseline into rows with a `series` column does **NOT** work in Lakeview. One series renders correctly but the other appears as a flat line. **Always use Pattern C (multiple y-fields)** for related metrics from the same dataset.

## Axis Titles — Hiding and Labelling

All hiding styles work. Choose based on context:

| Goal | How | Notes |
|------|-----|-------|
| **Hide** axis title | `"axis": {"showTitle": false, "title": ""}` | Most common (40 uses in production dashboard) |
| **Hide** (UI-style) | `"axis": {}` | Used by UI editor (6 uses) |
| **Hide** (alternative) | `"axis": {"hideTitle": true}` | Also works (8 uses) |
| **Show** custom label | `"axis": {"title": "#Trips"}` | For Pattern A/B charts |

**Key insight:** `axis.title` controls the **axis label**. `displayName` controls **legend/tooltip**.
Setting `displayName` does NOT set the axis label — you must use `axis.title`.

### X-axis date format

- Custom format: `"scale": {"type": "temporal", "format": "dd, MMM"}`
- Default format: omit `format`, just use `"scale": {"type": "temporal"}` and `"axis": {}`

## Bar Chart Widget

Same encoding structure as line charts. Supports `mark.layout` for stacking.

```json
{
  "spec": {
    "version": 3,
    "widgetType": "bar",
    "encodings": {
      "x": {"fieldName": "category", "scale": {"type": "categorical"}, "axis": {}},
      "y": {"fieldName": "sum(value)", "scale": {"type": "quantitative"}, "axis": {"title": "Count"}},
      "color": {"fieldName": "segment", "scale": {"type": "categorical"}, "displayName": "Segment"}
    },
    "mark": {"layout": "stack"},
    "frame": {"showTitle": true, "title": "My Bar Chart"}
  }
}
```

`mark.layout` options: `"stack"` | `"group"` | `"layer"`

## Pie Chart Widget

Uses `angle` and `color` encodings (NOT x/y).

```json
{
  "spec": {
    "version": 3,
    "widgetType": "pie",
    "encodings": {
      "angle": {"fieldName": "value_col", "displayName": "Value", "scale": {"type": "quantitative"}},
      "color": {"fieldName": "category_col", "displayName": "Category", "scale": {"type": "categorical"}},
      "label": {"show": true}
    },
    "frame": {"showTitle": true, "title": "My Pie Chart"}
  }
}
```

## Forecast Charts

**Use the built-in forecast visualization — do NOT plot forecast as separate series lines.**

```json
{
  "spec": {
    "version": 3,
    "widgetType": "line",
    "frame": {"showTitle": true, "title": "Revenue Forecast"},
    "encodings": {
      "x": {"displayName": "Date", "fieldName": "date", "scale": {"type": "temporal"}},
      "y": {
        "fields": [{"fieldName": "actual_value", "displayName": "Revenue (USD)"}],
        "scale": {"type": "quantitative"}
      },
      "yForecast": {"displayName": "Forecast", "fieldName": "forecast_value"},
      "yForecastLower": {"displayName": "Lower Bound", "fieldName": "lower_bound"},
      "yForecastUpper": {"displayName": "Upper Bound", "fieldName": "upper_bound"}
    }
  }
}
```

## Counter Widget

```json
{
  "spec": {
    "version": 2,
    "widgetType": "counter",
    "frame": {"showTitle": true, "title": "Total Revenue (USD)"},
    "encodings": {
      "value": {"displayName": "Total Revenue (USD)", "fieldName": "total_revenue"}
    }
  }
}
```

## Scale Types and Color Mappings

| Type | Use for | Extra properties |
|------|---------|-----------------|
| `categorical` | Strings, categories | `mappings` (color map), `sort` (`natural-order`, `y`, `x`, etc.) |
| `quantitative` | Numbers | `domain` (`{min, max}`), `reverse` (bool) |
| `temporal` | Dates/timestamps | `format` (e.g. `"dd, MMM"`) — optional |

### Categorical color mapping

Two color formats — hex strings or semantic theme colors:

```json
"scale": {
  "type": "categorical",
  "mappings": [
    {"value": "iOS", "color": "#0693E3"},
    {"value": "Android", "color": "#002135"}
  ]
}
```

```json
"scale": {
  "type": "categorical",
  "mappings": [
    {"value": "GREEN", "color": {"themeColorType": "semanticColors", "semanticColorKey": "positive"}},
    {"value": "RED", "color": {"themeColorType": "semanticColors", "semanticColorKey": "negative"}},
    {"value": "YELLOW", "color": {"themeColorType": "semanticColors", "semanticColorKey": "warning"}}
  ]
}
```

## Mark Spec (colors, layout)

Two formats exist:

**Simple array (for line charts — confirmed working in production):**
```json
"mark": ["#2563EB", "#6b7280"]
```

**Object form (for bar stacking):**
```json
"mark": {
  "colors": ["#002135", "#0693E3", "#5AABE8", "#CF2E2E"],
  "layout": "stack"
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

## Filters — Field-Based (NOT Parameter-Based)

**CRITICAL: Use field-based filters, NOT parameters. Parameter-based filters cause "visualization not shown" errors.**

### Correct Filter Pattern

Filters reference datasets by field, not by parameter injection:

```json
{
  "widget": {
    "queries": [
      {
        "name": "filter_query_1",
        "query": {
          "datasetName": "a1b2c3d4",
          "fields": [{"expression": "`region`", "name": "region"}]
        }
      },
      {
        "name": "filter_query_2",
        "query": {
          "datasetName": "e5f6a7b8",
          "fields": [{"expression": "`region`", "name": "region"}]
        }
      }
    ],
    "spec": {
      "version": 2,
      "widgetType": "filter-multi-select",
      "frame": {"showTitle": true, "title": "Region"},
      "encodings": {
        "fields": [{"displayName": "Region", "fieldName": "region"}]
      }
    }
  }
}
```

### Filter Checklist

- Every dataset that should respond to a filter MUST have a query entry in that filter's `queries` array
- The field name in the filter MUST exactly match the column alias in the dataset SQL
- The filter field MUST be in the dataset's `SELECT` and `GROUP BY`
- Do NOT use `parameters` array, `:param_name` syntax, or `parameterName` anywhere
- Use `filter-multi-select` for categorical fields (populates dropdown automatically)
- Use `filter-date-range-picker` for date fields

## Table Widget

**Use version 1 with full column properties (version 3 does NOT render):**

```json
{
  "spec": {
    "version": 1,
    "widgetType": "table",
    "frame": {"showTitle": true, "title": "Order Details"},
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

## Widget Field Expressions

**Only simple aggregations work in widget `fields[].expression`:**
- Supported: `` `column_name` ``, `SUM()`, `COUNT()`, `AVG()`, `MIN()`, `MAX()`
- **NOT supported:** `CASE WHEN`, `IF`, string functions, arithmetic
- Complex logic belongs in the dataset SQL, not widget expressions

## Query Field Ordering

```json
"query": {
  "datasetName": "ds_data",
  "fields": [ ... ],
  "disaggregated": false,
  "orders": [
    {"direction": "ASC", "expression": "`column_name`"}
  ]
}
```

## Frame (title/description)

```json
"frame": {
  "showTitle": true,
  "title": "Widget Title",
  "showDescription": true,
  "description": "Optional description text"
}
```

## Deployment via MCP

This project has Lakeview MCP tools. Use them instead of raw REST calls:

```
lakeview_get_dashboard(dashboard_id)     — fetch current config
lakeview_push_dashboard(dashboard_id, serialized_dashboard, warehouse_id?, display_name?)
                                          — PATCH + auto-publish
```

The push tool accepts inline JSON (starting with `{`) or a file path to a `.lvdash.json` file.
It normalizes `queryLines` → `query` automatically.

## Common Mistakes

| Mistake | Fix |
|---------|-----|
| Using `y.fields` for single/color-split charts | Use `y.fieldName` (string) for single series and color-split; `y.fields` (array) only for multiple distinct Y columns |
| Color-split with `disaggregated: true` | MUST use `disaggregated: false` with aggregated field name like `"sum(trips)"` |
| Setting axis label via `displayName` | Use `axis.title` for axis labels; `displayName` is legend/tooltip only |
| UNION ALL + color for current vs baseline | Use multiple y-fields approach instead — UNION ALL renders one series as flat line |
| Table version 3 | Does NOT render — always use version 1 with 17+ column properties |
| Using raw column names as labels | Always use human-readable display names with units |
| Inconsistent chart sizes in a row | All charts in the same row must have equal width and height |
| Using default Databricks colors | Define `mark` color array on every chart widget |
| Parameter-based filters | Use field-based filters with query entries per dataset |
| `CASE WHEN` in widget expression | Move complex logic to dataset SQL |
| Not re-publishing after PATCH | MUST `POST /lakeview/dashboards/{id}/published` after every PATCH |
| Using `ctx.apiToken()` in jobs | Throws `NoSuchElementException` — use `os.environ.get("DATABRICKS_TOKEN")` |
| Using `requests` library in jobs | May fail — use `urllib.request` for HTTP calls in job-context notebooks |
