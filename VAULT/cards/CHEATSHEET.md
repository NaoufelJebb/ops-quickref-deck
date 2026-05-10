# CHEATSHEET

## Setup

```bash
export VAULT_ADDR=https://vault.example.com:8200
export VAULT_TOKEN=<token>
vault status
vault login                           # interactive login
vault login -method=ldap username=me
vault login -method=oidc
```

## KV v2

```bash
vault kv put secret/myapp/config key=value key2=value2
vault kv get secret/myapp/config
vault kv get -field=key secret/myapp/config
vault kv get -format=json secret/myapp/config | jq '.data.data'
vault kv patch secret/myapp/config key=newvalue
vault kv list secret/myapp
vault kv metadata get secret/myapp/config
vault kv get -version=2 secret/myapp/config
vault kv delete secret/myapp/config
vault kv metadata delete secret/myapp/config    # permanent
```

## Tokens

```bash
vault token create -policy=mypolicy -ttl=24h -renewable=true
vault token lookup
vault token lookup <token>
vault token renew
vault token renew -increment=12h
vault token revoke <token>
vault token capabilities secret/data/myapp/config
```

## AppRole

```bash
vault auth enable approle
vault write auth/approle/role/myapp token_policies=mypolicy token_ttl=1h secret_id_ttl=24h
vault read auth/approle/role/myapp/role-id          # get RoleID
vault write -f auth/approle/role/myapp/secret-id    # generate SecretID
vault write auth/approle/login role_id=<id> secret_id=<sid>
```

## Policies

```bash
vault policy list
vault policy read mypolicy
vault policy write mypolicy policy.hcl
vault policy delete mypolicy
```

## Leases

```bash
vault list sys/leases/lookup/database/creds/app-role/
vault lease lookup <lease-id>
vault lease renew <lease-id>
vault lease revoke <lease-id>
vault lease revoke -prefix database/creds/app-role/   # revoke all (emergency)
```

## Secrets engines

```bash
vault secrets list
vault secrets enable -path=secret kv-v2
vault secrets enable database
vault secrets enable pki
vault secrets enable transit
vault secrets disable mypath/        # ⚠️ deletes all secrets
vault secrets tune -default-lease-ttl=1h secret/
```

## Auth methods

```bash
vault auth list
vault auth enable approle
vault auth enable kubernetes
vault auth enable ldap
vault auth enable oidc
vault auth disable mymethod          # ⚠️ revokes all tokens
```

## Ops

```bash
vault status
vault operator raft list-peers
vault operator raft snapshot save snapshot.snap
vault operator raft snapshot restore snapshot.snap
vault audit list
vault audit enable file file_path=/vault/logs/audit.log
vault operator unseal <key>
vault operator generate-root -init
```

## API (curl)

```bash
# Read KV v2
curl "$VAULT_ADDR/v1/secret/data/myapp/config" \
  -H "X-Vault-Token: $VAULT_TOKEN" | jq '.data.data'

# Write KV v2
curl -X POST "$VAULT_ADDR/v1/secret/data/myapp/config" \
  -H "X-Vault-Token: $VAULT_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{"data":{"key":"value"}}'

# AppRole login
curl -X POST "$VAULT_ADDR/v1/auth/approle/login" \
  -d '{"role_id":"<id>","secret_id":"<sid>"}' | jq '.auth.client_token'

# K8s login
curl -X POST "$VAULT_ADDR/v1/auth/kubernetes/login" \
  -d "{\"role\":\"myapp\",\"jwt\":\"$(cat /var/run/secrets/kubernetes.io/serviceaccount/token)\"}" \
  | jq '.auth.client_token'
```
