# ARCHITECTURE

## Cluster topology

```
                        [ OpenShift Cluster ]

Control Plane (Masters)              Workers
┌─────────────────────┐         ┌────────────────┐
│  API Server         │         │  kubelet       │
│  etcd               │─────────│  CRI-O runtime │
│  Controller Manager │         │  Your workloads│
│  Scheduler          │         └────────────────┘
│  OAuth server       │
│  OpenShift Router   │         Infra Nodes (optional)
└─────────────────────┘         ┌────────────────┐
                                │  HAProxy router│
                                │  Image registry│
                                │  Monitoring    │
                                └────────────────┘
```

## Node types

| Role | Purpose | Taint |
|---|---|---|
| **Control plane / master** | API, scheduler, etcd | `node-role.kubernetes.io/master` |
| **Worker** | Run application workloads | none |
| **Infra** | Router, registry, monitoring — move off workers | custom |
| **Edge** (SNO) | Single-node all-in-one | all roles |

Recommended production: 3 control plane + 2+ workers (odd etcd count for quorum).

## OpenShift flavors

| Variant | Description |
|---|---|
| **OCP (OpenShift Container Platform)** | Self-managed, on-prem or cloud, full control |
| **ROSA** | Red Hat OpenShift Service on AWS — managed control plane |
| **ARO** | Azure Red Hat OpenShift — managed |
| **RHOIC** | OpenShift on IBM Cloud — managed |
| **OSD** | OpenShift Dedicated — Red Hat managed on AWS/GCP |
| **MicroShift** | Lightweight edge variant |
| **SNO** | Single Node OpenShift — dev / edge |
| **CRC** | CodeReady Containers — local laptop dev |

## Networking

### Default CNI: OVN-Kubernetes (4.12+) / OpenShift SDN (older)

```
Pod → Service (ClusterIP) → Route (HAProxy) → External client
```

### Network namespaces

Projects have network isolation by default via NetworkPolicy. OVN-Kubernetes supports:
- `NetworkPolicy` (standard K8s)
- `EgressNetworkPolicy` (restrict outbound from namespace)
- `MultiNetworkPolicy` (Multus — secondary NICs)

### Ingress — HAProxy Router

```bash
# Check router pods
kubectl get pods -n openshift-ingress

# Check IngressController
oc get ingresscontroller -n openshift-ingress-operator

# Default wildcard domain: *.apps.<cluster-domain>
# Routes auto-get: <service>-<namespace>.apps.<cluster-domain>
```

### Service types

```
ClusterIP   → internal only (default)
NodePort    → avoid in OpenShift, use Routes instead
LoadBalancer → cloud LB (AWS ELB, Azure LB)
Route       → OpenShift preferred — HAProxy, TLS, custom hostnames
```

## Storage

| Type | Use for |
|---|---|
| `EmptyDir` | Ephemeral, pod lifetime |
| `PersistentVolumeClaim` | Stateful workloads |
| `ConfigMap` / `Secret` | Config and credentials |
| `HostPath` | Node-local (avoid) |

```bash
# List storage classes
oc get storageclass

# Create a PVC
oc create -f - <<EOF
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: my-pvc
spec:
  accessModes: [ReadWriteOnce]
  resources:
    requests:
      storage: 10Gi
  storageClassName: gp3-csi
EOF
```

## Key ports

| Port | Service |
|---|---|
| `6443` | API Server (HTTPS) |
| `443` | Router HTTPS (apps) |
| `80` | Router HTTP (apps) |
| `22623` | Machine Config Server |
| `2379–2380` | etcd |
| `10250` | kubelet |

## MachineConfig / RHCOS

Worker and control plane nodes run RHCOS (Red Hat CoreOS) — immutable, managed by the Machine Config Operator.

```bash
# List MachineConfigPools
oc get mcp

# List MachineConfigs
oc get mc

# Check node update status
oc get mcp worker -o yaml | grep -A5 status

# Never SSH and manually edit node files — use MachineConfig instead
```
