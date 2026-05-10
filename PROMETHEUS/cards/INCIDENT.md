# INCIDENT

## Alerts not firing

```bash
# 1. Is the expression returning results?
#    Paste the alert expr into Prometheus UI → Graph tab → execute
#    If no data: check label matchers, check metric exists

# 2. Check alert state (pending vs firing)
curl http://localhost:9090/api/v1/alerts | \
  jq '.data.alerts[] | {name: .labels.alertname, state: .state, value: .value}'
# pending = for duration not met yet — wait or reduce `for:`
# firing = should be in Alertmanager

# 3. Is the rule loading correctly?
curl http://localhost:9090/api/v1/rules | \
  jq '.data.groups[].rules[] | select(.type == "alerting") | {name, lastEvaluation, evaluationTime}'

# 4. Rule file syntax error?
promtool check rules /etc/prometheus/rules/*.yml
# Check Prometheus logs for rule load errors

# 5. Alert reaching Alertmanager?
curl http://localhost:9093/api/v2/alerts | jq '.[] | {name: .labels.alertname}'

# 6. Alertmanager routing correctly?
amtool config routes test \
  --config.file=alertmanager.yml \
  alertname=MyAlert severity=critical
```

---

## Targets are down / missing

```bash
# List unhealthy targets
curl http://localhost:9090/api/v1/targets | \
  jq '.data.activeTargets[] | select(.health != "up") | {job, instance, lastError}'

# Common errors and fixes:
# "connection refused"          → service not running or wrong port
# "context deadline exceeded"   → increase scrape_timeout, or fix slow /metrics endpoint
# "no such host"                → DNS failure — check service name
# "i/o timeout"                 → network connectivity issue
# "401 Unauthorized"            → add basicAuth or bearerToken to scrape config
# "x509 certificate"            → add tls_config: insecure_skip_verify: true (dev)

# In operator mode — is the ServiceMonitor picked up?
kubectl exec -n monitoring prometheus-kube-prom-0 -- \
  curl -s http://localhost:9090/api/v1/targets | \
  jq '.data.activeTargets[] | select(.labels.job == "myapp")'
# Nothing returned = ServiceMonitor label selector mismatch
```

---

## Prometheus is slow / high memory

```bash
# How many active series?
curl http://localhost:9090/api/v1/tsdb/status | \
  jq '.data.headStats | {numSeries, numLabelPairs, chunkCount}'

# Top metrics by series count
curl http://localhost:9090/api/v1/tsdb/status | \
  jq '.data.seriesCountByMetricName[:10] | sort_by(.count) | reverse[]'

# Top label values (cardinality problem)
curl http://localhost:9090/api/v1/tsdb/status | \
  jq '.data.seriesCountByLabelValuePair[:10]'

# Immediate fix — drop high-cardinality metrics at scrape
# Add to scrape config:
metric_relabel_configs:
  - source_labels: [__name__]
    regex: "high_cardinality_metric.*"
    action: drop
  - regex: "user_id|request_id|session_id"
    action: labeldrop
```

---

## Alertmanager not delivering

```bash
# Check Alertmanager is receiving alerts from Prometheus
curl http://localhost:9093/api/v2/alerts | jq length
# 0 = Prometheus not sending — check alertmanager config in prometheus.yml

# Check for delivery errors
docker logs alertmanager 2>&1 | grep -i "error\|fail\|timeout" | tail -20
kubectl logs -n monitoring alertmanager-0 -c alertmanager | grep -i error

# Test routing
amtool config routes test --config.file=alertmanager.yml \
  alertname=TestAlert severity=critical team=platform

# Is there an active silence suppressing the alert?
curl http://localhost:9093/api/v2/silences | \
  jq '.[] | select(.status.state == "active")'

# Validate Alertmanager config
amtool check-config alertmanager.yml
```

---

## Emergency: silence all alerts (maintenance)

```bash
# Silence everything for 2 hours
curl -X POST http://localhost:9093/api/v2/silences \
  -H "Content-Type: application/json" \
  -d '{
    "matchers": [{"name":"alertname","value":".+","isRegex":true}],
    "startsAt": "'$(date -u +%Y-%m-%dT%H:%M:%SZ)'",
    "endsAt":   "'$(date -u -d "+2 hours" +%Y-%m-%dT%H:%M:%SZ)'",
    "comment":  "Emergency maintenance",
    "createdBy":"ops"
  }'

# Remove silence after maintenance
curl -X DELETE http://localhost:9093/api/v2/silences/<silence-id>
```

---

## Prometheus TSDB corruption

```bash
# Prometheus won't start after crash — check logs
journalctl -u prometheus -f
# Look for: "error opening storage" or "corrupted block"

# Delete the corrupt block (Prometheus will rebuild from WAL)
# Block directories look like: /prometheus/01ABCDE...
ls /prometheus/
rm -rf /prometheus/01ABCDE_CORRUPT_BLOCK/

# If WAL is corrupt — last resort, lose recent data
rm -rf /prometheus/wal/
# Prometheus recreates WAL on next start
```
