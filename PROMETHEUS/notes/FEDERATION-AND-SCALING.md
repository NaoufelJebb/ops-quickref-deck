# FEDERATION AND SCALING

## Why federation and scaling matter

A single Prometheus instance has limits:
- Local TSDB — no long-term storage beyond retention period
- Single node — no HA (two Prometheus can scrape the same targets, but can't query each other)
- No global view — can't query across multiple clusters

Solutions:

| Approach | Global query | Long-term storage | HA | Complexity |
|---|---|---|---|---|
| **Federation** | Partial | No | No | Low |
| **Remote Write → Thanos Receive** | Yes | Yes | Yes | Medium |
| **Thanos Sidecar** | Yes | Yes | Partial | Medium |
| **Cortex / Mimir** | Yes | Yes | Yes | High |

---

## Federation

A "global" Prometheus scrapes metrics from "leaf" Prometheus instances. Useful for aggregating across clusters without full Thanos setup.

```yaml
# Global Prometheus — federates from leaf instances
scrape_configs:
  - job_name: federate
    scrape_interval: 60s
    honor_labels: true          # keep original job/instance labels
    metrics_path: /federate
    params:
      match[]:
        - '{job="myapp"}'               # only federate this job
        - '{__name__=~"job:.*"}'        # only federate recording rules
        - 'up'                          # federate the up metric
    static_configs:
      - targets:
          - prometheus-eu:9090
          - prometheus-us:9090
          - prometheus-ap:9090
        labels:
          source: federate
```

### What to federate

Federate **aggregated metrics only** (recording rules), not raw high-cardinality metrics:

```yaml
# On each leaf Prometheus — pre-aggregate before federating
rule_files:
  - /etc/prometheus/rules/aggregations.yml
```

```yaml
# aggregations.yml — recording rules on the leaf
groups:
  - name: aggregations
    rules:
      - record: job:http_requests_total:rate5m
        expr: sum(rate(http_requests_total[5m])) by (job, status)
      - record: job:http_request_duration_seconds:p99_5m
        expr: |
          histogram_quantile(0.99,
            sum(rate(http_request_duration_seconds_bucket[5m])) by (le, job)
          )
```

Then on the global Prometheus:
```yaml
params:
  match[]:
    - '{__name__=~"job:.*"}'    # only federate pre-aggregated recording rules
```

### Federation limitations

- Pull interval is usually 60s+ — data staleness compared to leaf (15s scrape)
- No backfill — if global Prometheus is down, data is lost
- Labels can conflict — use `honor_labels: true` carefully

---

## Remote Write

Prometheus pushes metrics to a remote endpoint in real time. The local Prometheus still scrapes and stores locally; remote write is additive.

```yaml
# prometheus.yml
remote_write:
  - url: https://thanos-receive.example.com/api/v1/receive
    remote_timeout: 30s
    queue_config:
      capacity:           10000
      max_shards:         200
      min_shards:         1
      max_samples_per_send: 1000
      batch_send_deadline: 5s
      max_retries:        10
    # Optional: filter what you send
    write_relabel_configs:
      - source_labels: [__name__]
        regex: "go_.*"
        action: drop            # don't send Go runtime metrics to remote
    # TLS / auth
    tls_config:
      ca_file: /etc/prometheus/ca.crt
    authorization:
      credentials_file: /etc/prometheus/token

  # Send to multiple destinations
  - url: https://mimir.example.com/api/v1/push
    headers:
      X-Scope-OrgID: myteam     # Mimir tenant header
```

Remote write in Operator mode:

```yaml
# values.yaml or Prometheus CR
prometheus:
  prometheusSpec:
    remoteWrite:
      - url: https://thanos-receive.example.com/api/v1/receive
        writeRelabelConfigs:
          - sourceLabels: [__name__]
            regex: "go_.*"
            action: drop
```

---

## Thanos

Thanos extends Prometheus with long-term storage, global query, and HA.

### Architecture options

**Sidecar mode** (recommended for existing Prometheus setups):
```
Prometheus ← Thanos Sidecar (uploads TSDB blocks to object storage)
                    ↑
           Thanos Store Gateway (queries object storage)
                    ↑
           Thanos Query (global query layer — deduplicates HA replicas)
                    ↑
           Grafana / Users
```

**Receive mode** (remote write from Prometheus):
```
Prometheus ──remote write──▶ Thanos Receive (ingests + stores to object storage)
                                     ↑
                             Thanos Store Gateway
                                     ↑
                             Thanos Query
```

### Thanos Sidecar setup

```yaml
# Add to Prometheus pod spec (or use kube-prometheus-stack values)
prometheus:
  prometheusSpec:
    thanos:
      image: quay.io/thanos/thanos:v0.35.0
      objectStorageConfig:
        name: thanos-objstore-secret
        key: objstore.yml
```

```yaml
# thanos-objstore-secret (objstore.yml)
type: S3
config:
  bucket: my-thanos-bucket
  endpoint: s3.eu-west-1.amazonaws.com
  region: eu-west-1
  # Use IAM role — no access key needed on AWS
```

### Thanos Query — global queries

```bash
# Query deduplicates across HA replicas using the replica label
thanos query \
  --http-address=0.0.0.0:10902 \
  --grpc-address=0.0.0.0:10901 \
  --query.replica-label=prometheus_replica \
  --store=thanos-sidecar-1:10901 \
  --store=thanos-sidecar-2:10901 \
  --store=thanos-store-gateway:10901
```

Thanos Query exposes a Prometheus-compatible API — point Grafana at it instead of individual Prometheus instances.

### Helm (bitnami Thanos)

```bash
helm install thanos bitnami/thanos \
  --namespace monitoring \
  -f thanos-values.yaml
```

---

## Cortex / Mimir (multi-tenant, horizontally scalable)

Grafana Mimir is the fully open-source evolution of Cortex — horizontally scalable Prometheus-compatible TSDB with multi-tenancy.

```bash
# Prometheus sends data via remote write with tenant header
remote_write:
  - url: https://mimir.example.com/api/v1/push
    headers:
      X-Scope-OrgID: team-a      # tenant ID
```

```bash
# Query Mimir (Prometheus-compatible API)
curl -H "X-Scope-OrgID: team-a" \
  "https://mimir.example.com/prometheus/api/v1/query?query=up"
```

Grafana datasource for Mimir: type `Prometheus`, URL = `https://mimir.example.com/prometheus`, custom header `X-Scope-OrgID: team-a`.

---

## Recording rules — the federation pre-requisite

Recording rules pre-compute expensive queries and store results as new metrics. Essential for:
- Federation (send aggregates, not raw metrics)
- Dashboards with expensive queries (compute once, read many)
- SLO calculations

```yaml
groups:
  - name: slo.rules
    interval: 30s
    rules:
      # Error ratio (5m window)
      - record: slo:http_errors:ratio5m
        expr: |
          sum(rate(http_requests_total{status=~"5.."}[5m])) by (service)
          / sum(rate(http_requests_total[5m])) by (service)

      # Availability (1h window for SLO tracking)
      - record: slo:http_availability:ratio1h
        expr: |
          1 - (
            sum(increase(http_requests_total{status=~"5.."}[1h])) by (service)
            / sum(increase(http_requests_total[1h])) by (service)
          )
```

---

## HA — two Prometheus scraping the same targets

```yaml
# Both Prometheus instances scrape identically
# Differentiate replicas with an external label
global:
  external_labels:
    cluster: prod
    prometheus_replica: replica-0   # different on each instance

# Thanos Query or Grafana deduplicates using the replica label
# --query.replica-label=prometheus_replica
```
