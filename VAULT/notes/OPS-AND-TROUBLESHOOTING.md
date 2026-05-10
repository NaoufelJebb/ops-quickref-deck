# OPS AND TROUBLESHOOTING

## Daily health checks

```bash
# Seal status and HA leader
vault status

# HA cluster peers
vault operator raft list-peers

# List enabled secret engines
vault secrets list

# List enabled auth methods
vault auth list

# Active leases (count by prefix)
vault list sys/leases/lookup/database/creds/app-role/

# Check audit devices
vault audit list
```

---

## Audit logging

Always enable audit logging in production — required for compliance.

```bash
# Enable file audit log
vault audit enable file file_path=/vault/logs/audit.log

# Enable syslog
vault audit enable syslog

# Verify audit is enabled
vault audit list

# Tail the audit log (JSON, one entry per line)
tail -f /vault/logs/audit.log | jq '{time, type, auth: .auth.display_name, path: .request.path, op: .request.operation}'
```

Vault will **block all requests** if no audit device is reachable. Always have two audit devices configured.

---

## Snapshots (Raft backup)

```bash
# Save a snapshot
vault operator raft snapshot save vault-snapshot-$(date +%Y%m%d).snap

# Verify snapshot
vault operator raft snapshot inspect vault-snapshot-$(date +%Y%m%d).snap

# Restore (requires a sealed, initialized cluster)
vault operator raft snapshot restore vault-snapshot.snap

# Automate with a cron job or CI/CD
```

---

## Lease management

```bash
# List leases for a path
vault list sys/leases/lookup/database/creds/app-role/

# Look up a specific lease
vault lease lookup <lease-id>

# Renew a lease
vault lease renew <lease-id>
vault lease renew -increment=2h <lease-id>

# Revoke a specific lease
vault lease revoke <lease-id>

# Revoke all leases under a prefix (emergency — e.g. DB creds)
vault lease revoke -prefix database/creds/app-role/

# Renew your own token
vault token renew
```

---

## Secret engine and auth method management

```bash
# Enable a secret engine at a custom path
vault secrets enable -path=myteam/kv kv-v2

# Disable (and delete all secrets — careful)
vault secrets disable myteam/kv

# Move a secrets engine to a new path
vault secrets move myteam/kv newteam/kv

# Tune engine settings (default TTL, max TTL)
vault secrets tune -default-lease-ttl=1h -max-lease-ttl=24h secret/

# Enable auth method
vault auth enable -path=myapprole approle

# Disable auth method
vault auth disable myapprole

# List all auth methods with details
vault auth list -detailed
```

---

## Root token management

```bash
# Generate a new root token (requires unseal key quorum)
vault operator generate-root -init
# Follow the prompts — each key holder provides their share

# Revoke the root token after use (don't keep it active)
vault token revoke <root-token>

# Check if root token is in use
vault token lookup <root-token>
```

---

## Troubleshooting

### Permission denied

```bash
# Check token policies
vault token lookup

# Check token capabilities on the exact path
vault token capabilities secret/data/myapp/config

# Check what policy is required
vault policy read myapp-policy

# Common causes:
# - Path mismatch: policy says secret/data/* but request is to secret/myapp/* (KV v2 uses /data/ prefix)
# - Missing list capability — can read but not discover
# - Namespace mismatch (Enterprise)
```

### KV v2 path confusion

```bash
# KV v2 paths always include /data/ for read/write and /metadata/ for list
# Policy:    path "secret/data/myapp/*"  ← correct for KV v2
# NOT:       path "secret/myapp/*"       ← KV v1 style

# CLI abstracts this:
vault kv get secret/myapp/config              # CLI adds /data/ automatically

# API requires explicit path:
GET /v1/secret/data/myapp/config              # correct
GET /v1/secret/myapp/config                   # wrong — 404
```

### Auth failures

```bash
# AppRole: "invalid role id"
vault read auth/approle/role/myapp/role-id    # verify role exists

# Kubernetes: "service account not authorized"
# Check the bound_service_account_names and bound_service_account_namespaces
vault read auth/kubernetes/role/myapp

# LDAP: "invalid credentials"
# Test LDAP bind separately
ldapsearch -H ldaps://dc1.corp.local:636 \
  -D "CN=svc-vault,OU=Service Accounts,DC=corp,DC=local" \
  -w "$BIND_PASS" -b "OU=Users,DC=corp,DC=local" "(sAMAccountName=testuser)"

# LDAP: groups not resolving to policies
vault read auth/ldap/groups/vault-admins      # check policy mapping exists
```

### Vault sealed after restart

```bash
# Check status
vault status

# Unseal manually (Shamir)
vault operator unseal <key-1>
vault operator unseal <key-2>
vault operator unseal <key-3>

# If auto-unseal is configured but failing:
# Check KMS connectivity / IAM permissions
# Check vault logs: journalctl -u vault -f
```

### Raft cluster issues

```bash
# Check cluster health
vault operator raft list-peers

# A node shows "voter: false" or is missing — rejoin it
vault operator raft join https://vault-leader:8200

# Remove a dead peer
vault operator raft remove-peer <node-id>

# Check Raft autopilot status
vault operator raft autopilot state
```

### Vault logs

```bash
# Systemd
journalctl -u vault -f

# Docker
docker logs vault -f

# Kubernetes
kubectl logs -n vault vault-0 -f
kubectl logs -n vault vault-0 -c vault-agent-init  # agent init

# Increase log level temporarily
vault audit enable file log_raw=true file_path=/vault/logs/debug.log
# CAUTION: log_raw=true logs plaintext secrets — dev only
```

---

## Ops checklist

- [ ] Audit log enabled (2 devices — file + syslog)
- [ ] Auto-unseal configured — Vault recovers on restart without manual intervention
- [ ] Raft snapshot scheduled daily and stored offsite
- [ ] Root token revoked — not stored or used for daily operations
- [ ] Admin policy created — separate from root token
- [ ] Token TTLs set — no infinite tokens in production
- [ ] Lease expiry monitored — alert on lease count spikes
- [ ] Vault HA — 3 nodes minimum, Raft quorum healthy
- [ ] `vault status` in monitoring — alert if sealed
- [ ] Secret engine paths documented — team knows where to find what
