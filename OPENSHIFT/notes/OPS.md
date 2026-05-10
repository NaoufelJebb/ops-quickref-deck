# OPS

## Daily health checks

```bash
# Cluster operators — all should be Available=True, Progressing=False, Degraded=False
oc get co

# Node status — all Ready
oc get nodes

# Pod issues across all namespaces
oc get pods -A | grep -v "Running\|Completed"

# etcd health
oc get pods -n openshift-etcd

# Pending PVCs
oc get pvc -A | grep -v Bound

# Check MachineConfigPool (node update status)
oc get mcp
```

---

## Cluster upgrades

```bash
# Check current version
oc get clusterversion

# Check available updates
oc adm upgrade

# View update graph
oc adm upgrade --to-latest=false   # shows available channels and versions

# Switch channel (e.g. stable-4.14 → stable-4.15)
oc patch clusterversion version --type=merge \
  -p '{"spec":{"channel":"stable-4.15"}}'

# Trigger upgrade to a specific version
oc adm upgrade --to=4.15.3

# Watch upgrade progress
oc get clusterversion -w
oc get co -w       # watch operator upgrades
oc get nodes -w    # watch node reboots

# Check upgrade history
oc get clusterversion -o jsonpath='{.items[0].status.history}' | jq .
```

Upgrade order: Control plane updates first, then workers via MachineConfigPool.

---

## Certificate management

### Check certificate expiry

```bash
# API server serving cert
oc get secret kube-apiserver-to-kubelet-signer \
  -n openshift-kube-apiserver-operator \
  -o jsonpath='{.metadata.annotations.auth\.openshift\.io/certificate-not-after}'

# All certificates (must-gather approach)
oc adm must-gather -- gather_network_logs

# Check router (ingress) certificate
oc get secret router-certs-default -n openshift-ingress \
  -o jsonpath='{.data.tls\.crt}' | base64 -d | openssl x509 -noout -dates
```

### Replace the default ingress (*.apps) certificate

```bash
# 1. Create a secret with the wildcard cert
oc create secret tls custom-ingress-cert \
  --cert=wildcard.apps.crt \
  --key=wildcard.apps.key \
  -n openshift-ingress

# 2. Patch the IngressController to use it
oc patch ingresscontroller default \
  -n openshift-ingress-operator \
  --type=merge \
  -p '{"spec":{"defaultCertificate":{"name":"custom-ingress-cert"}}}'
```

### Replace the API server certificate

```bash
# 1. Create secret in openshift-config
oc create secret tls api-server-cert \
  --cert=api.crt --key=api.key \
  -n openshift-config

# 2. Patch the API server
oc patch apiserver cluster --type=merge \
  -p '{"spec":{"servingCerts":{"namedCertificates":[{"names":["api.example.com"],"servingCertificate":{"name":"api-server-cert"}}]}}}'
```

---

## etcd backup

```bash
# Run the etcd backup script on a control plane node
oc debug node/<master-node> -- chroot /host \
  /usr/local/bin/cluster-backup.sh /home/core/assets/backup

# Copy backup from node
oc debug node/<master-node> -- \
  chroot /host ls -la /home/core/assets/backup

# The backup contains:
# - etcd snapshot
# - static pod manifests
# Store offsite — used for disaster recovery
```

---

## Scale and node management

```bash
# Check MachineSet (controls worker node count in cloud)
oc get machineset -n openshift-machine-api

# Scale a MachineSet
oc scale machineset <machineset-name> --replicas=5 \
  -n openshift-machine-api

# Label a node (for node selectors)
oc label node <node-name> node-role.kubernetes.io/infra=""

# Add infra taint (prevent workloads on infra nodes)
oc adm taint node <node-name> node-role.kubernetes.io/infra=reserved:NoSchedule

# Cordon (stop scheduling new pods)
oc adm cordon <node-name>

# Drain (evict all pods safely)
oc adm drain <node-name> --ignore-daemonsets --delete-emptydir-data --force

# Uncordon (return to scheduling)
oc adm uncordon <node-name>
```

---

## Image registry

```bash
# Check registry status
oc get config.imageregistry cluster

# Expose the registry externally (default is internal only)
oc patch config.imageregistry cluster \
  --type=merge -p '{"spec":{"defaultRoute":true}}'

# Get registry route
oc get route default-route -n openshift-image-registry

# Log in to the internal registry
oc registry login

# Or manually:
TOKEN=$(oc whoami -t)
REGISTRY=$(oc get route default-route -n openshift-image-registry -o jsonpath='{.spec.host}')
docker login -u $(oc whoami) -p $TOKEN $REGISTRY

# List images in the registry
oc get imagestreams -A
```

---

## Ops checklist

- [ ] `oc get co` — all operators Available, none Degraded
- [ ] `oc get nodes` — all nodes Ready
- [ ] Pods checked — no unexpected CrashLoopBackOff or Pending
- [ ] etcd healthy — 3 members, all connected
- [ ] MachineConfigPool updated — no nodes in degraded state
- [ ] Certificate expiry monitored — alert at ≤ 30 days
- [ ] Cluster update channel set correctly for planned upgrades
- [ ] etcd backup running and stored offsite
- [ ] ResourceQuotas reviewed — not at 100% in critical namespaces
- [ ] Alertmanager routing tested — notifications reaching teams
