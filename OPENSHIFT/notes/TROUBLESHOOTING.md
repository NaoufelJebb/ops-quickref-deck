# TROUBLESHOOTING

## Pod not starting

```bash
# Get pod status
oc get pods -n my-app
oc describe pod <pod-name> -n my-app   # read Events section at the bottom

# Get logs
oc logs <pod-name> -n my-app
oc logs <pod-name> -n my-app --previous    # logs from crashed container
oc logs <pod-name> -n my-app -c <container-name>  # multi-container pod

# Shell into a running pod
oc rsh <pod-name> -n my-app

# Debug a crashed pod (starts a copy with sleep)
oc debug pod/<pod-name> -n my-app
```

### Common pod failure states

| State | Cause | Fix |
|---|---|---|
| `Pending` | No node with enough resources, or PVC not bound | Check `oc describe pod` events |
| `ImagePullBackOff` | Image not found or registry auth failure | Check image name, pull secret |
| `CrashLoopBackOff` | Container crashes on start | Check logs with `--previous` |
| `OOMKilled` | Container exceeded memory limit | Increase memory limit |
| `CreateContainerConfigError` | Secret or ConfigMap missing | Check `oc describe pod` |
| `SCC violation` | Pod spec violates Security Context Constraints | See SCC section below |

---

## SCC / permission errors

```bash
# Check SCC violation in describe
oc describe pod <pod> -n my-app | grep -A5 "SCC\|forbidden\|unable to validate"

# Test what SCC a pod spec would need
oc adm policy scc-review -f pod.yaml -n my-app

# Check what SCCs a service account can use
oc adm policy review -z <service-account> -n my-app

# Quick fix — grant anyuid (understand security implications)
oc adm policy add-scc-to-user anyuid -z default -n my-app
```

---

## Route not reachable

```bash
# 1. Verify the route exists
oc get route -n my-app

# 2. Check the service exists and has endpoints
oc get svc -n my-app
oc get endpoints <service-name> -n my-app   # should show pod IPs

# 3. Check router pods
oc get pods -n openshift-ingress
oc logs -n openshift-ingress <router-pod> | grep ERROR

# 4. Test from inside the cluster
oc rsh <any-pod> -n my-app -- curl http://<service>:8080/health

# 5. Verify the service port name matches the route targetPort
oc describe route my-app -n my-app
oc describe svc my-app -n my-app
```

---

## Image pull failures

```bash
# Check the exact error
oc describe pod <pod> -n my-app | grep -A5 "Failed to pull"

# Verify the pull secret exists
oc get secrets -n my-app | grep docker
oc describe secret <pull-secret> -n my-app

# Test image pull manually from a debug pod
oc debug node/<node-name> -- chroot /host crictl pull <image>

# Add pull secret to a namespace
oc create secret docker-registry my-pull-secret \
  --docker-server=registry.example.com \
  --docker-username=myuser \
  --docker-password=mypass \
  -n my-app

# Link to service account
oc secrets link default my-pull-secret --for=pull -n my-app
```

---

## Node issues

```bash
# Node status
oc get nodes
oc describe node <node-name>    # check Conditions and Events

# Node resource usage
oc adm top nodes

# Node is NotReady
oc describe node <node-name> | grep -A10 Conditions

# Debug a node directly
oc debug node/<node-name>
# then: chroot /host

# Check MachineConfig progress
oc get mcp
oc describe mcp worker | grep -A10 Conditions

# Drain a node (for maintenance)
oc adm drain <node-name> --ignore-daemonsets --delete-emptydir-data
oc adm uncordon <node-name>    # bring back

# Cordon without draining
oc adm cordon <node-name>
```

---

## Build failures

```bash
# Watch build logs live
oc logs -f bc/my-app -n my-app

# List builds and status
oc get builds -n my-app

# Describe a failed build
oc describe build my-app-3 -n my-app

# Start a new build with verbose output
oc start-build my-app --follow -n my-app

# Common causes:
# - Source clone failed: check Git URL and credentials
# - S2I assemble failed: check app dependencies / build script
# - Push failed: check image registry access
```

---

## Operator not reconciling

```bash
# Check CSV status
oc get csv -n <namespace>
oc describe csv <csv-name> -n <namespace> | grep -A10 "Conditions\|Phase"

# Check operator pod logs
oc logs deployment/<operator-name> -n <namespace> -f

# Check if CRD is installed
oc get crd | grep <operator>

# Check subscription
oc describe subscription <name> -n <namespace>

# Check install plan
oc get installplan -n <namespace>
oc describe installplan <name> -n <namespace>
```

---

## etcd issues (control plane)

```bash
# Check etcd pod health
oc get pods -n openshift-etcd

# Check etcd cluster health
oc exec -n openshift-etcd etcd-<master-node> -- etcdctl endpoint health \
  --cluster --cacert=/etc/kubernetes/static-pod-certs/configmaps/etcd-serving-ca/ca-bundle.crt \
  --cert=/etc/kubernetes/static-pod-certs/secrets/etcd-all-certs/etcd-serving-<node>.crt \
  --key=/etc/kubernetes/static-pod-certs/secrets/etcd-all-certs/etcd-serving-<node>.key

# Check etcd logs
oc logs -n openshift-etcd etcd-<master-node> | grep -E "error|warn|slow"
```

---

## Cluster operators degraded

```bash
# Check all cluster operators
oc get co

# Find degraded ones
oc get co | grep -v "True.*False.*False"

# Describe degraded operator
oc describe co <operator-name>

# Common degraded operators and fixes:
# authentication: OAuth config error → check oc edit oauth cluster
# ingress: Router pod crash → check openshift-ingress pods
# kube-apiserver: etcd issue → check etcd health
# machine-config: node stuck updating → check mcp status
```

---

## Useful diagnostic commands

```bash
# Cluster info summary
oc cluster-info

# All resources in a namespace
oc get all -n my-app

# Events (sorted by time)
oc get events -n my-app --sort-by='.lastTimestamp'

# Resource usage by pod
oc adm top pods -n my-app

# Cluster-wide resource usage
oc adm top nodes

# Check API server health
curl -k https://$(oc whoami --show-server)/readyz

# Must-gather (collect diagnostic bundle)
oc adm must-gather --dest-dir=./must-gather

# Must-gather for a specific operator
oc adm must-gather --image=registry.redhat.io/openshift4/ose-must-gather
```
