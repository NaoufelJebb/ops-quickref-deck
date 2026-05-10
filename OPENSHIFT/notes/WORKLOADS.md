# WORKLOADS

## Deploy from an image

```bash
# Deploy from a public image
oc new-app nginx:latest --name=my-nginx -n my-app

# Deploy from internal registry
oc new-app image-registry.openshift-image-registry.svc:5000/my-app/my-image:latest

# Deploy with environment variables
oc new-app my-image:latest -e DB_HOST=postgres -e DB_PORT=5432

# Expose as a Route
oc expose svc/my-nginx
```

## Deployment vs DeploymentConfig

| | `Deployment` | `DeploymentConfig` |
|---|---|---|
| Origin | Kubernetes native | OpenShift-native (legacy) |
| Triggers | Manual / ArgoCD | Image change, config change (auto) |
| Strategy | RollingUpdate / Recreate | Rolling / Recreate / Custom |
| Recommendation | ✅ Prefer this | Legacy — avoid for new apps |

```bash
# Scale a deployment
oc scale deployment my-app --replicas=3 -n my-app

# Rollout status
oc rollout status deployment/my-app -n my-app

# Rollback
oc rollout undo deployment/my-app -n my-app

# Rollout history
oc rollout history deployment/my-app -n my-app

# Pause / resume rollout
oc rollout pause deployment/my-app -n my-app
oc rollout resume deployment/my-app -n my-app
```

## Routes

Routes expose services externally via the HAProxy router.

```bash
# Expose a service as a Route (auto hostname)
oc expose svc/my-app -n my-app
# → my-app-my-app.apps.<cluster-domain>

# Expose with custom hostname
oc expose svc/my-app --hostname=api.example.com -n my-app

# List routes
oc get routes -n my-app

# Delete route
oc delete route my-app -n my-app
```

### Route with TLS (edge termination)

```yaml
apiVersion: route.openshift.io/v1
kind: Route
metadata:
  name: my-app
  namespace: my-app
spec:
  host: api.example.com
  to:
    kind: Service
    name: my-app
    weight: 100
  port:
    targetPort: 8080
  tls:
    termination: edge               # edge | passthrough | reencrypt
    insecureEdgeTerminationPolicy: Redirect
    certificate: |
      -----BEGIN CERTIFICATE-----
      ...
    key: |
      -----BEGIN RSA PRIVATE KEY-----
      ...
```

### TLS termination types

| Type | Where TLS ends | Use for |
|---|---|---|
| `edge` | At the router | Most apps — router decrypts, sends HTTP to pod |
| `passthrough` | At the pod | Apps that manage their own TLS |
| `reencrypt` | At the router + re-encrypt to pod | When pod TLS is required end-to-end |

---

## ImageStreams

ImageStreams track image versions and trigger deployments on change.

```bash
# Create an imagestream tag pointing to an external image
oc tag docker.io/nginx:latest my-app/nginx:stable -n my-app

# Import an image into the internal registry
oc import-image nginx:latest \
  --from=docker.io/nginx:latest \
  --confirm -n my-app

# List imagestreams
oc get is -n my-app

# Watch image tags
oc describe is nginx -n my-app
```

---

## BuildConfig — build from source (S2I)

Source-to-Image (S2I) builds a runnable image from your source code without a Dockerfile.

```bash
# Create a BuildConfig from a Git repo (S2I)
oc new-app python:3.11~https://github.com/myorg/myapp.git \
  --name=my-python-app -n my-app

# Create from Dockerfile in a Git repo
oc new-build https://github.com/myorg/myapp.git \
  --strategy=docker --name=my-app -n my-app

# Start a build manually
oc start-build my-app -n my-app

# Watch build logs
oc logs -f bc/my-app -n my-app

# List builds
oc get builds -n my-app

# Cancel a running build
oc cancel-build my-app-3 -n my-app
```

### BuildConfig YAML

```yaml
apiVersion: build.openshift.io/v1
kind: BuildConfig
metadata:
  name: my-app
  namespace: my-app
spec:
  source:
    type: Git
    git:
      uri: https://github.com/myorg/myapp.git
      ref: main
    contextDir: /app
  strategy:
    type: Source                    # S2I
    sourceStrategy:
      from:
        kind: ImageStreamTag
        name: python:3.11
        namespace: openshift
  output:
    to:
      kind: ImageStreamTag
      name: my-app:latest
  triggers:
    - type: ConfigChange
    - type: ImageChange
```

---

## ConfigMaps and Secrets

```bash
# Create ConfigMap from literals
oc create configmap my-config \
  --from-literal=DB_HOST=postgres \
  --from-literal=DB_PORT=5432 -n my-app

# Create ConfigMap from a file
oc create configmap app-config --from-file=config.yaml -n my-app

# Create Secret (base64 encoded automatically)
oc create secret generic my-secret \
  --from-literal=DB_PASSWORD=supersecret \
  --from-literal=API_KEY=abc123 -n my-app

# Create TLS secret
oc create secret tls my-tls \
  --cert=tls.crt --key=tls.key -n my-app

# Mount secret as env vars in a deployment
oc set env deployment/my-app \
  --from=secret/my-secret -n my-app

# Mount configmap as volume
oc set volume deployment/my-app \
  --add --name=config-vol \
  --type=configmap \
  --configmap-name=my-config \
  --mount-path=/etc/config -n my-app
```

---

## Health probes

```yaml
spec:
  containers:
    - name: my-app
      livenessProbe:
        httpGet:
          path: /health/live
          port: 8080
        initialDelaySeconds: 10
        periodSeconds: 10
        failureThreshold: 3
      readinessProbe:
        httpGet:
          path: /health/ready
          port: 8080
        initialDelaySeconds: 5
        periodSeconds: 5
        failureThreshold: 3
      startupProbe:
        httpGet:
          path: /health/live
          port: 8080
        failureThreshold: 30
        periodSeconds: 10
```
