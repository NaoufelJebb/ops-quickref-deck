# SETUP

## Docker Compose — dev / single node

```yaml
version: "3.8"
services:
  vault:
    image: hashicorp/vault:1.16
    cap_add: [IPC_LOCK]
    ports:
      - "8200:8200"
    environment:
      VAULT_DEV_ROOT_TOKEN_ID: root        # dev mode only
      VAULT_DEV_LISTEN_ADDRESS: "0.0.0.0:8200"
    command: server -dev                   # dev mode — in-memory, pre-unsealed
```

```bash
export VAULT_ADDR=http://localhost:8200
export VAULT_TOKEN=root
vault status
```

## Production config — `vault.hcl`

```hcl
ui            = true
log_level     = "info"
api_addr      = "https://vault.example.com:8200"
cluster_addr  = "https://vault.example.com:8201"

listener "tcp" {
  address       = "0.0.0.0:8200"
  tls_cert_file = "/vault/tls/tls.crt"
  tls_key_file  = "/vault/tls/tls.key"
}

storage "raft" {                          # integrated Raft storage — recommended
  path    = "/vault/data"
  node_id = "vault-1"
}

# Auto-unseal via AWS KMS
seal "awskms" {
  region     = "eu-west-1"
  kms_key_id = "alias/vault-unseal"
}

# Or: Integrated storage without auto-unseal — use Shamir shares
```

## Initialize Vault (first time only)

```bash
# Initialize — generates unseal keys and root token
vault operator init \
  -key-shares=5 \
  -key-threshold=3 \
  -format=json > vault-init.json

# STORE vault-init.json SECURELY — contains unseal keys and root token
# Distribute unseal keys to separate people / systems

# Unseal (requires threshold number of keys)
vault operator unseal $(jq -r '.unseal_keys_b64[0]' vault-init.json)
vault operator unseal $(jq -r '.unseal_keys_b64[1]' vault-init.json)
vault operator unseal $(jq -r '.unseal_keys_b64[2]' vault-init.json)

# Log in with root token (only for initial setup — then create admin policy)
vault login $(jq -r '.root_token' vault-init.json)
```

## HA with Raft (recommended)

```hcl
# vault-2.hcl — join existing cluster
storage "raft" {
  path    = "/vault/data"
  node_id = "vault-2"

  retry_join {
    leader_api_addr = "https://vault-1.example.com:8200"
  }
}
```

```bash
# On vault-2, after starting:
vault operator raft join https://vault-1.example.com:8200

# Check cluster peers
vault operator raft list-peers
```

## Kubernetes — Vault Helm chart

```bash
helm repo add hashicorp https://helm.releases.hashicorp.com
helm repo update

helm install vault hashicorp/vault \
  --namespace vault --create-namespace \
  -f vault-values.yaml
```

```yaml
# vault-values.yaml
server:
  ha:
    enabled: true
    replicas: 3
    raft:
      enabled: true
      setNodeId: true
      config: |
        ui = true
        listener "tcp" {
          tls_disable = 1
          address = "[::]:8200"
        }
        storage "raft" {
          path = "/vault/data"
        }
        service_registration "kubernetes" {}

  affinity: |
    podAntiAffinity:
      requiredDuringSchedulingIgnoredDuringExecution:
        - topologyKey: kubernetes.io/hostname

injector:
  enabled: true       # Vault Agent sidecar injection

ui:
  enabled: true
  serviceType: ClusterIP
```

```bash
# Initialize the first pod
kubectl exec -n vault vault-0 -- vault operator init \
  -key-shares=5 -key-threshold=3 -format=json > vault-init.json

# Unseal vault-0
for i in 0 1 2; do
  kubectl exec -n vault vault-0 -- vault operator unseal \
    $(jq -r ".unseal_keys_b64[$i]" vault-init.json)
done

# Join and unseal other pods
kubectl exec -n vault vault-1 -- vault operator raft join http://vault-0.vault-internal:8200
kubectl exec -n vault vault-2 -- vault operator raft join http://vault-0.vault-internal:8200

for pod in vault-1 vault-2; do
  for i in 0 1 2; do
    kubectl exec -n vault $pod -- vault operator unseal \
      $(jq -r ".unseal_keys_b64[$i]" vault-init.json)
  done
done
```

## Environment variables

```bash
export VAULT_ADDR=https://vault.example.com:8200
export VAULT_TOKEN=<your-token>
export VAULT_NAMESPACE=admin          # Vault Enterprise namespaces
export VAULT_SKIP_VERIFY=true         # dev only — skip TLS verification
export VAULT_CACERT=/path/to/ca.crt   # custom CA
```
