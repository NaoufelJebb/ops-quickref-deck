# CONCEPTS

## What OpenShift is

OpenShift is Red Hat's enterprise Kubernetes distribution. It adds an opinionated layer on top of Kubernetes: built-in CI/CD (Pipelines), a container registry, developer tooling, stricter security defaults (SCCs), role-based multi-tenancy, and a web console. Everything that works in Kubernetes works in OpenShift — OpenShift adds on top.

```
OpenShift = Kubernetes + CRI-O runtime + OLM (Operator Lifecycle Manager)
          + Routes (ingress) + ImageStreams + BuildConfigs
          + SCCs (Security Context Constraints) + Projects (namespaces)
          + OAuth server + Web console
```

## OpenShift vs Kubernetes — key differences

| Concept | Kubernetes | OpenShift |
|---|---|---|
| Namespace | `Namespace` | `Project` (wrapper around Namespace) |
| Ingress | `Ingress` | `Route` (HAProxy-based, simpler TLS) |
| Image management | External registry | `ImageStream` + internal registry |
| Build | External CI | `BuildConfig` (S2I, Dockerfile, custom) |
| Security | `PodSecurityAdmission` | `SCC` (Security Context Constraints) — stricter |
| Operators | Manual | OLM (Operator Lifecycle Manager) manages lifecycle |
| Auth | External OIDC | Built-in OAuth server + identity providers |
| CLI | `kubectl` | `oc` (superset of kubectl) |

## Core objects

| Object | What it is |
|---|---|
| **Project** | Isolated namespace with quota, RBAC, and network policy |
| **Pod** | One or more containers — same as Kubernetes |
| **Deployment** / **DeploymentConfig** | Manages pod replicas — DC is OpenShift-native with triggers |
| **Service** | Internal L4 load balancer — same as Kubernetes |
| **Route** | External HTTP/HTTPS exposure — OpenShift's Ingress |
| **ImageStream** | Tracks container image versions and triggers deployments |
| **BuildConfig** | Defines how to build an image from source |
| **SCC** | Security Context Constraints — what a pod is allowed to do |
| **Operator** | A controller that manages a complex application lifecycle |
| **OperatorHub** | Marketplace of available operators |
| **MachineConfig** | Node-level OS configuration (CoreOS / RHCOS) |
| **ClusterOperator** | Built-in OpenShift component operators (DNS, ingress, auth…) |

## CLI — `oc` vs `kubectl`

`oc` is a superset of `kubectl`. All `kubectl` commands work. `oc` adds:

```bash
oc new-project          # create project
oc new-app              # deploy from source/image
oc expose               # create a Route from a Service
oc rollout              # manage deployments
oc start-build          # trigger a build
oc logs -f bc/myapp     # build logs
oc rsh <pod>            # remote shell
oc debug node/<node>    # debug a node
oc adm                  # cluster administration
oc login                # authenticate
oc whoami               # current user/token
oc project              # switch project
```

## Key namespaces

| Namespace | Contains |
|---|---|
| `openshift` | Shared templates and imagestreams |
| `openshift-config` | Cluster configuration (OAuth, proxy, ingress) |
| `openshift-ingress` | HAProxy router pods |
| `openshift-image-registry` | Internal image registry |
| `openshift-monitoring` | Prometheus, Alertmanager, Grafana stack |
| `openshift-logging` | Log aggregation (Loki / Elasticsearch) |
| `openshift-operators` | Cluster-wide operators |
| `kube-system` | Core Kubernetes components |
