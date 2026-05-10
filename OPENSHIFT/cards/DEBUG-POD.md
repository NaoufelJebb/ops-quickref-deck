# DEBUG — POD NOT STARTING

## Step 1 — Get the state

```bash
oc get pods -n my-app
oc describe pod <pod> -n my-app      # read Events at the bottom
oc logs <pod> -n my-app              # app logs
oc logs <pod> -n my-app --previous   # if CrashLoopBackOff
```

## Step 2 — Match state to cause

| State | Most likely cause | Next step |
|---|---|---|
| `Pending` | No resources or PVC not bound | Check `describe pod` events |
| `ImagePullBackOff` | Wrong image name or missing pull secret | See image pull section |
| `CrashLoopBackOff` | App crashes at startup | `oc logs --previous` |
| `OOMKilled` | Memory limit too low | Increase `resources.limits.memory` |
| `CreateContainerConfigError` | Secret or ConfigMap missing | `describe pod` → missing volume/env |
| `Init:CrashLoopBackOff` | Init container failing | `oc logs <pod> -c <init-container>` |
| `RunContainerError` | SCC / permission violation | Check events for "forbidden" |

## Step 3 — SCC violation

```bash
# Check events for SCC errors
oc describe pod <pod> -n my-app | grep -i "scc\|forbidden\|unable to validate"

# Test what SCC the pod needs
oc adm policy scc-review -f pod.yaml

# Quick fix — grant anyuid to the service account (check security implications)
oc adm policy add-scc-to-user anyuid -z default -n my-app
```

## Step 4 — Image pull failure

```bash
# Check exact error
oc describe pod <pod> -n my-app | grep -A5 "Failed to pull\|ErrImagePull"

# Add pull secret for private registry
oc create secret docker-registry my-pull-secret \
  --docker-server=registry.example.com \
  --docker-username=user --docker-password=pass \
  -n my-app

# Link to service account
oc secrets link default my-pull-secret --for=pull -n my-app
```

## Step 5 — Debug interactively

```bash
# Shell into a running pod
oc rsh <pod> -n my-app

# Start a debug copy of a crashed pod (overrides entrypoint)
oc debug pod/<pod> -n my-app

# Debug a pod with a different image (e.g. add curl/dig)
oc debug pod/<pod> -n my-app --image=registry.access.redhat.com/ubi9/ubi

# Run a temporary debug pod in the same namespace
oc run debug --rm -it --image=registry.access.redhat.com/ubi9/ubi \
  -n my-app -- bash
```

## Step 6 — Resource constraints

```bash
# Check node resources
oc adm top nodes

# Check namespace quota
oc describe resourcequota -n my-app

# Check pod resource requests
oc describe pod <pod> -n my-app | grep -A6 Requests

# Pending pod — which node can't fit it?
oc describe pod <pod> -n my-app | grep -A10 "Events"
# Look for "Insufficient cpu/memory"
```

## Common fixes

```bash
# Increase memory limit
oc set resources deployment/my-app \
  --limits=memory=512Mi,cpu=500m \
  --requests=memory=256Mi,cpu=100m \
  -n my-app

# Add missing environment variable from secret
oc set env deployment/my-app --from=secret/my-secret -n my-app

# Add missing configmap volume
oc set volume deployment/my-app \
  --add --type=configmap --name=my-config \
  --configmap-name=my-config --mount-path=/etc/config \
  -n my-app

# Restart a deployment (rolling restart)
oc rollout restart deployment/my-app -n my-app
```
