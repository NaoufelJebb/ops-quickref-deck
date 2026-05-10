# SECURITY

## Security Context Constraints (SCCs)

SCCs are the primary pod security mechanism in OpenShift. Every pod is admitted against an SCC.

```bash
# List SCCs ordered by priority
oc get scc

# Check what SCC a running pod uses
oc get pod <pod> -n <ns> -o jsonpath='{.metadata.annotations.openshift\.io/scc}'

# Simulate SCC admission for a pod spec
oc adm policy scc-review -f pod.yaml

# Find which SCC allows a service account
oc adm policy review -z <sa> -n <ns>

# Grant SCC to a service account (least privilege — try nonroot before anyuid)
oc adm policy add-scc-to-user nonroot      -z my-sa -n my-app
oc adm policy add-scc-to-user anyuid       -z my-sa -n my-app   # only if needed
oc adm policy add-scc-to-user privileged   -z my-sa -n my-app   # avoid unless infra

# Remove SCC
oc adm policy remove-scc-from-user anyuid -z my-sa -n my-app
```

### SCC decision order

OpenShift tries SCCs in priority order. The first one the pod satisfies wins. If none match → pod rejected.

### Write a custom SCC (least privilege)

```yaml
apiVersion: security.openshift.io/v1
kind: SecurityContextConstraints
metadata:
  name: my-restricted-scc
allowPrivilegedContainer: false
allowPrivilegeEscalation: false
allowHostDirVolumePlugin: false
allowHostIPC: false
allowHostNetwork: false
allowHostPID: false
allowHostPorts: false
readOnlyRootFilesystem: true
runAsUser:
  type: MustRunAsNonRoot
seLinuxContext:
  type: MustRunAs
fsGroup:
  type: MustRunAs
  ranges:
    - min: 1000
      max: 65535
supplementalGroups:
  type: RunAsAny
volumes:
  - configMap
  - emptyDir
  - persistentVolumeClaim
  - secret
users: []
groups: []
```

---

## Image security

```bash
# View image vulnerabilities (if image scanning is configured)
oc get imagestreamtags -n my-app

# Restrict which registries images can be pulled from
oc edit image.config.openshift.io cluster
# spec:
#   registrySources:
#     allowedRegistries:
#       - registry.redhat.io
#       - quay.io
#       - image-registry.openshift-image-registry.svc:5000
#     insecureRegistries: []

# Block images that haven't been signed
oc edit image.config.openshift.io cluster
# spec:
#   additionalTrustedCA: ...
```

---

## Secrets management

```bash
# Never store plaintext secrets in Git — use Sealed Secrets, Vault, or External Secrets Operator

# Sealed Secrets (encrypt secrets for Git storage)
kubeseal --controller-name=sealed-secrets \
  --controller-namespace=sealed-secrets \
  < secret.yaml > sealed-secret.yaml

# HashiCorp Vault integration (via Vault Agent injector)
# Annotate pod:
metadata:
  annotations:
    vault.hashicorp.com/agent-inject: "true"
    vault.hashicorp.com/role: "my-app"
    vault.hashicorp.com/agent-inject-secret-config: "secret/my-app/config"
```

### Encrypt secrets at rest (etcd encryption)

```bash
# Enable encryption
oc edit apiserver cluster
# spec:
#   encryption:
#     type: aescbc     # or: aesgcm

# Check encryption status
oc get apiserver cluster -o json | jq '.spec.encryption'
oc get secret -n openshift-config-managed encryption-config -o yaml
```

---

## Audit logging

```bash
# View API server audit log profile
oc get apiserver cluster -o yaml | grep -A5 audit

# Set audit policy
oc patch apiserver cluster --type=merge \
  -p '{"spec":{"audit":{"profile":"WriteRequestBodies"}}}'
# Profiles: Default | WriteRequestBodies | AllRequestBodies | None

# Access audit logs (on master nodes)
oc adm node-logs --role=master --path=openshift-apiserver/audit.log | \
  jq 'select(.verb == "delete")' | head -50
```

---

## Network security

```bash
# Verify NetworkPolicies in place
oc get networkpolicy -n my-app

# Check if a pod can reach another
oc rsh <pod-a> -n my-app -- curl http://<pod-b-ip>:8080

# Verify EgressNetworkPolicy
oc get egressnetworkpolicy -n my-app

# Check OVN-K network policy enforcement
oc get netnamespace
```

---

## Security checklist

- [ ] `kubeadmin` secret deleted after IdP configured
- [ ] No workloads running in `default` namespace
- [ ] SCCs reviewed — no unnecessary `privileged` or `anyuid` grants
- [ ] `cluster-admin` assignments minimal and documented
- [ ] Secrets encrypted at rest (etcd encryption enabled)
- [ ] Image pull restricted to trusted registries
- [ ] NetworkPolicy applied — default deny + explicit allow
- [ ] EgressNetworkPolicy applied — restrict outbound
- [ ] Audit logging enabled (WriteRequestBodies or higher)
- [ ] Resource quotas applied to all project namespaces
- [ ] LimitRange applied — no pods without resource requests
- [ ] Secrets not stored in Git (use Vault / Sealed Secrets)
- [ ] RBAC reviewed — no wildcard verbs in production roles
