# INCIDENT

## Users can't log in

```bash
# 1. Is Keycloak healthy?
curl http://keycloak:8080/health/ready

# 2. Check recent login errors
# Realm → Events → Login events → filter by error

# 3. Check logs
docker logs keycloak 2>&1 | grep -E "ERROR|WARN" | tail -30
kubectl logs -n keycloak statefulset/keycloak --tail=30 | grep -E "ERROR|WARN"

# 4. Is DB reachable?
# Look for: "Unable to connect to database" in logs

# 5. Test the token endpoint directly
curl -X POST "$KC/realms/$REALM/protocol/openid-connect/token" \
  -d "grant_type=client_credentials&client_id=admin-cli&client_secret=xxx"
```

---

## Suspected account compromise

```bash
# 1. Force logout all sessions for the user
curl -X DELETE "$KC/admin/realms/$REALM/users/$USER_ID/sessions" \
  -H "Authorization: Bearer $ADMIN_TOKEN"

# 2. Disable the account
curl -X PUT "$KC/admin/realms/$REALM/users/$USER_ID" \
  -H "Authorization: Bearer $ADMIN_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{"enabled": false}'

# 3. Force password reset (re-enable + send email)
curl -X PUT "$KC/admin/realms/$REALM/users/$USER_ID/execute-actions-email" \
  -H "Authorization: Bearer $ADMIN_TOKEN" \
  -H "Content-Type: application/json" \
  -d '["UPDATE_PASSWORD", "CONFIGURE_TOTP"]'
```

---

## Brute force attack on login

```bash
# Check locked accounts count
curl "$KC/admin/realms/$REALM/attack-detection/brute-force/users" \
  -H "Authorization: Bearer $ADMIN_TOKEN" | jq 'length'

# Verify brute force protection is enabled
curl "$KC/admin/realms/$REALM" \
  -H "Authorization: Bearer $ADMIN_TOKEN" | jq '{bruteForceProtected, failureFactor}'

# Enable if off (emergency)
curl -X PUT "$KC/admin/realms/$REALM" \
  -H "Authorization: Bearer $ADMIN_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{"bruteForceProtected": true, "failureFactor": 5, "waitIncrementSeconds": 30}'

# Unlock a legitimate user caught in lockout
curl -X DELETE "$KC/admin/realms/$REALM/attack-detection/brute-force/users/$USER_ID" \
  -H "Authorization: Bearer $ADMIN_TOKEN"
```

---

## Signing key compromised

```bash
# 1. Add new key with high priority (becomes active immediately)
#    See OPS.md → Cert/key rotation for full curl command

# 2. Disable compromised key (old tokens immediately invalid)
curl -X PUT "$KC/admin/realms/$REALM/components/<compromised-key-id>" \
  -H "Authorization: Bearer $ADMIN_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{"config": {"active": ["false"], "enabled": ["false"]}}'

# 3. All existing tokens are now invalid — users must re-authenticate
# 4. Force logout all realm sessions
curl -X POST "$KC/admin/realms/$REALM/logout-all" \
  -H "Authorization: Bearer $ADMIN_TOKEN"
```

---

## Service account token abuse

```bash
# 1. Rotate the client secret immediately
# Clients → [client] → Credentials → Regenerate secret

# 2. Via API
curl -X POST "$KC/admin/realms/$REALM/clients/$CLIENT_ID/client-secret" \
  -H "Authorization: Bearer $ADMIN_TOKEN"

# 3. All existing access tokens for this client expire at their normal TTL
#    Refresh tokens are immediately invalid (new secret required)

# 4. Audit what the service account accessed — check admin events
curl "$KC/admin/realms/$REALM/admin-events?client=$CLIENT_ID&max=100" \
  -H "Authorization: Bearer $ADMIN_TOKEN" | jq .
```

---

## Restore

- Re-enable disabled users after investigation
- Re-enable brute force protection if temporarily disabled
- Update downstream service configurations with new client secrets
- Document in your incident log: what happened, what was done, timeline
