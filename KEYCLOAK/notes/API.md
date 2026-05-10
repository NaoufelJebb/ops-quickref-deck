# API

## Get an admin token

```bash
export KC=https://keycloak
export REALM=myrealm

# Get admin token (master realm)
ADMIN_TOKEN=$(curl -s -X POST "$KC/realms/master/protocol/openid-connect/token" \
  -d "grant_type=password" \
  -d "client_id=admin-cli" \
  -d "username=admin" \
  -d "password=$KC_ADMIN_PASSWORD" \
  | jq -r '.access_token')

# Or use a service account in your realm (preferred for automation)
ADMIN_TOKEN=$(curl -s -X POST "$KC/realms/$REALM/protocol/openid-connect/token" \
  -d "grant_type=client_credentials" \
  -d "client_id=my-admin-client" \
  -d "client_secret=$CLIENT_SECRET" \
  | jq -r '.access_token')
```

---

## Realm operations

```bash
# List all realms
curl "$KC/admin/realms" -H "Authorization: Bearer $ADMIN_TOKEN" | jq '.[].realm'

# Get realm config
curl "$KC/admin/realms/$REALM" -H "Authorization: Bearer $ADMIN_TOKEN" | jq .

# Update realm setting
curl -X PUT "$KC/admin/realms/$REALM" \
  -H "Authorization: Bearer $ADMIN_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{"bruteForceProtected": true, "failureFactor": 5}'

# Export realm (via partial export endpoint)
curl -X POST "$KC/admin/realms/$REALM/partial-export?exportClients=true&exportGroupsAndRoles=true" \
  -H "Authorization: Bearer $ADMIN_TOKEN" > realm-export.json
```

---

## User operations

```bash
# List users
curl "$KC/admin/realms/$REALM/users?max=50" \
  -H "Authorization: Bearer $ADMIN_TOKEN" | jq '.[].username'

# Search users
curl "$KC/admin/realms/$REALM/users?search=john&max=10" \
  -H "Authorization: Bearer $ADMIN_TOKEN" | jq .

# Get user by username
USER_ID=$(curl -s "$KC/admin/realms/$REALM/users?username=john.doe&exact=true" \
  -H "Authorization: Bearer $ADMIN_TOKEN" | jq -r '.[0].id')

# Get user detail
curl "$KC/admin/realms/$REALM/users/$USER_ID" \
  -H "Authorization: Bearer $ADMIN_TOKEN" | jq .

# Create user
curl -X POST "$KC/admin/realms/$REALM/users" \
  -H "Authorization: Bearer $ADMIN_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
    "username": "jane.doe",
    "email": "jane@example.com",
    "firstName": "Jane",
    "lastName": "Doe",
    "enabled": true,
    "emailVerified": true,
    "credentials": [{"type": "password", "value": "TempPass123!", "temporary": true}]
  }'

# Update user
curl -X PUT "$KC/admin/realms/$REALM/users/$USER_ID" \
  -H "Authorization: Bearer $ADMIN_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{"firstName": "Jane", "attributes": {"department": ["Engineering"]}}'

# Disable user
curl -X PUT "$KC/admin/realms/$REALM/users/$USER_ID" \
  -H "Authorization: Bearer $ADMIN_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{"enabled": false}'

# Delete user
curl -X DELETE "$KC/admin/realms/$REALM/users/$USER_ID" \
  -H "Authorization: Bearer $ADMIN_TOKEN"

# Reset password
curl -X PUT "$KC/admin/realms/$REALM/users/$USER_ID/reset-password" \
  -H "Authorization: Bearer $ADMIN_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{"type": "password", "value": "NewPass123!", "temporary": true}'

# Send reset password email
curl -X PUT "$KC/admin/realms/$REALM/users/$USER_ID/execute-actions-email" \
  -H "Authorization: Bearer $ADMIN_TOKEN" \
  -H "Content-Type: application/json" \
  -d '["UPDATE_PASSWORD"]'

# Get user sessions
curl "$KC/admin/realms/$REALM/users/$USER_ID/sessions" \
  -H "Authorization: Bearer $ADMIN_TOKEN" | jq .

# Revoke all user sessions (force logout)
curl -X DELETE "$KC/admin/realms/$REALM/users/$USER_ID/sessions" \
  -H "Authorization: Bearer $ADMIN_TOKEN"

# Unlock brute-force locked user
curl -X DELETE "$KC/admin/realms/$REALM/attack-detection/brute-force/users/$USER_ID" \
  -H "Authorization: Bearer $ADMIN_TOKEN"
```

---

## Role operations

```bash
# List realm roles
curl "$KC/admin/realms/$REALM/roles" \
  -H "Authorization: Bearer $ADMIN_TOKEN" | jq '.[].name'

# Get user's realm roles
curl "$KC/admin/realms/$REALM/users/$USER_ID/role-mappings/realm" \
  -H "Authorization: Bearer $ADMIN_TOKEN" | jq '.[].name'

# Assign realm role to user
ROLE=$(curl -s "$KC/admin/realms/$REALM/roles/app-admin" \
  -H "Authorization: Bearer $ADMIN_TOKEN")
curl -X POST "$KC/admin/realms/$REALM/users/$USER_ID/role-mappings/realm" \
  -H "Authorization: Bearer $ADMIN_TOKEN" \
  -H "Content-Type: application/json" \
  -d "[$ROLE]"

# Remove realm role from user
curl -X DELETE "$KC/admin/realms/$REALM/users/$USER_ID/role-mappings/realm" \
  -H "Authorization: Bearer $ADMIN_TOKEN" \
  -H "Content-Type: application/json" \
  -d "[$ROLE]"
```

---

## Group operations

```bash
# List groups
curl "$KC/admin/realms/$REALM/groups" \
  -H "Authorization: Bearer $ADMIN_TOKEN" | jq '.[].name'

# Get group members
curl "$KC/admin/realms/$REALM/groups/$GROUP_ID/members" \
  -H "Authorization: Bearer $ADMIN_TOKEN" | jq '.[].username'

# Add user to group
curl -X PUT "$KC/admin/realms/$REALM/users/$USER_ID/groups/$GROUP_ID" \
  -H "Authorization: Bearer $ADMIN_TOKEN"

# Remove user from group
curl -X DELETE "$KC/admin/realms/$REALM/users/$USER_ID/groups/$GROUP_ID" \
  -H "Authorization: Bearer $ADMIN_TOKEN"
```

---

## Session operations

```bash
# Active sessions in realm
curl "$KC/admin/realms/$REALM/sessions/stats" \
  -H "Authorization: Bearer $ADMIN_TOKEN" | jq .

# Sessions for a specific client
curl "$KC/admin/realms/$REALM/clients/$CLIENT_ID/user-sessions?max=100" \
  -H "Authorization: Bearer $ADMIN_TOKEN" | jq .

# Revoke a specific session
curl -X DELETE "$KC/admin/realms/$REALM/sessions/$SESSION_ID" \
  -H "Authorization: Bearer $ADMIN_TOKEN"

# Logout all sessions in realm (use carefully)
curl -X POST "$KC/admin/realms/$REALM/logout-all" \
  -H "Authorization: Bearer $ADMIN_TOKEN"
```
