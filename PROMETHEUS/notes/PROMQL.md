# PROMQL

## Selectors

```promql
# Exact match
http_requests_total{job="myapp"}

# Regex match
http_requests_total{status=~"5.."}

# Negative regex
http_requests_total{status!~"2.."}

# Multiple labels
http_requests_total{job="myapp", status="200", method="GET"}

# Time range (range vector)
http_requests_total{job="myapp"}[5m]
```

## Functions

### Counters

```promql
# Per-second rate over 5 minutes (use for dashboards)
rate(http_requests_total[5m])

# Per-second rate, more responsive to spikes (use for alerts)
irate(http_requests_total[5m])

# Total increase over a time window
increase(http_requests_total[1h])
```

### Aggregation

```promql
# Sum across all label combinations
sum(rate(http_requests_total[5m]))

# Sum by specific labels (keep those, drop rest)
sum(rate(http_requests_total[5m])) by (job, status)

# Sum without specific labels (drop those, keep rest)
sum(rate(http_requests_total[5m])) without (instance, pod)

# Other aggregators
avg(rate(http_requests_total[5m])) by (job)
max(rate(http_requests_total[5m])) by (job)
min(rate(http_requests_total[5m])) by (job)
count(up == 1) by (job)
topk(5, sum(rate(http_requests_total[5m])) by (service))
bottomk(3, sum(rate(http_requests_total[5m])) by (service))
```

### Histograms and quantiles

```promql
# P99 request latency
histogram_quantile(0.99,
  sum(rate(http_request_duration_seconds_bucket[5m])) by (le, job)
)

# P50 and P95 latency for a specific service
histogram_quantile(0.95,
  sum(rate(http_request_duration_seconds_bucket{job="myapp"}[5m])) by (le)
)

# Average request duration (from histogram)
rate(http_request_duration_seconds_sum[5m])
/ rate(http_request_duration_seconds_count[5m])
```

### Gauges and math

```promql
# Memory used percentage
100 * (1 - node_memory_MemAvailable_bytes / node_memory_MemFree_bytes)

# Disk usage percentage
100 * (1 - node_filesystem_avail_bytes / node_filesystem_size_bytes)

# CPU usage (1 - idle)
100 * (1 - avg(rate(node_cpu_seconds_total{mode="idle"}[5m])) by (instance))
```

### Time functions

```promql
# Value 1 hour ago
http_requests_total offset 1h

# Predict value in 4 hours (linear extrapolation)
predict_linear(node_filesystem_free_bytes[1h], 4 * 3600)
```

### Comparison and logic

```promql
# Binary comparison (returns 1 where true, nothing where false)
up == 0           # instances that are down
up == 1           # instances that are up

# Boolean modifier (returns 0 or 1 instead of filtering)
up == bool 0

# Logical operators
vector_a and vector_b     # intersection
vector_a or vector_b      # union
vector_a unless vector_b  # difference

# Join two metrics on a label
rate(http_requests_total[5m])
  * on(instance) group_left(region)
  node_info
```

### Absent and changes

```promql
# Alert if a metric disappears
absent(up{job="myapp"})

# Detect value changes (useful for config change detection)
changes(my_config_version[10m]) > 0
```

---

## Common patterns

### Error rate %

```promql
100 * sum(rate(http_requests_total{status=~"5.."}[5m])) by (service)
    / sum(rate(http_requests_total[5m])) by (service)
```

### Request rate (RPS)

```promql
sum(rate(http_requests_total[5m])) by (service)
```

### P99 latency

```promql
histogram_quantile(0.99,
  sum(rate(http_request_duration_seconds_bucket[5m])) by (le, service)
)
```

### Saturation — CPU

```promql
100 - (avg(rate(node_cpu_seconds_total{mode="idle"}[5m])) by (instance) * 100)
```

### Saturation — Memory

```promql
100 * (1 - node_memory_MemAvailable_bytes / node_memory_MemTotal_bytes)
```

### Saturation — Disk

```promql
100 * (1 - node_filesystem_avail_bytes{fstype!="tmpfs"} / node_filesystem_size_bytes{fstype!="tmpfs"})
```

### Instance down

```promql
up == 0
```

### Predict disk full

```promql
predict_linear(node_filesystem_free_bytes{fstype!="tmpfs"}[6h], 24 * 3600) < 0
```

### SLO error budget burn rate

```promql
# Fast burn: 5xx rate consuming budget at 14.4× for 5 minutes
sum(rate(http_requests_total{status=~"5.."}[5m])) by (service)
/ sum(rate(http_requests_total[5m])) by (service)
> (1 - 0.999) * 14.4
```

---

## Label manipulation in queries

```promql
# Add a static label to a result
label_replace(up, "env", "production", "", "")

# Replace a label value using regex
label_replace(
  http_requests_total,
  "short_method", "$1", "method", "(.).+"
)
```
