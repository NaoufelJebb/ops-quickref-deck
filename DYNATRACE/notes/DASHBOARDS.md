# DASHBOARDS

## Dashboard types

| Type | Use for |
|---|---|
| **Classic dashboard** | Tile-based, drag-and-drop, shared via link |
| **Notebook** | Exploratory analysis, DQL-based, shareable |
| **App (Dynatrace Apps)** | Pre-built purpose-specific views (K8s app, Log app…) |

---

## Classic dashboard — key tiles

| Tile | Use for |
|---|---|
| **Data explorer** | Any metric — chart, single value, table |
| **SLO** | SLO compliance and error budget |
| **Problems** | Open problem count or problem list |
| **Log viewer** | Embedded DQL log query |
| **Synthetic monitor** | Synthetic test status |
| **Service health** | Single service status |
| **Markdown** | Labels, headers, links |
| **Custom chart** | Legacy metric charting |
| **Top list** | Ranked entities by metric |

---

## Data explorer tile — useful configurations

```
Metric:     builtin:service.response.time
Split by:   Service
Aggregation: Percentile 95
Chart type: Line

Metric:     builtin:service.errors.total.rate
Split by:   Service
Chart type: Bar — shows which services have highest error rate

Metric:     builtin:host.cpu.usage
Filter by:  Tag: env=prod
Aggregation: Max
Chart type: Heatmap
```

---

## Dashboard variables (filters)

Add a variable to make dashboards environment/service-aware:

```
Variable name:  environment
Type:           Entity selector
Default value:  type(SERVICE),tag("env:prod")
Allow edits:    Yes
```

Reference in tiles: `$environment`

---

## Notebook — DQL analysis

Notebooks are better than dashboards for investigation and ad-hoc analysis.

```
Menu → Notebooks → New notebook

Each section is a DQL cell:
  fetch logs | filter loglevel == "ERROR" | summarize count(), by: {dt.entity.service}

Output types:
  Table (default)
  Line chart
  Bar chart
  Single value
  Pie chart
```

---

## Sharing

| Method | How |
|---|---|
| Share link | Dashboard menu → Share → Copy link |
| Embed | Share → Embed → Copy iframe code |
| Export JSON | Dashboard menu → Export as JSON |
| Management zone scoping | Dashboard is auto-filtered to viewer's management zone |

---

## Import a dashboard from JSON

```bash
# Via API
curl -X POST "$DT_ENV/api/config/v1/dashboards" \
  -H "Authorization: Api-Token $DT_TOKEN" \
  -H "Content-Type: application/json" \
  -d @my-dashboard.json
```

---

## Useful dashboard patterns

### Service health overview

```
Row 1: SLO tiles for each critical service
Row 2: Data explorer — P95 latency for top 5 services (line chart)
Row 3: Data explorer — error rate per service (bar chart)
Row 4: Problems tile — open problems count
Row 5: Log viewer — ERROR logs last 30 min
```

### Infrastructure overview

```
Row 1: Host count, Pod count (single value tiles)
Row 2: CPU usage top 10 hosts (top list)
Row 3: Memory usage over time (line chart, split by host)
Row 4: Disk usage per host (heatmap)
Row 5: K8s workload restart count
```

### Deployment impact view

```
Row 1: Event tile — deployment events (last 24h)
Row 2: Response time before/after (annotated time series)
Row 3: Error rate before/after
Row 4: Problems triggered in the deployment window
```
