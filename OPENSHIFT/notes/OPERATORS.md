# OPERATORS

## What operators are

An Operator is a controller that encodes operational knowledge for managing a complex application (databases, message queues, monitoring stacks). It watches custom resources (CRs) and reconciles the cluster state to match.

```
You create a CR → Operator watches it → Operator creates/manages Pods, Services, Secrets, etc.
```

## Operator Lifecycle Manager (OLM)

OLM manages operator installation, upgrades, and dependencies in OpenShift.

```bash
# List available operators in OperatorHub
oc get packagemanifests -n openshift-marketplace

# Search for an operator
oc get packagemanifests -n openshift-marketplace | grep -i postgres

# List installed operators (all namespaces)
oc get csv -A

# List installed operators in a namespace
oc get csv -n my-app

# Check operator status
oc get csv -n my-app -o wide
```

## Install an operator via the CLI

```yaml
# 1. Create a Subscription (triggers OLM to install)
apiVersion: operators.coreos.com/v1alpha1
kind: Subscription
metadata:
  name: my-operator
  namespace: openshift-operators       # cluster-wide operators go here
  # or: namespace: my-app             # namespace-scoped operators
spec:
  channel: stable
  name: my-operator-package-name      # from packagemanifest
  source: redhat-operators            # or: certified-operators, community-operators, redhat-marketplace
  sourceNamespace: openshift-marketplace
  installPlanApproval: Automatic      # Automatic | Manual
```

```bash
# Apply subscription
oc apply -f subscription.yaml

# Watch install progress
oc get csv -n openshift-operators -w

# If installPlanApproval: Manual — approve the install plan
oc get installplan -n openshift-operators
oc patch installplan <name> -n openshift-operators \
  --type=merge -p '{"spec":{"approved":true}}'
```

## Common operators

| Operator | Package name | Source |
|---|---|---|
| OpenShift Pipelines (Tekton) | `openshift-pipelines-operator-rh` | redhat-operators |
| OpenShift GitOps (ArgoCD) | `openshift-gitops-operator` | redhat-operators |
| Cert-Manager | `cert-manager` | community-operators |
| Jaeger / Tracing | `jaeger-product` | redhat-operators |
| Elasticsearch / Logging | `cluster-logging` | redhat-operators |
| Prometheus (User) | `prometheus` | community-operators |
| Vault (HashiCorp) | `vault-helm` | certified-operators |
| Strimzi (Kafka) | `strimzi-kafka-operator` | community-operators |
| PostgreSQL (Crunchy) | `crunchy-postgres-operator` | certified-operators |

## Create a Custom Resource (CR)

After an operator is installed, create instances via its CR.

```yaml
# Example: Kafka cluster via Strimzi
apiVersion: kafka.strimzi.io/v1beta2
kind: Kafka
metadata:
  name: my-cluster
  namespace: my-app
spec:
  kafka:
    version: 3.6.0
    replicas: 3
    listeners:
      - name: plain
        port: 9092
        type: internal
        tls: false
      - name: tls
        port: 9093
        type: internal
        tls: true
    storage:
      type: persistent-claim
      size: 100Gi
      deleteClaim: false
  zookeeper:
    replicas: 3
    storage:
      type: persistent-claim
      size: 10Gi
      deleteClaim: false
```

## Operator troubleshooting

```bash
# Check operator pod logs
oc logs deployment/my-operator-controller-manager -n openshift-operators -f

# Check CSV (ClusterServiceVersion) status
oc describe csv my-operator.v1.0.0 -n openshift-operators

# Check CRD was installed
oc get crd | grep myoperator

# Check subscription status
oc describe subscription my-operator -n openshift-operators

# Check operator install plan
oc get installplan -n openshift-operators
oc describe installplan <name> -n openshift-operators

# Operator CR status / conditions
oc describe <cr-kind> <cr-name> -n my-app

# Force operator reconciliation (delete and recreate CR)
oc delete <cr-kind> <cr-name> -n my-app
oc apply -f my-cr.yaml
```

## Uninstall an operator

```bash
# 1. Delete the Subscription
oc delete subscription my-operator -n openshift-operators

# 2. Delete the CSV
oc delete csv my-operator.v1.0.0 -n openshift-operators

# 3. Delete CRDs (removes all instances — careful in prod)
oc get crd | grep myoperator | awk '{print $1}' | xargs oc delete crd

# 4. Delete any remaining CRs
oc get <cr-kind> -A | awk '{print $1,$2}' | xargs -n2 oc delete <cr-kind> -n
```
