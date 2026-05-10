# OPS

## Daily health checks

```bash
# Health endpoint
curl http://keycloak:8080/health/ready | jq .

# Active sessions in realm
curl "$KC/admin/realms/$REALM/sessions/stats" \
  -H "Authorization: Bearer $ADMIN_TOKEN" | jq .

# Count active users
curl "$KC/admin/realms/$REALM/users/count" \
  -H "Authorization: Bearer $ADMIN_TOKEN"

# Check for failed logins (brute force)
curl "$KC/admin/realms/$REALM/attack-detection/brute-force/users" \
  -H "Authorization: Bearer $ADMIN_TOKEN" | jq 'length'
```

---

## Realm export / backup

```bash
# Export via Admin API (partial — clients + groups + roles, not credentials)
curl -X POST "$KC/admin/realms/$REALM/partial-export?exportClients=true&exportGroupsAndRoles=true" \
  -H "Authorization: Bearer $ADMIN_TOKEN" \
  -o realm-export-$(date +%Y%m%d).json

# Full export via CLI (includes hashed passwords — do offline)
# Stop or use a separate Keycloak instance pointing to same DB
/opt/keycloak/bin/kc.sh export \
  --dir /opt/keycloak/data/export \
  --realm myrealm \
  --users realm_file
```

## Realm import

```bash
# Import via Admin API (realm must not exist)
curl -X POST "$KC/admin/realms" \
  -H "Authorization: Bearer $ADMIN_TOKEN" \
  -H "Content-Type: application/json" \
  -d @realm-export.json

# Import via CLI (for full import with users)
/opt/keycloak/bin/kc.sh import \
  --dir /opt/keycloak/data/import \
  --override true
```

---

## Cert / key rotation

```bash
# Current active keys
curl "$KC/admin/realms/$REALM/keys" \
  -H "Authorization: Bearer $ADMIN_TOKEN" | jq '.active'

# Add a new RSA key provider (new key becomes active)
curl -X POST "$KC/admin/realms/$REALM/components" \
  -H "Authorization: Bearer $ADMIN_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
    "name": "rsa-key-2025",
    "providerId": "rsa-generated",
    "providerType": "org.keycloak.keys.KeyProvider",
    "parentId": "'$REALM'",
    "config": {
      "priority": ["200"],
      "enabled": ["true"],
      "active": ["true"],
      "algorithm": ["RS256"],
      "keySize": ["2048"]
    }
  }'

# Demote old key (set active: false, keep enabled for validation of existing tokens)
curl -X PUT "$KC/admin/realms/$REALM/components/<old-key-id>" \
  -H "Authorization: Bearer $ADMIN_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{"config": {"active": ["false"], "enabled": ["true"]}}'

# After all existing tokens expire, disable old key entirely
```

---

## Force logout all users (emergency)

```bash
# Logout all sessions in realm
curl -X POST "$KC/admin/realms/$REALM/logout-all" \
  -H "Authorization: Bearer $ADMIN_TOKEN"

# Logout one specific user
curl -X DELETE "$KC/admin/realms/$REALM/users/$USER_ID/sessions" \
  -H "Authorization: Bearer $ADMIN_TOKEN"
```

---

## Monitoring — Prometheus metrics

Enable: `metrics-enabled=true` in `keycloak.conf`

Scrape: `GET http://keycloak:9000/metrics` (or port 8080)

Key metrics:
```
keycloak_logins_total{realm, provider, client_id}          successful logins
keycloak_failed_login_attempts_total{realm, provider}       failed logins
keycloak_registrations_total{realm}                         new registrations
keycloak_request_duration_seconds{realm, resource}          request latency
jvm_memory_used_bytes                                       JVM heap usage
jvm_threads_live_threads                                    JVM threads
process_cpu_seconds_total                                   CPU
```

Alert suggestions:
- Failed login rate spike → brute force attempt
- JVM memory > 85% → risk of OOM
- `keycloak_request_duration_seconds` P99 > 2s → performance degradation
- Health endpoint returning non-200 → availability alert

---

## User management ops

```bash
# Bulk disable users matching a pattern
curl "$KC/admin/realms/$REALM/users?search=contractor-&max=200" \
  -H "Authorization: Bearer $ADMIN_TOKEN" | \
  jq -r '.[].id' | while read id; do
    curl -X PUT "$KC/admin/realms/$REALM/users/$id" \
      -H "Authorization: Bearer $ADMIN_TOKEN" \
      -H "Content-Type: application/json" \
      -d '{"enabled": false}'
  done

# Force password reset for a user
curl -X PUT "$KC/admin/realms/$REALM/users/$USER_ID/execute-actions-email" \
  -H "Authorization: Bearer $ADMIN_TOKEN" \
  -H "Content-Type: application/json" \
  -d '["UPDATE_PASSWORD"]'

# Trigger LDAP sync
curl -X POST "$KC/admin/realms/$REALM/user-storage/$PROVIDER_ID/sync?action=triggerFullSync" \
  -H "Authorization: Bearer $ADMIN_TOKEN"
```

---

## Ops checklist

- [ ] Health endpoint monitored — alert on non-ready state
- [ ] Realm exported on schedule (daily/weekly) and stored offsite
- [ ] Event logging enabled — login events + admin events
- [ ] Brute force protection enabled in all realms
- [ ] Token lifetimes reviewed — access token ≤ 5 min
- [ ] Admin password rotated from initial setup value
- [ ] Admin `master` realm access restricted to ops accounts only
- [ ] LDAP sync monitored — alert if sync fails
- [ ] Signing keys rotated annually (or on compromise)
- [ ] Unused clients disabled or deleted
- [ ] Service account tokens scoped to minimum required roles
