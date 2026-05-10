# OPS AND TROUBLESHOOTING

## Common exporters

| Exporter | Metrics | Default port |
|---|---|---|
| node_exporter | CPU, memory, disk, network, filesystem | 9100 |
| kube-state-metrics | K8s object state (pods, deployments, PVCs…) | 8080 |
| blackbox_exporter | HTTP/TCP/ICMP probes (uptime checks) | 9115 |
| mysqld_exporter | MySQL/MariaDB metrics | 9104 |
| postgres_exporter | PostgreSQL metrics | 9187 |
| redis_exporter | Redis metrics | 9121 |
| kafka_exporter | Kafka consumer lag, topic metrics | 9308 |
| elasticsearch_exporter | Elasticsearch cluster/index metrics | 9114 |
| mongodb_exporter | MongoDB metrics | 9216 |
| jmx_exporter | JVM and JMX metrics (Java apps) | 9404 |
| statsd_exporter | Translate StatsD → Prometheus | 9102 |
| pushgateway | Accept pushed metrics (batch jobs) | 9091 |

### Blackbox exporter — HTTP uptime check

```yaml
# blackbox.yml
modules:
  http_2xx:
    prober: http
    timeout: 5s
    http:
      valid_http_versions: ["HTTP/1.1", "HTTP/2.0"]
      valid_status_codes: [200]
      follow_redirects: true
      preferred_ip_protocol: ip4
```

```yaml
# Scrape config for blackbox
- job_name: blackbox-http
  metrics_path: /probe
  params:
    module: [http_2xx]
  static_configs:
    - targets:
        - https://api.example.com/health
        - https://app.example.com
  relabel_configs:
    - source_labels: [__address__]
      target_label: __param_target
    - source_labels: [__param_target]
      target_label: instance
    - target_label: __address__
      replacement: blackbox-exporter:9115
```

Alert on probe failure:
```yaml
- alert: EndpointDown
  expr: probe_success == 0
  for: 1m
  labels:
    severity: critical
  annotations:
    summary: "Endpoint {{ $labels.instance }} is down"
```

---

## Cardinality — the main performance problem

High cardinality = too many unique label value combinations = too many time-series = Prometheus runs out of memory.

### Find high-cardinality metrics

```bash
# Via TSDB status API
curl http://localhost:9090/api/v1/tsdb/status | jq '
  .data.seriesCountByMetricName[:10] |
  sort_by(.count) | reverse[]'

# In PromQL
topk(10, count by (__name__)({__name__=~".+"}))
```

### Fix cardinality issues

```promql
# Never use high-cardinality labels:
# user_id, request_id, session_id, trace_id, IP address, full URL

# BAD — unbounded cardinality
http_requests_total{user_id="12345"}

# GOOD — aggregate, don't label
http_requests_total{status="200", method="GET"}
```

```yaml
# Drop high-cardinality labels at scrape time
metric_relabel_configs:
  - source_labels: [user_id]
    action: labeldrop            # remove the label entirely
  - regex: "request_id|trace_id|session_id"
    action: labeldrop

  # Drop metrics you don't need
  - source_labels: [__name__]
    regex: "go_gc_.*|go_memstats_.*"
    action: drop
```

---

## Performance tuning

```bash
# How many time-series?
curl http://localhost:9090/api/v1/status/tsdb | jq '.data.headStats'

# Memory usage (rule of thumb: ~2–3KB per active time-series)
# 100,000 series ≈ 200–300MB RAM

# Key flags
--storage.tsdb.retention.time=30d
--storage.tsdb.retention.size=50GB    # prefer size over time
--query.max-concurrency=20
--query.timeout=2m
--scrape.discovery-reload-interval=5m

# WAL compression (reduces disk I/O)
--storage.tsdb.wal-compression
```

---

## Troubleshooting

### Target is "down" or unhealthy

```bash
# Check target status in UI: http://localhost:9090/targets

# API
curl http://localhost:9090/api/v1/targets | \
  jq '.data.activeTargets[] | select(.health != "up") | {job, instance, health, lastError}'

# Common causes:
# - "connection refused" → service not running or wrong port
# - "context deadline exceeded" → scrape_timeout too short or app is slow
# - "no such host" → DNS resolution failure
# - 401/403 → missing auth config in scrape_config
```

### Alerts not firing

```bash
# Check rule evaluation
curl http://localhost:9090/api/v1/rules | jq '.data.groups[].rules[] | select(.type == "alerting")'

# Check pending vs firing state
curl http://localhost:9090/api/v1/alerts | jq '.data.alerts[]'

# Is the expression returning results?
# Paste the expr into the Prometheus UI and execute manually

# Common causes:
# - `for` duration not met yet (shows as "pending" not "firing")
# - Expression returns no data (check label matchers)
# - Rule file syntax error — check Prometheus logs
```

### Validate config and rules before applying

```bash
# Validate prometheus.yml
promtool check config prometheus.yml

# Validate rule files
promtool check rules /etc/prometheus/rules/*.yml

# Test alerting rules against a snapshot
promtool test rules tests/alerts_test.yml

# Lint metrics format
promtool check metrics < /tmp/metrics.txt
```

### Alertmanager not routing

```bash
# Check Alertmanager config is valid
amtool check-config alertmanager.yml

# Test routing for a specific alert
amtool config routes test \
  --config.file=alertmanager.yml \
  alertname=HighErrorRate severity=critical service=myapp

# Check active alerts in Alertmanager
amtool alert --alertmanager.url=http://localhost:9093

# Check silences
amtool silence --alertmanager.url=http://localhost:9093

# Verify receiver is reachable
# Alertmanager logs show delivery failures:
docker logs alertmanager 2>&1 | grep -i "error\|fail"
```

---

## Ops checklist

- [ ] Retention set — time and/or size limit configured
- [ ] Storage monitored — alert before disk fills
- [ ] `promtool check config` runs in CI before config changes
- [ ] Cardinality monitored — alert if total series > threshold
- [ ] Two Alertmanager instances in HA cluster
- [ ] Alertmanager mesh configured — instances gossip on port 9094
- [ ] At least one alert for `up == 0` per critical job
- [ ] Audit audit log (Alertmanager sends resolved) — send_resolved: true
- [ ] Remote write / Thanos configured for long-term storage
- [ ] Recording rules for expensive queries used in dashboards
- [ ] Blackbox probes for external endpoint monitoring
