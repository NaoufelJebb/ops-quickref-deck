# METRIC INGEST

## MINT format (one line per data point)

```
key,dimension=value,dimension2=value2 number [timestamp_ms]

# Examples
app.queue.depth,queue=orders,env=prod 1234
app.response.ms,service=checkout 245.3
app.active.users,app=web 891 1704067200000
```

## Push via curl

```bash
curl -X POST "$DT_ENV/api/v2/metrics/ingest" \
  -H "Authorization: Api-Token $DT_TOKEN" \
  -H "Content-Type: text/plain; charset=utf-8" \
  -d "app.queue.depth,queue=orders,env=prod 1234"
```

Expect: `202 Accepted`

Required token scope: `metrics.ingest`

## Metric types

| Type | MINT syntax | Use for |
|---|---|---|
| Gauge | `key value` | Current value (depth, CPU %) |
| Count | `key count,delta=N` | Increments per interval |
| Summary | `key gauge,min=N,max=N,sum=N,count=N` | Pre-aggregated |

## Rules

- Key prefix: use your team/app namespace (`myteam.myapp.metric`)
- Dimension values: max **50 distinct values** per key — high cardinality = data loss
- Batch multiple lines in one POST, newline-separated
- Timestamp optional — omit to use server receive time

## Verify metric exists

```bash
curl "$DT_ENV/api/v2/metrics/my.queue.depth" \
  -H "Authorization: Api-Token $DT_TOKEN" | jq '{key: .metricId, unit: .unit, dimensions: .dimensionDefinitions}'
```

## OTLP alternative

```
Endpoint: $DT_ENV/api/v2/otlp/v1/metrics
Header:   Authorization: Api-Token <token>
Protocol: OTLP/HTTP
```
