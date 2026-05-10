# INCIDENT

## Vault is sealed

```bash
# Confirm
vault status | grep Sealed    # Sealed: true

# Unseal — requires threshold number of key shares
vault operator unseal <key-1>
vault operator unseal <key-2>
vault operator unseal <key-3>

# If auto-unseal is configured but failing:
# Check KMS access (IAM role, network, key policy)
journalctl -u vault -f    # or kubectl logs -n vault vault-0 -f
```

---

## Secret leaked / compromised

```bash
# 1. Revoke the specific lease immediately
vault lease revoke <lease-id>

# 2. If lease ID unknown — revoke all leases for that engine path
vault lease revoke -prefix database/creds/app-role/

# 3. Rotate the underlying credential Vault uses
vault write -force database/rotate-root/mydb      # DB engine
vault write -f transit/keys/myapp-key/rotate      # Transit key

# 4. For KV secret — overwrite with new value
vault kv put secret/myapp/config api_key=<new-value>

# 5. For AppRole SecretID — generate a new one, invalidate old
vault write -f auth/approle/role/myapp/secret-id
# Revoke old SecretID if you have its accessor
vault write auth/approle/role/myapp/secret-id-accessor/destroy \
  secret_id_accessor=<accessor>

# 6. Check audit log for who/what accessed the secret
grep "secret/data/myapp/config" /vault/logs/audit.log | \
  jq '{time, client: .auth.display_name, op: .request.operation}'
```

---

## Token / auth breach

```bash
# Revoke a compromised token (and all its children)
vault token revoke <token>

# Revoke all tokens under an auth method path (e.g. AppRole breach)
vault token revoke -mode=path auth/approle/

# Revoke all tokens with a specific policy
# No direct command — revoke each token or rotate the AppRole credentials

# Emergency: revoke all tokens in the system (use root token only)
vault token revoke -mode=orphan self   # ← don't; use targeted revocation instead

# Rotate AppRole SecretID (generate new, revoke old accessor)
vault write -f auth/approle/role/myapp/secret-id
```

---

## Raft cluster node lost

```bash
# Check current peers
vault operator raft list-peers

# Remove a dead peer (from the leader node)
vault operator raft remove-peer <node-id>

# Bring up a replacement node, then rejoin
vault operator raft join https://vault-leader:8200
vault operator unseal ...
```

---

## Permission denied — app can't read its secrets

```bash
# 1. What policies does the token have?
vault token lookup <token>

# 2. Does the policy cover the path?
vault token capabilities secret/data/myapp/config

# 3. Read the policy and look for path issues
vault policy read myapp-policy

# 4. Common fixes:
# KV v2: policy path must be secret/data/... not secret/...
# Wildcard: secret/data/myapp/* does NOT match secret/data/myapp/config (no trailing *)
#   Correct: secret/data/myapp/config or secret/data/myapp/*
# Missing list: add path "secret/metadata/myapp/" { capabilities = ["list"] }
```

---

## Restore from snapshot

```bash
# Only use if the cluster data is unrecoverable — this overwrites everything
vault operator raft snapshot restore \
  -force vault-snapshot-20240101.snap

# After restore — unseal all nodes
vault operator unseal <key-1>
vault operator unseal <key-2>
vault operator unseal <key-3>
```
