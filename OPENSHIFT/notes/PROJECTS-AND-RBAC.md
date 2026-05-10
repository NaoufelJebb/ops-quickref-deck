# PROJECTS AND RBAC

## Projects

A Project is a Kubernetes Namespace with OpenShift metadata and defaults applied automatically (network policy, default RBAC, resource quotas).

```bash
# Create project
oc new-project my-app \
  --display-name="My Application" \
  --description="Production workloads"

# Switch project
oc project my-app

# List all projects (your access)
oc projects

# Delete project (deletes all resources inside)
oc delete project my-app

# Admin — list all projects
oc get projects
```

## Resource Quotas

Limit total resource consumption in a project.

```yaml
apiVersion: v1
kind: ResourceQuota
metadata:
  name: project-quota
  namespace: my-app
spec:
  hard:
    requests.cpu: "4"
    requests.memory: 8Gi
    limits.cpu: "8"
    limits.memory: 16Gi
    pods: "20"
    persistentvolumeclaims: "10"
    services.loadbalancers: "0"    # prevent LB services
    requests.storage: 100Gi
```

```bash
oc get resourcequota -n my-app
oc describe resourcequota project-quota -n my-app
```

## LimitRange

Set default requests/limits for pods that don't specify them.

```yaml
apiVersion: v1
kind: LimitRange
metadata:
  name: default-limits
  namespace: my-app
spec:
  limits:
    - type: Container
      default:
        cpu: 500m
        memory: 256Mi
      defaultRequest:
        cpu: 100m
        memory: 128Mi
      max:
        cpu: "2"
        memory: 2Gi
```

---

## RBAC

OpenShift RBAC = Kubernetes RBAC + OpenShift-specific cluster roles.

### Built-in roles

| Role | Access |
|---|---|
| `cluster-admin` | Full cluster access — use sparingly |
| `admin` | Full access within a project |
| `edit` | Create/update/delete resources, not RBAC |
| `view` | Read-only within a project |
| `basic-user` | Can see own info, create projects |
| `self-provisioner` | Can create new projects |
| `registry-admin` | Full image registry access |
| `registry-viewer` | Pull images only |

### Assign roles

```bash
# Add user as project admin
oc adm policy add-role-to-user admin jane -n my-app

# Add user as view-only
oc adm policy add-role-to-user view john -n my-app

# Add group to role
oc adm policy add-role-to-group edit dev-team -n my-app

# Remove role
oc adm policy remove-role-from-user edit john -n my-app

# Make user cluster-admin (use with extreme caution)
oc adm policy add-cluster-role-to-user cluster-admin jane

# List role bindings in a project
oc get rolebindings -n my-app
oc describe rolebinding admin -n my-app

# List cluster role bindings
oc get clusterrolebindings | grep jane
```

### Custom role

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: deployment-manager
  namespace: my-app
rules:
  - apiGroups: ["apps"]
    resources: ["deployments", "replicasets"]
    verbs: ["get", "list", "watch", "create", "update", "patch"]
  - apiGroups: [""]
    resources: ["pods", "pods/log"]
    verbs: ["get", "list", "watch"]
```

```bash
oc apply -f role.yaml
oc adm policy add-role-to-user deployment-manager ci-user -n my-app
```

---

## Service Accounts

```bash
# Create a service account
oc create serviceaccount my-sa -n my-app

# Assign a role
oc adm policy add-role-to-user edit -z my-sa -n my-app
# -z = service account in current namespace

# Get the SA token (for CI/CD or API access)
oc create token my-sa -n my-app --duration=24h

# List service accounts
oc get sa -n my-app
```

---

## Security Context Constraints (SCCs)

SCCs are OpenShift's pod security layer — stricter than Kubernetes Pod Security Admission.

### Built-in SCCs (most to least restrictive)

| SCC | Allows |
|---|---|
| `restricted-v2` | Default — no root, no host access, random UID |
| `restricted` | Legacy default (pre-4.11) |
| `nonroot` | Must run as non-root, specific UID allowed |
| `nonroot-v2` | nonroot with Seccomp defaults |
| `anyuid` | Any UID including root — avoid unless needed |
| `privileged` | Full host access — operators / infra only |
| `hostnetwork` | Uses host network namespace |
| `hostmount-anyuid` | Host paths + any UID |

```bash
# Check what SCC a pod is running under
oc get pod my-pod -o yaml | grep scc
oc describe pod my-pod | grep scc

# List all SCCs
oc get scc

# Grant SCC to a service account
oc adm policy add-scc-to-user anyuid -z my-sa -n my-app

# Check which SCC a service account can use
oc adm policy who-can use scc anyuid

# Test if a pod spec would be admitted
oc adm policy scc-review -f pod.yaml
```

### Fix "container has runAsNonRoot and image has non-numeric user" errors

```bash
# Option 1: add anyuid SCC to service account (quick, less secure)
oc adm policy add-scc-to-user anyuid -z default -n my-app

# Option 2: fix the image to use a numeric UID (correct approach)
# Dockerfile: USER 1001

# Option 3: set runAsUser in the pod spec
securityContext:
  runAsUser: 1001
  runAsNonRoot: true
```
