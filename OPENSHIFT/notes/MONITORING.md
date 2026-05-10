# MONITORING

## Built-in stack

OpenShift ships a fully managed Prometheus + Alertmanager + Grafana stack in `openshift-monitoring`.

```
openshift-monitoring namespace:
├── prometheus-k8s            ← cluster metrics (nodes, pods, control plane)
├── alertmanager-main         ← alert routing
├── grafana                   ← dashboards (read-only, pre-built)
├── thanos-querier            ← unified query across cluster + user workload metrics
└── kube-state-metrics
```

## Enable user workload monitoring

Allows monitoring of your own application metrics in a separate Prometheus instance.

```bash
oc apply -f - <<EOF
apiVersion: v1
kind: ConfigMap
metadata:
  name: cluster-monitoring-config
  namespace: openshift-monitoring
data:
  config.yaml: |
    enableUserWorkload: true
EOF
```

This creates a second Prometheus in `openshift-user-workload-monitoring`.

---

## Scrape your application metrics

Annotate your Service with a ServiceMonitor:

```yaml
apiVersion: monitoring.coreos.com/v1
kind: ServiceMonitor
metadata:
  name: my-app-monitor
  namespace: my-app
spec:
  selector:
    matchLabels:
      app: my-app
  endpoints:
    - port: metrics              # port name on the Service
      path: /metrics
      interval: 30s
      scheme: http
```

Or use PodMonitor for pods without a Service:

```yaml
apiVersion: monitoring.coreos.com/v1
kind: PodMonitor
metadata:
  name: my-app-pods
  namespace: my-app
spec:
  selector:
    matchLabels:
      app: my-app
  podMetricsEndpoints:
    - port: metrics
      path: /metrics
      interval: 30s
```

---

## Grant monitoring access to your namespace

```bash
# Grant user view access to metrics in a namespace
oc adm policy add-role-to-user monitoring-rules-edit my-user -n my-app
oc adm policy add-role-to-user monitoring-rules-view my-user -n my-app

# Service account for Prometheus to scrape
oc adm policy add-cluster-role-to-user cluster-monitoring-view \
  -z prometheus-k8s -n openshift-monitoring
```

---

## Create custom alerting rules

```yaml
apiVersion: monitoring.coreos.com/v1
kind: PrometheusRule
metadata:
  name: my-app-alerts
  namespace: my-app
  labels:
    openshift.io/prometheus-rule-evaluation-scope: leaf-prometheus
spec:
  groups:
    - name: my-app
      interval: 30s
      rules:
        - alert: MyAppHighErrorRate
          expr: |
            sum(rate(http_requests_total{namespace="my-app",status=~"5.."}[5m]))
            / sum(rate(http_requests_total{namespace="my-app"}[5m])) > 0.05
          for: 2m
          labels:
            severity: critical
            namespace: my-app
          annotations:
            summary: "High error rate in my-app"
            description: "Error rate is {{ $value | humanizePercentage }}"

        - alert: MyAppPodNotReady
          expr: |
            kube_pod_status_ready{namespace="my-app",condition="true"} == 0
          for: 5m
          labels:
            severity: warning
          annotations:
            summary: "Pod {{ $labels.pod }} not ready"
```

---

## Configure Alertmanager

```bash
# Edit Alertmanager config
oc -n openshift-monitoring edit secret alertmanager-main

# The secret contains alertmanager.yaml (base64 encoded)
# Decode, edit, re-encode, apply

# Or use AlertmanagerConfig CR (user workload)
```

```yaml
apiVersion: monitoring.coreos.com/v1beta1
kind: AlertmanagerConfig
metadata:
  name: my-app-alerts
  namespace: my-app
spec:
  route:
    receiver: slack-notify
    groupBy: [alertname, namespace]
    groupWait: 30s
    groupInterval: 5m
    repeatInterval: 4h
    matchers:
      - name: namespace
        value: my-app
  receivers:
    - name: slack-notify
      slackConfigs:
        - apiURL:
            name: slack-webhook-secret
            key: webhookURL
          channel: "#alerts-my-app"
          sendResolved: true
          title: "{{ .GroupLabels.alertname }}"
          text: "{{ range .Alerts }}{{ .Annotations.description }}{{ end }}"
```

---

## Query metrics

```bash
# Via oc (thanos-querier)
oc exec -n openshift-monitoring thanos-querier-<pod> -- \
  curl -s 'http://localhost:9090/api/v1/query?query=up{namespace="my-app"}'

# Port-forward to Prometheus UI
oc port-forward -n openshift-user-workload-monitoring \
  prometheus-user-workload-0 9090:9090

# Port-forward to Alertmanager
oc port-forward -n openshift-monitoring alertmanager-main-0 9093:9093
```

---

## Access Grafana

```bash
# Get Grafana route URL
oc get route grafana -n openshift-monitoring

# Login with your OpenShift credentials
# Pre-built dashboards: cluster capacity, node utilization, workload overview
```

## Useful built-in metrics

```promql
# CPU usage by namespace
sum(rate(container_cpu_usage_seconds_total{namespace!=""}[5m])) by (namespace)

# Memory usage by pod
sum(container_memory_working_set_bytes{container!=""}) by (pod, namespace)

# Pod restart count
increase(kube_pod_container_status_restarts_total[1h]) > 0

# Node disk pressure
kube_node_status_condition{condition="DiskPressure",status="true"} == 1

# PVC usage
kubelet_volume_stats_used_bytes / kubelet_volume_stats_capacity_bytes > 0.8
```
