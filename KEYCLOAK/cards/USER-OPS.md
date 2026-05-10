# USER OPS

## Setup

```bash
export KC=https://keycloak
export REALM=myrealm
ADMIN_TOKEN=$(curl -s -X POST "$KC/realms/master/protocol/openid-connect/token" \
  -d "grant_type=password&client_id=admin-cli&username=admin&password=$KC_ADMIN_PASSWORD" \
  | jq -r '.access_token')

# Get user ID by username
USER_ID=$(curl -s "$KC/admin/realms/$REALM/users?username=john.doe&exact=true" \
  -H "Authorization: Bearer $ADMIN_TOKEN" | jq -r '.[0].id')
```

## Common actions

```bash
# Create user
curl -X POST "$KC/admin/realms/$REALM/users" \
  -H "Authorization: Bearer $ADMIN_TOKEN" -H "Content-Type: application/json" \
  -d '{"username":"jane.doe","email":"jane@example.com","firstName":"Jane","lastName":"Doe","enabled":true,"emailVerified":true}'

# Disable user
curl -X PUT "$KC/admin/realms/$REALM/users/$USER_ID" \
  -H "Authorization: Bearer $ADMIN_TOKEN" -H "Content-Type: application/json" \
  -d '{"enabled": false}'

# Reset password (temporary)
curl -X PUT "$KC/admin/realms/$REALM/users/$USER_ID/reset-password" \
  -H "Authorization: Bearer $ADMIN_TOKEN" -H "Content-Type: application/json" \
  -d '{"type":"password","value":"TempPass123!","temporary":true}'

# Send password reset email
curl -X PUT "$KC/admin/realms/$REALM/users/$USER_ID/execute-actions-email" \
  -H "Authorization: Bearer $ADMIN_TOKEN" -H "Content-Type: application/json" \
  -d '["UPDATE_PASSWORD"]'

# Force MFA setup email
curl -X PUT "$KC/admin/realms/$REALM/users/$USER_ID/execute-actions-email" \
  -H "Authorization: Bearer $ADMIN_TOKEN" -H "Content-Type: application/json" \
  -d '["CONFIGURE_TOTP"]'

# Force logout all sessions
curl -X DELETE "$KC/admin/realms/$REALM/users/$USER_ID/sessions" \
  -H "Authorization: Bearer $ADMIN_TOKEN"

# Unlock brute-force lockout
curl -X DELETE "$KC/admin/realms/$REALM/attack-detection/brute-force/users/$USER_ID" \
  -H "Authorization: Bearer $ADMIN_TOKEN"

# Add to group
curl -X PUT "$KC/admin/realms/$REALM/users/$USER_ID/groups/$GROUP_ID" \
  -H "Authorization: Bearer $ADMIN_TOKEN"

# Remove from group
curl -X DELETE "$KC/admin/realms/$REALM/users/$USER_ID/groups/$GROUP_ID" \
  -H "Authorization: Bearer $ADMIN_TOKEN"

# Assign realm role
ROLE=$(curl -s "$KC/admin/realms/$REALM/roles/app-admin" -H "Authorization: Bearer $ADMIN_TOKEN")
curl -X POST "$KC/admin/realms/$REALM/users/$USER_ID/role-mappings/realm" \
  -H "Authorization: Bearer $ADMIN_TOKEN" -H "Content-Type: application/json" -d "[$ROLE]"

# Delete user
curl -X DELETE "$KC/admin/realms/$REALM/users/$USER_ID" \
  -H "Authorization: Bearer $ADMIN_TOKEN"
```

## Check user state

```bash
# All user details
curl "$KC/admin/realms/$REALM/users/$USER_ID" -H "Authorization: Bearer $ADMIN_TOKEN" | jq .

# Roles
curl "$KC/admin/realms/$REALM/users/$USER_ID/role-mappings" \
  -H "Authorization: Bearer $ADMIN_TOKEN" | jq '{realm: .realmMappings[].name}'

# Groups
curl "$KC/admin/realms/$REALM/users/$USER_ID/groups" \
  -H "Authorization: Bearer $ADMIN_TOKEN" | jq '.[].name'

# Active sessions
curl "$KC/admin/realms/$REALM/users/$USER_ID/sessions" \
  -H "Authorization: Bearer $ADMIN_TOKEN" | jq '.[].id'

# Brute force status
curl "$KC/admin/realms/$REALM/attack-detection/brute-force/users/$USER_ID" \
  -H "Authorization: Bearer $ADMIN_TOKEN" | jq '{disabled, numFailures}'
```
