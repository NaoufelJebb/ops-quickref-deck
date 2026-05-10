# INCIDENT

## 1. Triage — what's broken?

```bash
# Cluster operators — find degraded ones
oc get co | grep -v "True.*False.*False"

# Nodes — any NotReady?
oc get nodes

# Problem pods across all namespaces
oc get pods -A | grep -v "Running\|Completed\|Succeeded"

# Recent events — what just happened?
oc get events -A --sort-by='.lastTimestamp' | tail -30
```

---

## 2. Application is down

```bash
# Check pods
oc get pods -n my-app
oc describe pod <pod> -n my-app   # read Events
oc logs <pod> -n my-app --previous

# Check the route resolves
curl -vk https://$(oc get route my-app -n my-app -o jsonpath='{.spec.host}')/health

# Check service has endpoints (pods are registered)
oc get endpoints my-app -n my-app

# Check router pods
oc get pods -n openshift-ingress
```

---

## 3. Node NotReady

```bash
# Describe the node
oc describe node <node> | grep -A15 Conditions

# Check kubelet on the node
oc debug node/<node> -- chroot /host systemctl status kubelet

# Check node logs
oc adm node-logs <node> --unit=kubelet | tail -50

# If node is stuck — cordon + drain + investigate
oc adm cordon <node>
oc adm drain <node> --ignore-daemonsets --delete-emptydir-data --force

# Check MachineConfig update status (may be stuck updating)
oc get mcp
oc describe mcp worker | grep -A10 Conditions
```

---

## 4. Cluster operator degraded

```bash
# Find which operator and why
oc describe co <operator-name>
# Read: Message and Reason in Conditions

# Common operators and where to look:
# authentication   → oc get pods -n openshift-authentication
# ingress          → oc get pods -n openshift-ingress
# kube-apiserver   → oc get pods -n openshift-kube-apiserver
# etcd             → oc get pods -n openshift-etcd
# image-registry   → oc get pods -n openshift-image-registry
# machine-config   → oc get mcp

oc logs -n <namespace> <pod> | grep -E "error|fatal" | tail -30
```

---

## 5. etcd issues (most critical)

```bash
# Check etcd pod status
oc get pods -n openshift-etcd

# Check etcd health
oc exec -n openshift-etcd \
  $(oc get pods -n openshift-etcd -l app=etcd -o name | head -1) -- \
  etcdctl endpoint health --cluster 2>/dev/null

# Check for slow operations
oc logs -n openshift-etcd <etcd-pod> | grep "slow fdatasync\|took too long"
```

---

## 6. Mitigate

```bash
# Rolling restart of a deployment
oc rollout restart deployment/my-app -n my-app

# Force pod restart (delete pod — Deployment recreates it)
oc delete pod <pod> -n my-app

# Scale to 0 then back up
oc scale deployment/my-app --replicas=0 -n my-app
oc scale deployment/my-app --replicas=3 -n my-app

# Roll back a deployment
oc rollout undo deployment/my-app -n my-app

# Cordon a bad node — stop scheduling on it
oc adm cordon <node>
```

---

## 7. Collect diagnostics

```bash
# Full must-gather (upload to Red Hat support)
oc adm must-gather --dest-dir=./must-gather-$(date +%Y%m%d)

# Targeted must-gather for specific operator
oc adm must-gather \
  --image=registry.redhat.io/openshift4/ose-must-gather \
  --dest-dir=./must-gather

# Inspect a specific namespace
oc adm inspect ns/my-app --dest-dir=./inspect-my-app
```
