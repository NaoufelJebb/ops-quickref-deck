# SETUP

## Docker Compose — quick start

```yaml
version: "3.8"
services:
  prometheus:
    image: prom/prometheus:v2.51.0
    ports:
      - "9090:9090"
    volumes:
      - ./prometheus.yml:/etc/prometheus/prometheus.yml
      - prometheus_data:/prometheus
    command:
      - --config.file=/etc/prometheus/prometheus.yml
      - --storage.tsdb.retention.time=30d
      - --storage.tsdb.path=/prometheus
      - --web.enable-lifecycle          # allows POST /-/reload
      - --web.enable-admin-api          # allows snapshot, tsdb clean

  alertmanager:
    image: prom/alertmanager:v0.27.0
    ports:
      - "9093:9093"
    volumes:
      - ./alertmanager.yml:/etc/alertmanager/alertmanager.yml

volumes:
  prometheus_data:
```

## `prometheus.yml` — core config

```yaml
global:
  scrape_interval:     15s      # how often to scrape targets
  evaluation_interval: 15s      # how often to evaluate rules
  scrape_timeout:      10s      # per-scrape timeout
  external_labels:              # added to all metrics and alerts
    cluster: prod
    region:  eu-west-1

# Load alerting rules from files
rule_files:
  - /etc/prometheus/rules/*.yml

# Alertmanager connection
alerting:
  alertmanagers:
    - static_configs:
        - targets: ["alertmanager:9093"]

# Scrape configurations
scrape_configs:

  # Prometheus itself
  - job_name: prometheus
    static_configs:
      - targets: ["localhost:9090"]

  # Node exporter — host metrics
  - job_name: node
    static_configs:
      - targets:
          - node-1:9100
          - node-2:9100
        labels:
          env: prod

  # App with relabeling
  - job_name: myapp
    metrics_path: /metrics
    scheme: http
    static_configs:
      - targets: ["myapp:8080"]
        labels:
          app: myapp
          team: platform
    relabel_configs:
      - source_labels: [__address__]
        target_label: instance

  # Scrape via file-based service discovery
  - job_name: file_sd
    file_sd_configs:
      - files: [/etc/prometheus/targets/*.json]
        refresh_interval: 30s
```

## Reload config without restart

```bash
curl -X POST http://localhost:9090/-/reload
# or send SIGHUP
kill -HUP $(pgrep prometheus)
```

## Retention and storage

```bash
# Retention by time (default: 15d)
--storage.tsdb.retention.time=30d

# Retention by size
--storage.tsdb.retention.size=50GB

# Storage path
--storage.tsdb.path=/prometheus

# Check disk usage
du -sh /prometheus

# TSDB stats via API
curl http://localhost:9090/api/v1/tsdb/status | jq .
```

## Service discovery

### Kubernetes SD (without operator)

```yaml
scrape_configs:
  - job_name: kubernetes-pods
    kubernetes_sd_configs:
      - role: pod
        namespaces:
          names: [my-namespace]
    relabel_configs:
      # Only scrape pods with annotation prometheus.io/scrape: "true"
      - source_labels: [__meta_kubernetes_pod_annotation_prometheus_io_scrape]
        action: keep
        regex: "true"
      # Use custom port annotation
      - source_labels: [__meta_kubernetes_pod_annotation_prometheus_io_port]
        action: replace
        target_label: __address__
        regex: (.+)
        replacement: $1
      # Use custom path annotation
      - source_labels: [__meta_kubernetes_pod_annotation_prometheus_io_path]
        action: replace
        target_label: __metrics_path__
        regex: (.+)
      # Add namespace as label
      - source_labels: [__meta_kubernetes_namespace]
        target_label: namespace
      # Add pod name as label
      - source_labels: [__meta_kubernetes_pod_name]
        target_label: pod
```

### File-based SD (targets in JSON)

```json
[
  {
    "targets": ["app1:8080", "app2:8080"],
    "labels": {
      "job": "myapp",
      "env": "prod",
      "team": "platform"
    }
  }
]
```

## Relabeling cheatsheet

```yaml
relabel_configs:
  # Keep only targets matching a regex
  - source_labels: [__meta_kubernetes_namespace]
    action: keep
    regex: (prod|staging)

  # Drop targets matching a regex
  - source_labels: [__meta_kubernetes_pod_label_app]
    action: drop
    regex: test-.*

  # Rename a label
  - source_labels: [__meta_kubernetes_pod_label_app]
    target_label: app

  # Compose a label from multiple sources
  - source_labels: [__meta_kubernetes_namespace, __meta_kubernetes_pod_name]
    separator: /
    target_label: pod_id

  # Replace __address__ (change scrape target)
  - source_labels: [__address__, __meta_kubernetes_pod_annotation_prometheus_io_port]
    separator: ":"
    regex: ([^:]+)(?::\d+)?;(\d+)
    replacement: $1:$2
    target_label: __address__
```
