# OBSERVABILITY

## Prometheus scrape endpoint

```
GET http://kong:8001/metrics
```

## Key metrics

```
kong_http_requests_total          {service, route, status, consumer}
kong_request_latency_ms_bucket    {service, type=request|upstream|kong, le}
kong_upstream_target_health       {upstream, target, state}
kong_bandwidth_bytes_total        {service, direction=ingress|egress}
kong_datastore_reachable          → alert if 0
kong_memory_lua_shared_dict_bytes {shared_dict}
```

## Useful PromQL

```promql
# 5xx error rate %
sum(rate(kong_http_requests_total{status=~"5.."}[5m]))
/ sum(rate(kong_http_requests_total[5m])) * 100

# P99 request latency per service
histogram_quantile(0.99,
  sum(rate(kong_request_latency_ms_bucket{type="request"}[5m])) by (le, service)
)

# Kong's own overhead (not upstream)
histogram_quantile(0.99,
  sum(rate(kong_request_latency_ms_bucket{type="kong"}[5m])) by (le)
)

# Top consumers by volume
sum(rate(kong_http_requests_total[5m])) by (consumer) > 0
```

## Alert rules

```yaml
groups:
  - name: kong
    rules:
      - alert: KongHighErrorRate
        expr: |
          sum(rate(kong_http_requests_total{status=~"5.."}[5m]))
          / sum(rate(kong_http_requests_total[5m])) > 0.01
        for: 2m
        labels: { severity: critical }

      - alert: KongHighLatencyP99
        expr: |
          histogram_quantile(0.99,
            sum(rate(kong_request_latency_ms_bucket{type="request"}[5m])) by (le, service)
          ) > 2000
        for: 5m
        labels: { severity: warning }

      - alert: KongDatastoreUnreachable
        expr: kong_datastore_reachable == 0
        for: 1m
        labels: { severity: critical }

      - alert: KongUpstreamUnhealthy
        expr: kong_upstream_target_health{state="healthchecks_off"} == 0
        for: 1m
        labels: { severity: critical }

      - alert: KongMemoryPressure
        expr: |
          kong_memory_lua_shared_dict_bytes{shared_dict="kong_db_cache"}
          / kong_memory_lua_shared_dict_capacity_bytes{shared_dict="kong_db_cache"} > 0.9
        for: 5m
        labels: { severity: warning }
```

## Grafana

Official dashboard: **ID 7424** — import from grafana.com.

| Panel | Query hint |
|---|---|
| Request rate by service | `sum by (service) rate(kong_http_requests_total[1m])` |
| Error rate % | 5xx / total × 100 |
| P99 latency | `histogram_quantile(0.99, ...)` |
| Upstream health | `kong_upstream_target_health` |
| Top consumers | `sum by (consumer)` |
| Bandwidth | `kong_bandwidth_bytes_total` |

## Distributed tracing

```yaml
# OpenTelemetry (Kong 3.x+, preferred)
- name: opentelemetry
  config:
    endpoint: http://otel-collector:4318/v1/traces
    resource_attributes:
      service.name: kong-gateway
    propagation:
      default_format: w3c
    sampling_rate: 1.0

# Zipkin (legacy)
- name: zipkin
  config:
    http_endpoint: http://zipkin:9411/api/v2/spans
    sample_ratio: 0.1
    header_type: b3
```

Trace IDs are added to access logs automatically — correlate Kong logs with upstream traces.

## Latency budget

| Layer | Overhead |
|---|---|
| Kong (no plugins) | ~1 ms |
| + JWT validation | ~2–3 ms |
| + Rate limiting (Redis) | ~3–5 ms |
| + Logging (async) | < 1 ms |
| Upstream service | 50–500 ms — optimize here first |
