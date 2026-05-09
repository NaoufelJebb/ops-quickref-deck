# PROMETHEUS QUERIES

Scrape endpoint: `GET http://kong:8001/metrics`

## Paste-ready queries

```promql
# 5xx error rate across all services
sum(rate(kong_http_requests_total{status=~"5.."}[5m]))
/ sum(rate(kong_http_requests_total[5m])) * 100

# P99 request latency per service
histogram_quantile(0.99,
  sum(rate(kong_request_latency_ms_bucket{type="request"}[5m])) by (le, service)
)

# P99 upstream latency per service (excludes Kong overhead)
histogram_quantile(0.99,
  sum(rate(kong_request_latency_ms_bucket{type="upstream"}[5m])) by (le, service)
)

# Kong's own processing overhead
histogram_quantile(0.99,
  sum(rate(kong_request_latency_ms_bucket{type="kong"}[5m])) by (le)
)

# Request rate per service
sum(rate(kong_http_requests_total[1m])) by (service)

# Top consumers by request volume
sum(rate(kong_http_requests_total[5m])) by (consumer) > 0

# Upstream target health (1=healthy, 0=unhealthy)
kong_upstream_target_health

# Datastore reachability (alert if 0)
kong_datastore_reachable

# Memory pressure on DB cache dict
kong_memory_lua_shared_dict_bytes{shared_dict="kong_db_cache"}
/ kong_memory_lua_shared_dict_capacity_bytes{shared_dict="kong_db_cache"} * 100
```

## Key metric names

```
kong_http_requests_total          labels: service, route, status, consumer
kong_request_latency_ms_bucket    labels: service, type, le
kong_upstream_target_health       labels: upstream, target, state
kong_bandwidth_bytes_total        labels: service, direction
kong_datastore_reachable
kong_memory_lua_shared_dict_bytes
kong_nginx_connections_total
```
