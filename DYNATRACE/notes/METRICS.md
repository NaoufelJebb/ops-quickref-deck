# METRICS

## Metric ingestion methods

| Method | Best for |
|---|---|
| **OneAgent auto-instrumentation** | JVM, Node, .NET, Go, PHP, Nginx — zero config |
| **Metrics API v2 (MINT)** | Pushing custom metrics from any source |
| **Extensions 2.0** | Polling external systems (JMX, SNMP, Prometheus, WMI) |
| **OpenTelemetry** | Standards-based SDK instrumentation |
| **Prometheus scraping** | Existing Prometheus exporters |
| **StatsD / Telegraf** | Legacy metric pipelines |

---

## Metrics API v2 — ingest custom metrics

Endpoint: `POST https://<env>.live.dynatrace.com/api/v2/metrics/ingest`

### MINT format (text protocol)

```
# One line per data point
my.metric.name,dimension1=value1,dimension2=value2 42 1704067200000

# Examples
app.queue.depth,queue=payments,env=prod 1234
app.response.time.ms,service=checkout,region=eu 245.3
app.active.users count,app=web 891 1704067200000
```

### Curl example

```bash
curl -X POST "https://<env>.live.dynatrace.com/api/v2/metrics/ingest" \
  -H "Authorization: Api-Token $DT_API_TOKEN" \
  -H "Content-Type: text/plain; charset=utf-8" \
  -d "app.queue.depth,queue=payments,env=prod 1234"
```

Required token scope: `metrics.ingest`

### Metric types

| Type | MINT syntax | Use for |
|---|---|---|
| Gauge | `key value` | Current value (queue depth, CPU) |
| Count | `key count,delta=N` | Increments (request count) |
| Delta | `key count,delta=N` | Monotonic counters |
| Summary | `key gauge,min=N,max=N,sum=N,count=N` | Pre-aggregated |

---

## Prometheus scraping

Dynatrace can scrape any Prometheus `/metrics` endpoint via:
- ActiveGate Extension (Prometheus Remote Write)
- OneAgent (annotated pods in K8s)

### K8s pod annotation (OneAgent scrapes automatically)

```yaml
annotations:
  metrics.dynatrace.com/scrape: "true"
  metrics.dynatrace.com/port: "8080"
  metrics.dynatrace.com/path: "/metrics"
```

Metrics appear prefixed as `ext:` in Dynatrace.

---

## OpenTelemetry

Dynatrace is a native OTLP endpoint.

```bash
# OTLP endpoint
https://<env>.live.dynatrace.com/api/v2/otlp/v1/metrics
https://<env>.live.dynatrace.com/api/v2/otlp/v1/traces
https://<env>.live.dynatrace.com/api/v2/otlp/v1/logs

# Headers required
Authorization: Api-Token <token>
```

### OpenTelemetry SDK example (Python)

```python
from opentelemetry.exporter.otlp.proto.http.metric_exporter import OTLPMetricExporter
from opentelemetry.sdk.metrics import MeterProvider

exporter = OTLPMetricExporter(
    endpoint="https://<env>.live.dynatrace.com/api/v2/otlp/v1/metrics",
    headers={"Authorization": f"Api-Token {token}"}
)
```

---

## Metric API — query metrics

```bash
# List all available metrics
curl "https://<env>.live.dynatrace.com/api/v2/metrics" \
  -H "Authorization: Api-Token $DT_API_TOKEN"

# Query a metric — last 2 hours, per service
curl "https://<env>.live.dynatrace.com/api/v2/metrics/query" \
  -H "Authorization: Api-Token $DT_API_TOKEN" \
  -G \
  --data-urlencode "metricSelector=builtin:service.response.time:avg:auto" \
  --data-urlencode "resolution=5m" \
  --data-urlencode "from=now-2h" \
  --data-urlencode "entitySelector=type(SERVICE)"
```

### Metric selector syntax

```
builtin:service.response.time                    # raw metric
builtin:service.response.time:avg                # aggregation
builtin:service.response.time:percentile(95)     # percentile
builtin:service.response.time:avg:auto           # auto resolution
builtin:service.errors.total.rate:filter(...)    # with filter
```

---

## Calculated metrics (server-side aggregations)

Create reusable derived metrics in the UI:

`Settings → Server-side service monitoring → Calculated service metrics`

Example: P95 response time for a specific service tag, available everywhere as a named metric.

---

## Metric units and naming

- Prefix custom metrics with your team/app: `myteam.myapp.metric.name`
- Units: use built-in unit types (Bit, Byte, MicroSecond, MilliSecond, Second, Count, Percent)
- Dimensions: max 50 distinct values per dimension key — high-cardinality dimensions cause data loss
