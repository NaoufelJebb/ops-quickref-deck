# NETWORKING

## Service → Route flow

```
External client
     │
     ▼
HAProxy Router (openshift-ingress namespace)
     │  Matches host header to Route
     ▼
Service (ClusterIP)
     │
     ▼
Pod(s)
```

## Routes

```bash
# List all routes
oc get routes -A

# Describe a route
oc describe route my-app -n my-app

# Get the route URL
oc get route my-app -n my-app -o jsonpath='{.spec.host}'

# Annotate a route (HAProxy tuning)
oc annotate route my-app \
  haproxy.router.openshift.io/timeout=60s \
  haproxy.router.openshift.io/balance=leastconn \
  -n my-app
```

### Useful Route annotations

```yaml
metadata:
  annotations:
    haproxy.router.openshift.io/timeout: 60s
    haproxy.router.openshift.io/balance: leastconn      # roundrobin | leastconn | source
    haproxy.router.openshift.io/rate-limit-connections: "true"
    haproxy.router.openshift.io/rate-limit-connections.rate-http: "100"
    router.openshift.io/cookie_name: my-cookie          # sticky sessions
    haproxy.router.openshift.io/ip_whitelist: "10.0.0.0/8 192.168.0.0/16"
```

---

## NetworkPolicy

By default in OpenShift 4.x, projects allow all ingress within the cluster. Tighten with NetworkPolicy.

### Deny all ingress by default (then allow selectively)

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: deny-all-ingress
  namespace: my-app
spec:
  podSelector: {}              # applies to all pods in namespace
  policyTypes: [Ingress]
  # no ingress rules = deny all
```

### Allow only from another namespace

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-from-frontend
  namespace: my-api
spec:
  podSelector:
    matchLabels:
      app: api
  policyTypes: [Ingress]
  ingress:
    - from:
        - namespaceSelector:
            matchLabels:
              kubernetes.io/metadata.name: my-frontend
```

### Allow from the OpenShift router (for Routes to work)

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-from-router
  namespace: my-app
spec:
  podSelector: {}
  ingress:
    - from:
        - namespaceSelector:
            matchLabels:
              policy-group.network.openshift.io/ingress: ""
```

### Allow from monitoring (Prometheus scraping)

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-from-monitoring
  namespace: my-app
spec:
  podSelector: {}
  ingress:
    - from:
        - namespaceSelector:
            matchLabels:
              kubernetes.io/metadata.name: openshift-monitoring
```

---

## EgressNetworkPolicy (restrict outbound)

```yaml
apiVersion: network.openshift.io/v1
kind: EgressNetworkPolicy
metadata:
  name: restrict-egress
  namespace: my-app
spec:
  egress:
    - type: Allow
      to:
        cidrSelector: 10.0.0.0/8       # internal network
    - type: Allow
      to:
        dnsName: api.partner.com        # specific external host
    - type: Deny
      to:
        cidrSelector: 0.0.0.0/0        # deny everything else
```

---

## DNS

OpenShift uses CoreDNS. Service DNS patterns:

```
<service-name>.<namespace>.svc.cluster.local
<service-name>.<namespace>.svc
<service-name>.<namespace>
<service-name>          (within the same namespace)
```

```bash
# Test DNS from inside a pod
oc rsh my-pod -- nslookup my-service.my-app.svc.cluster.local

# Debug DNS issues
oc rsh my-pod -- curl http://my-service.my-app.svc.cluster.local:8080/health
```

---

## Cluster proxy / egress

For clusters behind a corporate proxy:

```bash
# View current proxy config
oc get proxy cluster -o yaml

# Set cluster proxy
oc edit proxy cluster
# spec:
#   httpProxy: http://proxy.corp.local:3128
#   httpsProxy: http://proxy.corp.local:3128
#   noProxy: .cluster.local,.svc,10.0.0.0/8,172.16.0.0/12
```

---

## IngressController (router) configuration

```bash
# View IngressController
oc get ingresscontroller default -n openshift-ingress-operator -o yaml

# Scale router replicas
oc patch ingresscontroller default -n openshift-ingress-operator \
  --type=merge -p '{"spec":{"replicas":3}}'

# Set custom default certificate for *.apps domain
oc create secret tls custom-certs \
  --cert=wildcard.crt --key=wildcard.key \
  -n openshift-ingress

oc patch ingresscontroller default -n openshift-ingress-operator \
  --type=merge \
  -p '{"spec":{"defaultCertificate":{"name":"custom-certs"}}}'
```

---

## Multus — secondary network interfaces

Attach pods to additional networks (VLANs, SR-IOV, etc.) via NetworkAttachmentDefinition:

```yaml
apiVersion: k8s.cni.cncf.io/v1
kind: NetworkAttachmentDefinition
metadata:
  name: storage-network
  namespace: my-app
spec:
  config: |
    {
      "cniVersion": "0.3.1",
      "type": "macvlan",
      "master": "eth1",
      "mode": "bridge",
      "ipam": {
        "type": "static",
        "addresses": [{"address": "192.168.10.0/24"}]
      }
    }
```

```yaml
# Annotate pod to use it
metadata:
  annotations:
    k8s.v1.cni.cncf.io/networks: storage-network
```
