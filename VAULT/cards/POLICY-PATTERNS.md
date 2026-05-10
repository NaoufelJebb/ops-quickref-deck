# POLICY PATTERNS

## Read KV v2 secrets (app)

```hcl
# Read secrets at a specific path — most common app policy
path "secret/data/myapp/*" {
  capabilities = ["read", "list"]
}
path "secret/metadata/myapp/*" {
  capabilities = ["list"]
}
path "auth/token/renew-self" {
  capabilities = ["update"]
}
path "auth/token/lookup-self" {
  capabilities = ["read"]
}
```

## Read/write KV v2 (developer / CI)

```hcl
path "secret/data/myteam/*" {
  capabilities = ["create", "read", "update", "delete", "list"]
}
path "secret/metadata/myteam/*" {
  capabilities = ["list", "delete"]
}
path "secret/delete/myteam/*" {
  capabilities = ["update"]
}
path "secret/destroy/myteam/*" {
  capabilities = ["update"]
}
path "secret/undelete/myteam/*" {
  capabilities = ["update"]
}
```

## Dynamic DB credentials

```hcl
path "database/creds/app-role" {
  capabilities = ["read"]
}
path "sys/leases/renew" {
  capabilities = ["update"]
}
path "sys/leases/revoke" {
  capabilities = ["update"]
}
```

## PKI — issue certificates

```hcl
path "pki_int/issue/myapp" {
  capabilities = ["create", "update"]
}
path "pki_int/sign/myapp" {
  capabilities = ["create", "update"]
}
path "pki_int/cert/ca" {
  capabilities = ["read"]
}
```

## Transit — encrypt/decrypt only

```hcl
path "transit/encrypt/myapp-key" {
  capabilities = ["update"]
}
path "transit/decrypt/myapp-key" {
  capabilities = ["update"]
}
# Read the key metadata only — not the key itself
path "transit/keys/myapp-key" {
  capabilities = ["read"]
}
```

## Admin — manage secrets engines and auth

```hcl
# Full access to sys — use sparingly
path "sys/*" {
  capabilities = ["create", "read", "update", "delete", "list", "sudo"]
}
path "auth/*" {
  capabilities = ["create", "read", "update", "delete", "list", "sudo"]
}
path "secret/*" {
  capabilities = ["create", "read", "update", "delete", "list"]
}
```

## CI/CD — read secrets + issue PKI certs

```hcl
path "secret/data/myapp/ci/*" {
  capabilities = ["read"]
}
path "pki_int/issue/ci-certs" {
  capabilities = ["create", "update"]
}
path "auth/token/renew-self" {
  capabilities = ["update"]
}
```

## Deny sensitive paths (override)

```hcl
# Add this to any policy to block access regardless of other rules
path "secret/data/myapp/prod/master-keys" {
  capabilities = ["deny"]
}
```

## Path templating — per-user paths

```hcl
# Each user can only read their own secrets
path "secret/data/users/{{identity.entity.name}}/*" {
  capabilities = ["create", "read", "update", "delete", "list"]
}
```

## Tips

- KV v2 paths must include `/data/` for CRUD and `/metadata/` for list — easy to miss
- `list` on `/metadata/` path is required to browse in the UI
- `deny` always wins — useful for exceptions in a broad policy
- Test policies with `vault token capabilities <path>` before deploying
