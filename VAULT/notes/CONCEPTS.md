# CONCEPTS

## What Vault is

HashiCorp Vault is a secrets management platform. It centralizes storage, access control, and auditing of secrets — API keys, passwords, certificates, tokens, database credentials — and can generate them dynamically on demand.

```
App / CI-CD / Human
        │
        ▼  authenticate (token, AppRole, K8s SA, LDAP...)
   [ Vault ]
        │
        ├── Static secrets    (KV store)
        ├── Dynamic secrets   (DB creds, AWS keys, PKI certs — generated on request)
        ├── Encryption as a Service  (Transit engine)
        └── PKI / SSH signing
```

## Core concepts

| Concept | What it is |
|---|---|
| **Secret Engine** | A plugin that stores or generates secrets (KV, Database, PKI, AWS, Transit…) |
| **Auth Method** | How a client proves identity (Token, AppRole, Kubernetes, LDAP, OIDC…) |
| **Policy** | HCL rules that define what paths a token can read/write |
| **Token** | A credential issued after successful auth — carries policies |
| **Lease** | A time-to-live on a secret or token — Vault revokes automatically on expiry |
| **Seal / Unseal** | Vault starts sealed (encrypted). Unseal keys (or auto-unseal) decrypt the storage |
| **Path** | Every Vault operation is a read/write/delete on a path: `secret/data/myapp/config` |

## Secret engines

| Engine | Use for |
|---|---|
| `kv` v2 | Versioned static secrets — most common |
| `database` | Dynamic DB credentials (Postgres, MySQL, MSSQL, MongoDB…) |
| `pki` | Certificate authority — issue TLS certs on demand |
| `aws` | Dynamic AWS IAM credentials |
| `azure` | Dynamic Azure service principals |
| `transit` | Encrypt/decrypt data without storing it |
| `ssh` | Sign SSH keys — short-lived SSH access |
| `totp` | TOTP code generation |

## Auth methods

| Method | Use for |
|---|---|
| `token` | Direct token — default, root token at init |
| `approle` | Machine-to-machine — CI/CD, services |
| `kubernetes` | Pods authenticate with their K8s service account JWT |
| `ldap` | Users authenticate with AD/LDAP credentials |
| `oidc` | SSO via Keycloak, Okta, GitHub… |
| `aws` | EC2 instances or IAM roles authenticate automatically |
| `gcp` | GCP service accounts |
| `cert` | mTLS client certificate |

## Policy syntax

```hcl
# Allow read on a specific path
path "secret/data/myapp/*" {
  capabilities = ["read", "list"]
}

# Allow all operations on a path
path "secret/data/myapp/dev/*" {
  capabilities = ["create", "read", "update", "delete", "list"]
}

# Allow token renewal
path "auth/token/renew-self" {
  capabilities = ["update"]
}

# Deny overrides everything
path "secret/data/myapp/prod/keys" {
  capabilities = ["deny"]
}
```

Capabilities: `create` `read` `update` `delete` `list` `sudo` `deny`

## Seal / Unseal

```bash
# Check seal status
vault status

# Unseal (Shamir key shares — requires threshold number of keys)
vault operator unseal <unseal-key-1>
vault operator unseal <unseal-key-2>
vault operator unseal <unseal-key-3>

# Auto-unseal delegates to a KMS (AWS KMS, GCP CKMS, Azure Key Vault, HSM)
# Configured in vault.hcl — no manual unseal needed on restart
```
