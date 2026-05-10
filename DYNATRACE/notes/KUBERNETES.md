# KUBERNETES

## Install Dynatrace Operator

```bash
# 1. Create namespace
kubectl create namespace dynatrace

# 2. Install operator via Helm
helm repo add dynatrace https://raw.githubusercontent.com/Dynatrace/dynatrace-operator/main/config/helm/repos/stable
helm repo update

helm install dynatrace-operator dynatrace/dynatrace-operator \
  --namespace dynatrace \
  --atomic

# 3. Create API token secret
kubectl create secret generic dynakube \
  --namespace dynatrace \
  --from-literal="apiToken=<API_TOKEN>" \
  --from-literal="dataIngestToken=<DATA_INGEST_TOKEN>"
```

Required API token scopes:
- `activeGateTokenManagement.create`
- `entities.read`
- `settings.read` / `settings.write`
- `DataExport`
- `InstallerDownload`

---

## DynaKube CR — core config

```yaml
apiVersion: dynatrace.com/v1beta1
kind: DynaKube
metadata:
  name: dynakube
  namespace: dynatrace
spec:
  apiUrl: https://<env-id>.live.dynatrace.com/api

  oneAgent:
    cloudNativeFullStack:           # recommended mode
      env:
        - name: ONEAGENT_ENABLE_VOLUME_STORAGE
          value: "true"
      resources:
        requests:
          cpu: 100m
          memory: 512Mi
        limits:
          cpu: 1
          memory: 1.5Gi

  activeGate:
    capabilities:
      - routing                     # route OneAgent traffic
      - kubernetes-monitoring       # cluster-level metrics
      - dynatrace-api               # API access from within cluster
    resources:
      requests:
        cpu: 500m
        memory: 512Mi
      limits:
        cpu: 1
        memory: 1.5Gi

  metadataEnrichment:
    enabled: true                   # enrich spans/metrics with K8s metadata

  namespaceSelector:                # monitor only specific namespaces
    matchLabels:
      monitoring: dynatrace
```

---

## Injection modes

| Mode | How it works | Restart required? |
|---|---|---|
| `classicFullStack` | DaemonSet + init container | Yes |
| `cloudNativeFullStack` | DaemonSet + CSI driver injection | No |
| `applicationMonitoring` | Webhook injection only (no DaemonSet) | No |
| `hostMonitoring` | Infrastructure only, no APM | No |

`cloudNativeFullStack` is the recommended mode for K8s — no pod restarts needed for injection.

---

## Annotate workloads for custom config

```yaml
# Disable monitoring for a deployment
metadata:
  annotations:
    oneagent.dynatrace.com/inject: "false"

# Override process name
metadata:
  annotations:
    oneagent.dynatrace.com/technologies: "java"
    oneagent.dynatrace.com/process-name: "my-custom-name"
```

---

## Prometheus scraping from K8s

```yaml
# Annotate a pod to have OneAgent scrape its /metrics endpoint
metadata:
  annotations:
    metrics.dynatrace.com/scrape: "true"
    metrics.dynatrace.com/port: "9090"
    metrics.dynatrace.com/path: "/metrics"
    metrics.dynatrace.com/secure: "false"
```

Scraped metrics appear as `ext:prometheus.*` in Dynatrace.

---

## K8s monitoring coverage

| What | How detected |
|---|---|
| Node CPU/Memory | OneAgent DaemonSet + ActiveGate |
| Pod CPU/Memory | ActiveGate kubernetes-monitoring capability |
| Container restarts | ActiveGate |
| Deployment replica counts | ActiveGate |
| Service traces (PurePath) | OneAgent injection into pods |
| Custom app metrics | OTLP or Metrics API |
| K8s events | ActiveGate |

---

## Useful kubectl commands

```bash
# Check operator status
kubectl get pods -n dynatrace

# Check DynaKube status
kubectl describe dynakube dynakube -n dynatrace

# Check which pods have OneAgent injected
kubectl get pods -A -o jsonpath='{range .items[*]}{.metadata.namespace}/{.metadata.name}{"\t"}{.metadata.annotations.oneagent\.dynatrace\.com/inject}{"\n"}{end}'

# Operator logs
kubectl logs -n dynatrace deployment/dynatrace-operator -f

# OneAgent pod logs (per node)
kubectl logs -n dynatrace daemonset/dynakube-oneagent -f

# ActiveGate logs
kubectl logs -n dynatrace statefulset/dynakube-activegate -f
```

---

## Namespace-scoped monitoring

To monitor only selected namespaces, label them and use `namespaceSelector` in the DynaKube CR:

```bash
# Label a namespace for monitoring
kubectl label namespace my-app monitoring=dynatrace

# Exclude a namespace
kubectl annotate namespace kube-system \
  oneagent.dynatrace.com/inject=false
```
