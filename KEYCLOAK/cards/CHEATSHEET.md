# CHEATSHEET

## Env setup

```bash
export KC=https://keycloak
export REALM=myrealm

# Get admin token
ADMIN_TOKEN=$(curl -s -X POST "$KC/realms/master/protocol/openid-connect/token" \
  -d "grant_type=password&client_id=admin-cli&username=admin&password=$KC_ADMIN_PASSWORD" \
  | jq -r '.access_token')
```

## Users

```bash
# Search
curl "$KC/admin/realms/$REALM/users?search=jane&max=10" -H "Authorization: Bearer $ADMIN_TOKEN"

# Get ID by username
USER_ID=$(curl -s "$KC/admin/realms/$REALM/users?username=jane.doe&exact=true" \
  -H "Authorization: Bearer $ADMIN_TOKEN" | jq -r '.[0].id')

# Disable
curl -X PUT "$KC/admin/realms/$REALM/users/$USER_ID" \
  -H "Authorization: Bearer $ADMIN_TOKEN" -H "Content-Type: application/json" \
  -d '{"enabled": false}'

# Reset password
curl -X PUT "$KC/admin/realms/$REALM/users/$USER_ID/reset-password" \
  -H "Authorization: Bearer $ADMIN_TOKEN" -H "Content-Type: application/json" \
  -d '{"type":"password","value":"TempPass123!","temporary":true}'

# Send reset password email
curl -X PUT "$KC/admin/realms/$REALM/users/$USER_ID/execute-actions-email" \
  -H "Authorization: Bearer $ADMIN_TOKEN" -H "Content-Type: application/json" \
  -d '["UPDATE_PASSWORD"]'

# Force logout all sessions
curl -X DELETE "$KC/admin/realms/$REALM/users/$USER_ID/sessions" \
  -H "Authorization: Bearer $ADMIN_TOKEN"

# Unlock brute-force
curl -X DELETE "$KC/admin/realms/$REALM/attack-detection/brute-force/users/$USER_ID" \
  -H "Authorization: Bearer $ADMIN_TOKEN"

# Get roles
curl "$KC/admin/realms/$REALM/users/$USER_ID/role-mappings" \
  -H "Authorization: Bearer $ADMIN_TOKEN" | jq '.realmMappings[].name'

# Add to group
curl -X PUT "$KC/admin/realms/$REALM/users/$USER_ID/groups/$GROUP_ID" \
  -H "Authorization: Bearer $ADMIN_TOKEN"
```

## Sessions

```bash
# Active session stats
curl "$KC/admin/realms/$REALM/sessions/stats" -H "Authorization: Bearer $ADMIN_TOKEN"

# Logout entire realm (emergency)
curl -X POST "$KC/admin/realms/$REALM/logout-all" -H "Authorization: Bearer $ADMIN_TOKEN"
```

## LDAP sync

```bash
# Full sync
curl -X POST "$KC/admin/realms/$REALM/user-storage/$PROVIDER_ID/sync?action=triggerFullSync" \
  -H "Authorization: Bearer $ADMIN_TOKEN"

# Changed users sync
curl -X POST "$KC/admin/realms/$REALM/user-storage/$PROVIDER_ID/sync?action=triggerChangedUsersSync" \
  -H "Authorization: Bearer $ADMIN_TOKEN"
```

## Tokens

```bash
# Client credentials token
curl -X POST "$KC/realms/$REALM/protocol/openid-connect/token" \
  -d "grant_type=client_credentials&client_id=my-client&client_secret=$SECRET"

# Introspect a token
curl -X POST "$KC/realms/$REALM/protocol/openid-connect/token/introspect" \
  -u "my-client:$SECRET" -d "token=$ACCESS_TOKEN" | jq '{active, sub, exp}'

# Revoke a token
curl -X POST "$KC/realms/$REALM/protocol/openid-connect/revoke" \
  -u "my-client:$SECRET" -d "token=$REFRESH_TOKEN"
```

## Key UI paths

| Task | Navigation |
|---|---|
| Create realm | Top-left dropdown → Create realm |
| Create client | Realm → Clients → Create client |
| Create user | Realm → Users → Add user |
| Manage LDAP | Realm → User Federation → Add provider → LDAP |
| Add IdP | Realm → Identity Providers → Add provider |
| Configure flows | Realm → Authentication → Flows |
| Enable MFA | Authentication → Flows → browser → copy → add OTP |
| Token lifetimes | Realm settings → Tokens |
| Brute force | Realm settings → Security defenses |
| Events/audit | Realm settings → Events |
| Export realm | Realm → Realm settings → Export (or partial-export API) |
| Token evaluator | Clients → [client] → Client scopes → Evaluate |
| Signing keys | Realm settings → Keys |
