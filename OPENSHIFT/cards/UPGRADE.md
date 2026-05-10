# CLUSTER UPGRADE

## Before you start

```bash
# 1. Check current version and health
oc get clusterversion
oc get co | grep -v "True.*False.*False"    # no degraded operators
oc get nodes                                # all Ready

# 2. Check available updates
oc adm upgrade

# 3. Verify etcd health
oc get pods -n openshift-etcd

# 4. Check MachineConfigPools are not updating
oc get mcp
# All should show: UPDATED=True, UPDATING=False, DEGRADED=False
```

---

## Switch channel (minor version upgrade)

```bash
# Check current channel
oc get clusterversion -o jsonpath='{.items[0].spec.channel}'

# Switch channel (e.g. to pick up 4.15 updates)
oc patch clusterversion version --type=merge \
  -p '{"spec":{"channel":"stable-4.15"}}'

# Confirm available updates in the new channel
oc adm upgrade
```

---

## Start the upgrade

```bash
# Upgrade to latest in current channel
oc adm upgrade --to-latest

# Or upgrade to a specific version
oc adm upgrade --to=4.15.3

# Follow progress
watch oc get clusterversion
```

---

## Monitor upgrade progress

```bash
# Overall version status
oc get clusterversion

# Individual operator upgrades
watch oc get co

# Node reboots (workers update one by one via MachineConfigPool)
watch oc get nodes

# MachineConfigPool progress
watch oc get mcp

# Upgrade history
oc get clusterversion -o jsonpath='{.items[0].status.history}' | jq .
```

---

## Upgrade order (automatic)

```
1. Control plane operators update
2. Master nodes reboot (one at a time — etcd quorum maintained)
3. Worker MachineConfigPool updates nodes (batch, configurable)
4. Infra nodes update
```

---

## If upgrade stalls

```bash
# Check which operator is blocking
oc get co | grep -v "True.*False.*False"
oc describe co <degraded-operator>

# Check cluster version conditions
oc describe clusterversion version

# Check if a node is stuck
oc get nodes
oc describe node <node> | grep -A10 Conditions
oc get mcp worker -o yaml | grep -A20 status

# Force drain a stuck node (if safe)
oc adm drain <node> --ignore-daemonsets --delete-emptydir-data --force
```

---

## Post-upgrade checks

```bash
# All operators Available
oc get co | grep -v "True.*False.*False"

# All nodes Ready on new version
oc get nodes -o wide

# All MCPs updated
oc get mcp

# Cluster version shows correct version
oc get clusterversion

# Verify workloads still running
oc get pods -A | grep -v "Running\|Completed\|Succeeded"
```

---

## Upgrade tips

- Schedule during maintenance window — workers reboot during upgrade
- `stable` channel is safer than `fast` or `candidate`
- Skip-level upgrades (e.g. 4.13 → 4.15) are allowed only via specific paths — check the update graph at `access.redhat.com/labs/ocpupgradegraph`
- If using a proxy, ensure `noProxy` includes `.cluster.local` and internal CIDRs before upgrading
