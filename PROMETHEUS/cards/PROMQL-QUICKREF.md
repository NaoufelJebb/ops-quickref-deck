# PROMQL QUICK REFERENCE

## The essentials

```promql
# Rate of a counter (per second, over 5m)
rate(http_requests_total[5m])

# Total increase over a window
increase(http_requests_total[1h])

# Aggregate across instances
sum(rate(http_requests_total[5m])) by (service, status)

# P99 latency from a histogram
histogram_quantile(0.99,
  sum(rate(http_request_duration_seconds_bucket[5m])) by (le, service)
)

# Average latency
rate(http_request_duration_seconds_sum[5m])
/ rate(http_request_duration_seconds_count[5m])
```

---

## RED method (Request, Error, Duration)

```promql
# Request rate (RPS)
sum(rate(http_requests_total[5m])) by (service)

# Error rate %
100 * sum(rate(http_requests_total{status=~"5.."}[5m])) by (service)
    / sum(rate(http_requests_total[5m])) by (service)

# P95 / P99 duration
histogram_quantile(0.95,
  sum(rate(http_request_duration_seconds_bucket[5m])) by (le, service)
)
```

---

## USE method (Utilization, Saturation, Errors) — infrastructure

```promql
# CPU utilization %
100 - (avg(rate(node_cpu_seconds_total{mode="idle"}[5m])) by (instance) * 100)

# Memory utilization %
100 * (1 - node_memory_MemAvailable_bytes / node_memory_MemTotal_bytes)

# Disk utilization %
100 * (1 - node_filesystem_avail_bytes{fstype!="tmpfs"}
         / node_filesystem_size_bytes{fstype!="tmpfs"})

# CPU saturation (run queue > CPUs)
node_load1 > count(node_cpu_seconds_total{mode="idle"}) without (cpu, mode)

# Memory saturation (swapping)
rate(node_vmstat_pswpin[5m]) + rate(node_vmstat_pswpout[5m]) > 0

# Disk I/O saturation
rate(node_disk_io_time_seconds_total[5m]) > 0.9
```

---

## Kubernetes

```promql
# Pod CPU usage
sum(rate(container_cpu_usage_seconds_total{container!=""}[5m])) by (pod, namespace)

# Pod memory usage
sum(container_memory_working_set_bytes{container!=""}) by (pod, namespace)

# Pod restarts (last hour)
increase(kube_pod_container_status_restarts_total[1h]) > 0

# Pods not ready
kube_pod_status_ready{condition="true"} == 0

# Deployment replicas mismatch
kube_deployment_status_replicas_available
  != kube_deployment_spec_replicas

# PVC usage %
100 * kubelet_volume_stats_used_bytes
    / kubelet_volume_stats_capacity_bytes > 80

# Node disk pressure
kube_node_status_condition{condition="DiskPressure", status="true"} == 1
```

---

## SLO / Error budget

```promql
# 30-day availability (rolling)
1 - (
  sum(increase(http_requests_total{status=~"5.."}[30d])) by (service)
  / sum(increase(http_requests_total[30d])) by (service)
)

# Error budget remaining % (for 99.9% SLO)
100 * (
  1 - (
    sum(increase(http_requests_total{status=~"5.."}[30d])) by (service)
    / sum(increase(http_requests_total[30d])) by (service)
  )
) / 0.001

# Fast burn rate (14.4× — page now)
sum(rate(http_requests_total{status=~"5.."}[5m])) by (service)
/ sum(rate(http_requests_total[5m])) by (service)
> 14.4 * 0.001

# Slow burn rate (6× — ticket)
sum(rate(http_requests_total{status=~"5.."}[1h])) by (service)
/ sum(rate(http_requests_total[1h])) by (service)
> 6 * 0.001
```

---

## Useful one-liners

```promql
# Instance down
up == 0

# Predict disk full in 24h
predict_linear(node_filesystem_free_bytes{fstype!="tmpfs"}[6h], 86400) < 0

# Top 5 services by request volume
topk(5, sum(rate(http_requests_total[5m])) by (service))

# Metric disappeared (alert on absence)
absent(up{job="myapp"})

# Count active time-series per job
count by (job) ({__name__=~".+"})
```

---

## Label selector quick syntax

```promql
{label="exact"}          exact match
{label!="value"}         not equal
{label=~"regex"}         regex match
{label!~"regex"}         regex not match
{label=~"a|b|c"}         matches a, b, or c
{status=~"5.."}          any 5xx status
{namespace!~"kube-.*"}   not kube- namespaces
```
