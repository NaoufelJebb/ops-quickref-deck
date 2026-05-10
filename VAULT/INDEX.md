# VAULT NOTES — INDEX

## Notes (deep reference)

| File | Open when you need to... |
|---|---|
| [CONCEPTS.md](./notes/CONCEPTS.md) | Understand secret engines, auth methods, policies, tokens, seal/unseal |
| [SETUP.md](./notes/SETUP.md) | Install Vault, init, HA Raft config, Kubernetes Helm install |
| [SECRETS.md](./notes/SECRETS.md) | KV v2, database dynamic creds, PKI, Transit encryption, AWS engine |
| [AUTH-AND-POLICIES.md](./notes/AUTH-AND-POLICIES.md) | AppRole, Kubernetes, LDAP, OIDC, token management, policy syntax |
| [KUBERNETES-INTEGRATION.md](./notes/KUBERNETES-INTEGRATION.md) | Vault Agent Injector, CSI driver, External Secrets Operator |
| [OPS-AND-TROUBLESHOOTING.md](./notes/OPS-AND-TROUBLESHOOTING.md) | Audit logs, snapshots, lease management, debug, ops checklist |

## Cards (fast access)

| File | Open when you need to... |
|---|---|
| [CHEATSHEET.md](./cards/CHEATSHEET.md) | Any CLI command — KV, tokens, AppRole, leases, API |
| [POLICY-PATTERNS.md](./cards/POLICY-PATTERNS.md) | Copy-paste HCL policy templates for any use case |
| [INCIDENT.md](./cards/INCIDENT.md) | Sealed vault, leaked secret, auth breach, cluster recovery |

---

## Situation → file map

| I'm doing... | Open... |
|---|---|
| First day, learning Vault | [CONCEPTS.md](./notes/CONCEPTS.md) → [SETUP.md](./notes/SETUP.md) |
| Storing / reading a secret | [CHEATSHEET.md](./cards/CHEATSHEET.md) → KV section |
| Setting up dynamic DB creds | [SECRETS.md](./notes/SECRETS.md) → Database engine |
| Issuing TLS certificates | [SECRETS.md](./notes/SECRETS.md) → PKI engine |
| Encrypting data (no storage) | [SECRETS.md](./notes/SECRETS.md) → Transit engine |
| Setting up AppRole for a service | [AUTH-AND-POLICIES.md](./notes/AUTH-AND-POLICIES.md) + [CHEATSHEET.md](./cards/CHEATSHEET.md) |
| Connecting K8s pods to Vault | [KUBERNETES-INTEGRATION.md](./notes/KUBERNETES-INTEGRATION.md) |
| Configuring LDAP / AD login | [AUTH-AND-POLICIES.md](./notes/AUTH-AND-POLICIES.md) → LDAP section |
| Writing a policy | [POLICY-PATTERNS.md](./cards/POLICY-PATTERNS.md) |
| Vault is sealed | [INCIDENT.md](./cards/INCIDENT.md) |
| Secret was leaked | [INCIDENT.md](./cards/INCIDENT.md) |
| Permission denied error | [INCIDENT.md](./cards/INCIDENT.md) + [POLICY-PATTERNS.md](./cards/POLICY-PATTERNS.md) |
| Setting up audit / snapshots | [OPS-AND-TROUBLESHOOTING.md](./notes/OPS-AND-TROUBLESHOOTING.md) |
| Any CLI command | [CHEATSHEET.md](./cards/CHEATSHEET.md) |
