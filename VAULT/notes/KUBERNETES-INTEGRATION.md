# KUBERNETES INTEGRATION

## Three ways to get secrets into pods

| Method | How | Best for |
|---|---|---|
| **Vault Agent Injector** | Sidecar writes secrets to shared volume | Existing apps — no code change |
| **Vault CSI Driver** | Mounts secrets as files via CSI | File-based config, no sidecar overhead |
| **External Secrets Operator** | Syncs secrets to native K8s Secrets | Teams that prefer K8s-native Secrets |

---

## Vault Agent Injector

The injector mutates pods — adds an init container (fetch secrets) and a sidecar (renew leases).

### Prerequisites

```bash
# Vault Agent Injector is included in the Helm chart
# Ensure the injector is enabled:
helm upgrade vault hashicorp/vault --set injector.enabled=true -n vault
```

### Annotate your pod

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: myapp
  namespace: my-namespace
spec:
  template:
    metadata:
      annotations:
        vault.hashicorp.com/agent-inject: "true"
        vault.hashicorp.com/role: "myapp"        # K8s auth role

        # Inject a KV secret as a file
        vault.hashicorp.com/agent-inject-secret-config: "secret/data/myapp/config"
        vault.hashicorp.com/agent-inject-template-config: |
          {{- with secret "secret/data/myapp/config" -}}
          DB_HOST={{ .Data.data.db_host }}
          DB_PASSWORD={{ .Data.data.db_password }}
          {{- end }}

        # Inject as JSON (no template)
        vault.hashicorp.com/agent-inject-secret-db: "database/creds/app-role"

        # TLS verification
        vault.hashicorp.com/tls-skip-verify: "false"
        vault.hashicorp.com/ca-cert: "/vault/tls/ca.crt"
    spec:
      serviceAccountName: myapp-sa    # must match K8s auth role
      containers:
        - name: myapp
          image: myapp:latest
          # Secret available at /vault/secrets/config
          # Read with: cat /vault/secrets/config
```

### Required K8s service account and Vault role

```bash
# K8s service account
kubectl create sa myapp-sa -n my-namespace

# Vault role binding the SA to a policy
vault write auth/kubernetes/role/myapp \
  bound_service_account_names=myapp-sa \
  bound_service_account_namespaces=my-namespace \
  token_policies=myapp-policy \
  token_ttl=1h
```

---

## Vault CSI Driver

```bash
# Install CSI driver
helm install vault-csi-provider hashicorp/vault-csi-provider \
  --namespace vault
```

```yaml
# SecretProviderClass — defines what to fetch
apiVersion: secrets-store.csi.x-k8s.io/v1
kind: SecretProviderClass
metadata:
  name: myapp-secrets
  namespace: my-namespace
spec:
  provider: vault
  parameters:
    vaultAddress: "https://vault.example.com:8200"
    roleName: "myapp"
    objects: |
      - objectName: "db_password"
        secretPath: "secret/data/myapp/config"
        secretKey: "db_password"
      - objectName: "api_key"
        secretPath: "secret/data/myapp/config"
        secretKey: "api_key"
  # Optionally sync to a K8s secret
  secretObjects:
    - secretName: myapp-k8s-secret
      type: Opaque
      data:
        - objectName: db_password
          key: DB_PASSWORD
```

```yaml
# Mount in a pod
spec:
  serviceAccountName: myapp-sa
  volumes:
    - name: secrets
      csi:
        driver: secrets-store.csi.k8s.io
        readOnly: true
        volumeAttributes:
          secretProviderClass: myapp-secrets
  containers:
    - name: myapp
      volumeMounts:
        - name: secrets
          mountPath: /mnt/secrets
          readOnly: true
      # Files available at /mnt/secrets/db_password, /mnt/secrets/api_key
```

---

## External Secrets Operator (ESO)

ESO syncs Vault secrets into native Kubernetes Secrets. Apps read K8s secrets — no Vault awareness needed.

```bash
helm repo add external-secrets https://charts.external-secrets.io
helm install external-secrets external-secrets/external-secrets \
  --namespace external-secrets --create-namespace
```

```yaml
# SecretStore — how to connect to Vault
apiVersion: external-secrets.io/v1beta1
kind: SecretStore
metadata:
  name: vault-backend
  namespace: my-namespace
spec:
  provider:
    vault:
      server: "https://vault.example.com:8200"
      path: "secret"
      version: "v2"
      auth:
        kubernetes:
          mountPath: "kubernetes"
          role: "myapp"
          serviceAccountRef:
            name: myapp-sa
---
# ExternalSecret — what to sync
apiVersion: external-secrets.io/v1beta1
kind: ExternalSecret
metadata:
  name: myapp-secret
  namespace: my-namespace
spec:
  refreshInterval: 1h
  secretStoreRef:
    name: vault-backend
    kind: SecretStore
  target:
    name: myapp-k8s-secret          # resulting K8s Secret name
    creationPolicy: Owner
  data:
    - secretKey: DB_PASSWORD
      remoteRef:
        key: myapp/config
        property: db_password
    - secretKey: API_KEY
      remoteRef:
        key: myapp/config
        property: api_key
```

The resulting `myapp-k8s-secret` Kubernetes Secret is refreshed every hour.

---

## Check Vault Agent injection status

```bash
# Verify injector is running
kubectl get pods -n vault | grep injector

# Check if a pod has the agent sidecar
kubectl describe pod <pod> -n my-namespace | grep vault-agent

# Vault Agent logs
kubectl logs <pod> -n my-namespace -c vault-agent-init
kubectl logs <pod> -n my-namespace -c vault-agent

# Check injected secret file
kubectl exec <pod> -n my-namespace -- cat /vault/secrets/config
```
