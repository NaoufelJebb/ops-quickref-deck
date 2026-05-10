# ALERTING

## Alert rules

Stored in `.yml` files, loaded via `rule_files` in `prometheus.yml`.

```yaml
# /etc/prometheus/rules/myapp.yml
groups:
  - name: myapp
    interval: 30s      # evaluate every 30s (default: global evaluation_interval)
    rules:

      # Simple threshold alert
      - alert: HighErrorRate
        expr: |
          sum(rate(http_requests_total{status=~"5.."}[5m])) by (service)
          / sum(rate(http_requests_total[5m])) by (service) > 0.05
        for: 2m          # must be true for 2m before firing
        labels:
          severity: critical
          team: platform
        annotations:
          summary: "High error rate on {{ $labels.service }}"
          description: "Error rate is {{ $value | humanizePercentage }} (threshold: 5%)"
          runbook: "https://wiki.example.com/runbooks/high-error-rate"

      # Instance down
      - alert: InstanceDown
        expr: up == 0
        for: 1m
        labels:
          severity: critical
        annotations:
          summary: "Instance {{ $labels.instance }} is down"

      # P99 latency
      - alert: HighLatencyP99
        expr: |
          histogram_quantile(0.99,
            sum(rate(http_request_duration_seconds_bucket[5m])) by (le, service)
          ) > 2
        for: 5m
        labels:
          severity: warning
        annotations:
          summary: "P99 latency above 2s for {{ $labels.service }}"
          description: "Current P99: {{ $value | humanizeDuration }}"

      # Disk filling up
      - alert: DiskWillFillIn24h
        expr: |
          predict_linear(node_filesystem_free_bytes{fstype!="tmpfs"}[6h], 24 * 3600) < 0
        for: 30m
        labels:
          severity: warning
        annotations:
          summary: "Disk on {{ $labels.instance }} will fill within 24h"

      # Recording rule (pre-compute expensive queries)
      - record: job:http_requests_total:rate5m
        expr: sum(rate(http_requests_total[5m])) by (job)
```

## Alertmanager configuration

```yaml
# alertmanager.yml
global:
  resolve_timeout: 5m
  smtp_from: alerts@example.com
  smtp_smarthost: smtp.example.com:587
  smtp_auth_username: alerts@example.com
  smtp_auth_password: password

templates:
  - /etc/alertmanager/templates/*.tmpl

route:
  # Default receiver
  receiver: default-slack
  group_by: [alertname, cluster, service]
  group_wait:      30s     # wait to batch alerts
  group_interval:  5m      # wait between re-groupings
  repeat_interval: 4h      # resend if still firing

  # Sub-routes (evaluated in order — first match wins)
  routes:
    # Critical alerts → PagerDuty
    - matchers:
        - severity = critical
      receiver: pagerduty
      repeat_interval: 1h

    # Warning alerts → Slack only
    - matchers:
        - severity = warning
      receiver: warning-slack
      repeat_interval: 4h

    # Team-specific routing
    - matchers:
        - team = platform
      receiver: platform-slack
      routes:
        - matchers:
            - severity = critical
          receiver: platform-pagerduty

receivers:
  - name: default-slack
    slack_configs:
      - api_url: https://hooks.slack.com/services/xxx
        channel: "#alerts"
        send_resolved: true
        title: "{{ .GroupLabels.alertname }}"
        text: |
          {{ range .Alerts }}
          *Alert:* {{ .Annotations.summary }}
          *Details:* {{ .Annotations.description }}
          *Labels:* {{ range .Labels.SortedPairs }}{{ .Name }}={{ .Value }} {{ end }}
          {{ end }}

  - name: pagerduty
    pagerduty_configs:
      - routing_key: <pagerduty-integration-key>
        severity: "{{ .CommonLabels.severity }}"
        description: "{{ .CommonAnnotations.summary }}"

  - name: warning-slack
    slack_configs:
      - api_url: https://hooks.slack.com/services/yyy
        channel: "#alerts-warning"
        send_resolved: true

  - name: platform-pagerduty
    pagerduty_configs:
      - routing_key: <platform-pagerduty-key>

inhibit_rules:
  # Suppress warning alerts when critical alert fires for same service
  - source_matchers:
      - severity = critical
    target_matchers:
      - severity = warning
    equal: [service, cluster]

  # Suppress all alerts when cluster is down
  - source_matchers:
      - alertname = ClusterDown
    target_matchers:
      - cluster = prod
```

## Reload Alertmanager config

```bash
curl -X POST http://localhost:9093/-/reload
```

## Manage silences via API

```bash
# Create a silence (suppress alerts during maintenance)
curl -X POST http://localhost:9093/api/v2/silences \
  -H "Content-Type: application/json" \
  -d '{
    "matchers": [
      {"name": "service", "value": "myapp", "isRegex": false},
      {"name": "severity", "value": "warning", "isRegex": false}
    ],
    "startsAt": "2024-01-01T22:00:00Z",
    "endsAt":   "2024-01-02T02:00:00Z",
    "comment":  "Planned maintenance window",
    "createdBy": "ops-team"
  }'

# List active silences
curl http://localhost:9093/api/v2/silences | jq '.[] | select(.status.state == "active")'

# Delete a silence
curl -X DELETE http://localhost:9093/api/v2/silences/<silence-id>
```

## Check firing alerts

```bash
# Via Prometheus API
curl http://localhost:9090/api/v1/alerts | jq '.data.alerts[] | {name: .labels.alertname, state: .state}'

# Via Alertmanager API
curl http://localhost:9093/api/v2/alerts | jq '.[] | {name: .labels.alertname, state: .status.state}'
```
