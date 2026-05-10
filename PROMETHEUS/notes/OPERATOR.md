# PROMETHEUS OPERATOR

## What the operator does

The Prometheus Operator manages Prometheus, Alertmanager, and related components on Kubernetes via CRDs. Instead of editing `prometheus.yml` directly, you create Kubernetes resources — the operator reconciles Prometheus config automatically.

```
You create:                    Operator generates:
ServiceMonitor    ──────────▶  scrape_configs in prometheus.yml
PodMonitor        ──────────▶  scrape_configs in prometheus.yml
PrometheusRule    ──────────▶  rule_files
AlertmanagerConfig──────────▶  alertmanager.yml routes/receivers
```

## Install — kube-prometheus-stack (recommended)

Installs: Prometheus Operator + Prometheus + Alertmanager + Grafana + node-exporter + kube-state-metrics + default dashboards and rules.

```bash
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm repo update

helm install kube-prom prometheus-community/kube-prometheus-stack \
  --namespace monitoring \
  --create-namespace \
  -f values.yaml
```

```yaml
# values.yaml — key overrides
prometheus:
  prometheusSpec:
    retention: 30d
    retentionSize: "50GB"
    storageSpec:
      volumeClaimTemplate:
        spec:
          storageClassName: gp3-csi
          accessModes: [ReadWriteOnce]
          resources:
            requests:
              storage: 100Gi
    # Watch ALL namespaces for ServiceMonitors (not just monitoring)
    serviceMonitorNamespaceSelector: {}
    serviceMonitorSelector: {}
    podMonitorNamespaceSelector: {}
    podMonitorSelector: {}
    ruleNamespaceSelector: {}
    ruleSelector: {}
    # External labels added to all metrics
    externalLabels:
      cluster: prod
      region: eu-west-1
    # Remote write to long-term storage
    remoteWrite:
      - url: https://thanos-receive.example.com/api/v1/receive
        # or: url: https://mimir.example.com/api/v1/push

alertmanager:
  alertmanagerSpec:
    storage:
      volumeClaimTemplate:
        spec:
          storageClassName: gp3-csi
          resources:
            requests:
              storage: 10Gi

grafana:
  adminPassword: changeme
  ingress:
    enabled: true
    hosts: [grafana.example.com]

# Disable components you don't need
kubeEtcd:
  enabled: false    # etcd metrics often not accessible
```

---

## CRDs

### ServiceMonitor — scrape a Service

```yaml
apiVersion: monitoring.coreos.com/v1
kind: ServiceMonitor
metadata:
  name: myapp-monitor
  namespace: my-namespace          # can be in any namespace if operator watches all
  labels:
    release: kube-prom             # must match serviceMonitorSelector in Prometheus CR
spec:
  selector:
    matchLabels:
      app: myapp                   # matches Service labels
  namespaceSelector:
    matchNames: [my-namespace]
  endpoints:
    - port: metrics                # port name on the Service (not number)
      path: /metrics
      interval: 30s
      scrapeTimeout: 10s
      scheme: http
      # TLS config (if endpoint uses HTTPS)
      tlsConfig:
        insecureSkipVerify: true
      # Basic auth
      basicAuth:
        username:
          name: scrape-secret
          key: username
        password:
          name: scrape-secret
          key: password
      # Bearer token
      bearerTokenSecret:
        name: scrape-token
        key: token
      # Relabeling
      relabelings:
        - sourceLabels: [__meta_kubernetes_pod_node_name]
          targetLabel: node
      metricRelabelings:
        - sourceLabels: [__name__]
          regex: "go_.*"
          action: drop             # drop Go runtime metrics
```

### PodMonitor — scrape pods directly (no Service)

```yaml
apiVersion: monitoring.coreos.com/v1
kind: PodMonitor
metadata:
  name: myapp-pods
  namespace: my-namespace
spec:
  selector:
    matchLabels:
      app: myapp
  podMetricsEndpoints:
    - port: metrics
      path: /metrics
      interval: 30s
```

### PrometheusRule — alert and recording rules

```yaml
apiVersion: monitoring.coreos.com/v1
kind: PrometheusRule
metadata:
  name: myapp-rules
  namespace: my-namespace
  labels:
    release: kube-prom             # must match ruleSelector in Prometheus CR
spec:
  groups:
    - name: myapp.alerts
      interval: 30s
      rules:
        - alert: MyAppHighErrorRate
          expr: |
            sum(rate(http_requests_total{namespace="my-namespace",status=~"5.."}[5m]))
            / sum(rate(http_requests_total{namespace="my-namespace"}[5m])) > 0.05
          for: 2m
          labels:
            severity: critical
            namespace: my-namespace
          annotations:
            summary: "High error rate in my-namespace"
            description: "{{ $value | humanizePercentage }} of requests are errors"

        - record: namespace:http_requests_total:rate5m
          expr: sum(rate(http_requests_total{namespace="my-namespace"}[5m]))
```

### AlertmanagerConfig — namespace-scoped alert routing

```yaml
apiVersion: monitoring.coreos.com/v1alpha1
kind: AlertmanagerConfig
metadata:
  name: myapp-alerting
  namespace: my-namespace
spec:
  route:
    receiver: myapp-slack
    matchers:
      - name: namespace
        value: my-namespace
    groupBy: [alertname]
    groupWait: 30s
    repeatInterval: 4h
  receivers:
    - name: myapp-slack
      slackConfigs:
        - apiURL:
            name: slack-webhook
            key: url
          channel: "#alerts-myapp"
          sendResolved: true
```

### Prometheus CR — customize the Prometheus instance

```yaml
apiVersion: monitoring.coreos.com/v1
kind: Prometheus
metadata:
  name: prometheus
  namespace: monitoring
spec:
  replicas: 2
  retention: 30d
  retentionSize: 50GB
  serviceAccountName: prometheus
  serviceMonitorSelector:
    matchLabels:
      release: kube-prom
  ruleSelector:
    matchLabels:
      release: kube-prom
  alerting:
    alertmanagers:
      - name: alertmanager-operated
        namespace: monitoring
        port: web
  storage:
    volumeClaimTemplate:
      spec:
        storageClassName: gp3-csi
        resources:
          requests:
            storage: 100Gi
  resources:
    requests:
      memory: 2Gi
      cpu: 500m
    limits:
      memory: 4Gi
```

---

## Useful kubectl commands

```bash
# List all CRDs installed by the operator
kubectl get crd | grep monitoring.coreos.com

# List ServiceMonitors in all namespaces
kubectl get servicemonitor -A

# Check if Prometheus picked up a ServiceMonitor
kubectl exec -n monitoring prometheus-kube-prom-0 \
  -- curl -s http://localhost:9090/api/v1/targets | \
  jq '.data.activeTargets[] | select(.labels.job == "myapp") | {job, health, lastError}'

# Check PrometheusRules
kubectl get prometheusrule -A

# Check AlertmanagerConfig
kubectl get alertmanagerconfig -A

# Reload Prometheus (operator handles this — but if manual)
kubectl exec -n monitoring prometheus-kube-prom-0 \
  -- curl -X POST http://localhost:9090/-/reload

# Check operator logs
kubectl logs -n monitoring deployment/kube-prom-operator -f

# Check Prometheus pod logs
kubectl logs -n monitoring prometheus-kube-prom-0 -c prometheus -f
```

---

## Common issues with the operator

| Symptom | Check |
|---|---|
| ServiceMonitor not picked up | Does `release` label match `serviceMonitorSelector`? |
| Target shows "connection refused" | Wrong port name or port number — use port **name**, not number |
| Target shows "no targets" | `selector.matchLabels` doesn't match Service labels |
| PrometheusRule not loading | `release` label missing or wrong on the rule |
| AlertmanagerConfig not routing | Namespace-scoped — check `matchers` include namespace |
| Metrics namespace missing | Add `relabeling` to preserve namespace label |
