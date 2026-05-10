# CONCEPTS

## What Prometheus is

Prometheus is a pull-based metrics monitoring system. It scrapes HTTP endpoints that expose metrics in a text format, stores them as time-series data, and provides a query language (PromQL) to analyze them. Alertmanager handles alert routing.

```
Target (app exposes /metrics)
         ↑  scrape (pull, HTTP)
   [ Prometheus ]  ←  static config / service discovery
         │
         ├── TSDB (local time-series storage)
         ├── PromQL (query engine)
         └── Alertmanager (alert routing → PagerDuty, Slack, email…)
```

## Data model

Every metric is a time-series identified by its **name** and a set of **labels**:

```
http_requests_total{method="GET", status="200", service="api"} 1234 @timestamp
│                  │                                          │     │
metric name        labels (key=value pairs)                   value  timestamp
```

Labels are the primary way to filter and aggregate. Every unique label combination is a separate time-series. High cardinality (many unique label values) = many series = performance problems.

## Metric types

| Type | Description | Example |
|---|---|---|
| **Counter** | Monotonically increasing, resets on restart | `http_requests_total` |
| **Gauge** | Can go up or down | `memory_usage_bytes`, `active_connections` |
| **Histogram** | Samples observations into configurable buckets, also tracks count and sum | `http_request_duration_seconds` |
| **Summary** | Client-side quantiles, count and sum | `rpc_duration_seconds` |

Use `rate()` / `increase()` on counters, never raw values.
Use histograms over summaries — histograms can be aggregated across instances.

## Metric exposition format

What your app (or an exporter) exposes at `/metrics`:

```
# HELP http_requests_total Total HTTP requests received
# TYPE http_requests_total counter
http_requests_total{method="GET",status="200"} 1234
http_requests_total{method="POST",status="201"} 56
http_requests_total{method="GET",status="500"} 7

# HELP process_resident_memory_bytes Resident memory size in bytes
# TYPE process_resident_memory_bytes gauge
process_resident_memory_bytes 52428800

# HELP http_request_duration_seconds HTTP request latency
# TYPE http_request_duration_seconds histogram
http_request_duration_seconds_bucket{le="0.05"} 24054
http_request_duration_seconds_bucket{le="0.1"}  33444
http_request_duration_seconds_bucket{le="0.5"}  100392
http_request_duration_seconds_bucket{le="+Inf"} 144320
http_request_duration_seconds_sum   53423
http_request_duration_seconds_count 144320
```

## Key components

| Component | Role |
|---|---|
| **Prometheus server** | Scrapes, stores, queries |
| **Alertmanager** | Routes alerts to receivers (Slack, PagerDuty, email…) |
| **Pushgateway** | Accepts metrics pushed from batch jobs |
| **Exporters** | Translate non-native metrics (node, mysql, blackbox…) |
| **Grafana** | Dashboard visualization over Prometheus |
| **Thanos / Cortex / Mimir** | Long-term storage and global query layer |
