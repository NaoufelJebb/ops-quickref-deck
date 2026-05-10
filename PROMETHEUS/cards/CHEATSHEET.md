# CHEATSHEET

## Prometheus API

```bash
export PROM=http://localhost:9090

# Health
curl $PROM/-/healthy
curl $PROM/-/ready

# Reload config (requires --web.enable-lifecycle)
curl -X POST $PROM/-/reload

# Instant query
curl "$PROM/api/v1/query?query=up" | jq .

# Range query
curl "$PROM/api/v1/query_range" \
  --data-urlencode "query=rate(http_requests_total[5m])" \
  --data-urlencode "start=$(date -d '1 hour ago' +%s)" \
  --data-urlencode "end=$(date +%s)" \
  --data-urlencode "step=60" | jq .

# List all targets
curl $PROM/api/v1/targets | jq '.data.activeTargets[] | {job, instance, health}'

# List unhealthy targets
curl $PROM/api/v1/targets | jq '.data.activeTargets[] | select(.health != "up") | {job, instance, lastError}'

# List firing alerts
curl $PROM/api/v1/alerts | jq '.data.alerts[] | select(.state == "firing")'

# List all rules
curl $PROM/api/v1/rules | jq '.data.groups[].rules[] | {name: .name, type: .type}'

# TSDB status (cardinality)
curl $PROM/api/v1/tsdb/status | jq '.data.seriesCountByMetricName[:10]'

# Label values
curl "$PROM/api/v1/label/job/values" | jq .
curl "$PROM/api/v1/label/__name__/values" | jq '.data | length'   # total metric count

# Series matching a selector
curl "$PROM/api/v1/series?match[]=http_requests_total" | jq '.data | length'
```

## Alertmanager API

```bash
export AM=http://localhost:9093

# Health
curl $AM/-/healthy

# Reload config
curl -X POST $AM/-/reload

# List active alerts
curl $AM/api/v2/alerts | jq '.[] | {name: .labels.alertname, state: .status.state}'

# List silences
curl $AM/api/v2/silences | jq '.[] | select(.status.state == "active")'

# Create silence
curl -X POST $AM/api/v2/silences \
  -H "Content-Type: application/json" \
  -d '{
    "matchers": [{"name":"alertname","value":"HighErrorRate","isRegex":false}],
    "startsAt": "'$(date -u +%Y-%m-%dT%H:%M:%SZ)'",
    "endsAt":   "'$(date -u -d '+2 hours' +%Y-%m-%dT%H:%M:%SZ)'",
    "comment":  "maintenance",
    "createdBy":"ops"
  }'

# Delete silence
curl -X DELETE $AM/api/v2/silences/<silence-id>
```

## promtool

```bash
# Validate config
promtool check config prometheus.yml

# Validate rules
promtool check rules rules/*.yml

# Test rules
promtool test rules tests/rules_test.yml

# Check metrics format
curl http://myapp:8080/metrics | promtool check metrics

# Query Prometheus from CLI
promtool query instant http://localhost:9090 'up'
promtool query range http://localhost:9090 'rate(http_requests_total[5m])' \
  --start=$(date -d '1h ago' +%s) --end=$(date +%s) --step=60s
```

## amtool

```bash
# Check Alertmanager config
amtool check-config alertmanager.yml

# Test alert routing
amtool config routes test \
  --config.file=alertmanager.yml \
  alertname=HighErrorRate severity=critical

# List alerts
amtool alert --alertmanager.url=http://localhost:9093

# List silences
amtool silence --alertmanager.url=http://localhost:9093

# Create silence (CLI)
amtool silence add alertname=HighErrorRate \
  --comment="maintenance" \
  --duration=2h \
  --alertmanager.url=http://localhost:9093
```

## Kubernetes / Operator

```bash
# List ServiceMonitors
kubectl get servicemonitor -A

# List PrometheusRules
kubectl get prometheusrule -A

# Check targets picked up by operator Prometheus
kubectl exec -n monitoring prometheus-kube-prom-0 -- \
  curl -s http://localhost:9090/api/v1/targets | \
  jq '.data.activeTargets[] | select(.health != "up") | {job, instance, lastError}'

# Operator logs
kubectl logs -n monitoring deployment/kube-prom-operator -f

# Prometheus logs
kubectl logs -n monitoring prometheus-kube-prom-0 -c prometheus -f
```

## node_exporter quick checks

```bash
# CPU usage %
100 - (avg(rate(node_cpu_seconds_total{mode="idle"}[5m])) by (instance) * 100)

# Memory usage %
100 * (1 - node_memory_MemAvailable_bytes / node_memory_MemTotal_bytes)

# Disk usage %
100 * (1 - node_filesystem_avail_bytes{fstype!="tmpfs"} / node_filesystem_size_bytes{fstype!="tmpfs"})

# Network in (bytes/s)
rate(node_network_receive_bytes_total{device!="lo"}[5m])
```
