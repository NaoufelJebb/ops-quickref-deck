# KUBERNETES

## Install via Helm

```bash
helm repo add kong https://charts.konghq.com && helm repo update

helm install kong kong/ingress \
  --namespace kong --create-namespace \
  -f values.yaml
```

## CRD → Kong concept mapping

| CRD | Equivalent |
|---|---|
| `Ingress` / `HTTPRoute` | Route |
| `KongPlugin` | Plugin (namespaced) |
| `KongClusterPlugin` | Global Plugin |
| `KongConsumer` | Consumer |
| `KongIngress` | Route/Service/Upstream overrides |

---

## Ingress (standard)

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: my-ingress
  annotations:
    konghq.com/strip-path: "true"
    konghq.com/plugins: rate-limit,jwt-auth   # comma-separated KongPlugin names
spec:
  ingressClassName: kong
  rules:
    - host: api.example.com
      http:
        paths:
          - path: /api/v1
            pathType: Prefix
            backend:
              service:
                name: my-service
                port:
                  number: 8080
```

## KongPlugin

```yaml
apiVersion: configuration.konghq.com/v1
kind: KongPlugin
metadata:
  name: rate-limit
  namespace: default
plugin: rate-limiting
config:
  minute: 1000
  policy: redis
  redis_host: redis.default.svc.cluster.local
  redis_port: 6379
```

## KongClusterPlugin (global)

```yaml
apiVersion: configuration.konghq.com/v1
kind: KongClusterPlugin
metadata:
  name: prometheus-global
  labels:
    global: "true"               # this makes it apply everywhere
plugin: prometheus
config:
  per_consumer: true
  latency_metrics: true
  status_code_metrics: true
```

## KongConsumer + credential Secret

```yaml
apiVersion: configuration.konghq.com/v1
kind: KongConsumer
metadata:
  name: my-app
  annotations:
    kubernetes.io/ingress.class: kong
username: my-app
credentials:
  - my-app-jwt                   # references Secret below
---
apiVersion: v1
kind: Secret
metadata:
  name: my-app-jwt
  labels:
    konghq.com/credential: jwt
type: Opaque
stringData:
  algorithm: HS256
  secret: "your-secret-here"
```

## HTTPRoute (Gateway API — modern)

```yaml
apiVersion: gateway.networking.k8s.io/v1
kind: HTTPRoute
metadata:
  name: my-route
  annotations:
    konghq.com/plugins: rate-limit
spec:
  parentRefs:
    - name: kong
      namespace: kong
  hostnames:
    - api.example.com
  rules:
    - matches:
        - path:
            type: PathPrefix
            value: /api/v1
      backendRefs:
        - name: my-service
          port: 8080
```

## Useful kubectl commands

```bash
# List all Kong resources
kubectl get kongplugins,kongconsumers,kongclusterplugins -A

# Check controller logs
kubectl logs -n kong deployment/kong-controller -f

# Check proxy logs
kubectl logs -n kong deployment/kong-proxy -f

# Describe a plugin (see events/errors)
kubectl describe kongplugin rate-limit

# All Ingresses managed by Kong
kubectl get ingress -A \
  -o jsonpath='{range .items[?(@.spec.ingressClassName=="kong")]}{.namespace}/{.name}{"\n"}{end}'
```
