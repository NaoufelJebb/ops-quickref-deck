# TROUBLESHOOTING

## Login failures

### Invalid client / redirect_uri mismatch

```
Error: Invalid redirect_uri
```

```bash
# Check registered redirect URIs for the client
curl "$KC/admin/realms/$REALM/clients?clientId=my-client" \
  -H "Authorization: Bearer $ADMIN_TOKEN" | jq '.[0].redirectUris'
```

Fix: add the exact redirect URI (including trailing slash if used) to `Valid redirect URIs`.

---

### Token validation fails (API returns 401)

Decode the token first:
```bash
echo "<payload-part>" | base64 -d | jq .
```

Check in order:
1. `exp` — is the token expired?
2. `iss` — does it match `https://keycloak/realms/{realm}`? Exact match including trailing slash
3. `aud` — does it contain your client/API identifier?
4. Signature — fetch JWKS and verify: `GET /realms/{realm}/protocol/openid-connect/certs`

---

### User can log in but gets wrong roles

```bash
# Check what roles the user actually has
curl "$KC/admin/realms/$REALM/users/$USER_ID/role-mappings" \
  -H "Authorization: Bearer $ADMIN_TOKEN" | jq .

# Use the token evaluator in the console
# Realm → Clients → [client] → Client scopes → Evaluate
# Enter the username, generate tokens, inspect claims
```

Common causes:
- Role assigned to a group the user isn't in
- Client scope doesn't include the `roles` mapper
- `roles` scope not in the client's default scopes
- LDAP group mapper not synced — run manual sync

---

### LDAP login fails

```
Error: Invalid credentials
```

```bash
# Test the LDAP bind from inside the Keycloak container
ldapsearch -H ldap://dc1.corp.local:389 \
  -D "CN=svc-keycloak,OU=Service Accounts,DC=corp,DC=local" \
  -w "$BIND_PASSWORD" \
  -b "OU=Users,DC=corp,DC=local" \
  "(sAMAccountName=testuser)"

# Enable LDAP debug logging temporarily
# Set env var: KC_LOG_LEVEL=org.keycloak.storage.ldap:debug
```

Check:
- Bind DN format — AD uses `CN=name,OU=...` not `uid=name,dc=...`
- Firewall: Keycloak → LDAP/AD port 389 or 636
- Service account not expired or locked in AD
- Users DN exists and contains the test user

---

### MFA not triggering

- Check that the authentication flow is bound: `Authentication → Bindings → Browser flow`
- Check that the Conditional OTP is set to `Required` not `Alternative`
- Check the OTP conditional policy: `Authentication → Policies → OTP policy`
- If using a custom flow, verify it's bound to the realm, not just saved

---

### Session expires too quickly

Check token lifetimes:
`Realm settings → Tokens tab`

| Setting to check | Location |
|---|---|
| Access token lifespan | Realm settings → Tokens |
| SSO Session idle | Realm settings → Tokens |
| SSO Session max | Realm settings → Tokens |
| Per-client override | Clients → [client] → Advanced → Token settings |

Also check: is the client sending the refresh token? Access tokens are short-lived by design — the client must use refresh tokens to renew.

---

### Brute force locked account

```bash
# Check if locked
curl "$KC/admin/realms/$REALM/attack-detection/brute-force/users/$USER_ID" \
  -H "Authorization: Bearer $ADMIN_TOKEN" | jq '{disabled, numFailures, lastIPFailure}'

# Unlock
curl -X DELETE "$KC/admin/realms/$REALM/attack-detection/brute-force/users/$USER_ID" \
  -H "Authorization: Bearer $ADMIN_TOKEN"
```

---

### Admin console inaccessible

```bash
# Check Keycloak is running
curl http://keycloak:8080/health/ready

# Check logs
docker logs keycloak 2>&1 | tail -50
kubectl logs -n keycloak statefulset/keycloak --tail=50

# Common causes:
# - DB connection failed at startup → check DB credentials and connectivity
# - Hostname misconfigured → KC_HOSTNAME must match what the browser uses
# - Proxy headers not passed → add X-Forwarded-Proto header from reverse proxy
```

---

### "Realm not found" error

- Realm name is case-sensitive — `myRealm` ≠ `myrealm`
- Realm may not exist — check: `GET /admin/realms` (lists all realms)
- If using `.well-known` endpoint, the realm must exist at that exact URL

---

## Logging

```bash
# Docker — view logs
docker logs keycloak -f

# K8s
kubectl logs -n keycloak statefulset/keycloak -f

# Increase log level temporarily (env var or keycloak.conf)
KC_LOG_LEVEL=debug                          # all debug
KC_LOG_LEVEL=org.keycloak:debug             # Keycloak only
KC_LOG_LEVEL=org.keycloak.storage.ldap:debug # LDAP only
KC_LOG_LEVEL=org.keycloak.events:debug      # auth events only

# Enable event logging in realm
# Realm settings → Events → Save events → On
# Then: Realm → Events → Login events tab
```

## Check events in the Admin Console

`Realm → Events → Login events` — shows every login, logout, token request, error.

`Realm → Events → Admin events` — shows every admin API call (who changed what, when).

Enable both under `Realm settings → Events`.
