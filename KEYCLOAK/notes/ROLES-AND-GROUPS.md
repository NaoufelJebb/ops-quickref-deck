# ROLES AND GROUPS

## Role types

| Type | Scope | Defined in |
|---|---|---|
| **Realm role** | Across all clients | `Realm → Realm roles` |
| **Client role** | Scoped to one client | `Clients → [client] → Roles` |
| **Composite role** | Role that includes other roles | Any role with sub-roles assigned |

## Realm roles — create and assign

```bash
# Create via Admin API
curl -X POST "https://keycloak/admin/realms/myrealm/roles" \
  -H "Authorization: Bearer $ADMIN_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{"name": "app-admin", "description": "Application administrator"}'

# Assign realm role to a user
curl -X POST "https://keycloak/admin/realms/myrealm/users/<user-id>/role-mappings/realm" \
  -H "Authorization: Bearer $ADMIN_TOKEN" \
  -H "Content-Type: application/json" \
  -d '[{"id": "<role-id>", "name": "app-admin"}]'
```

## Client roles — create and assign

```bash
# Create client role
curl -X POST "https://keycloak/admin/realms/myrealm/clients/<client-id>/roles" \
  -H "Authorization: Bearer $ADMIN_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{"name": "read", "description": "Read access"}'

# Assign client role to user
curl -X POST "https://keycloak/admin/realms/myrealm/users/<user-id>/role-mappings/clients/<client-id>" \
  -H "Authorization: Bearer $ADMIN_TOKEN" \
  -H "Content-Type: application/json" \
  -d '[{"id": "<role-id>", "name": "read"}]'
```

## Roles in tokens

Realm roles appear at:
```json
{
  "realm_access": {
    "roles": ["app-admin", "offline_access"]
  }
}
```

Client roles appear at:
```json
{
  "resource_access": {
    "my-client": {
      "roles": ["read", "write"]
    }
  }
}
```

---

## Groups

Groups are collections of users. Assign roles to a group — members inherit them.

```bash
# Create group
curl -X POST "https://keycloak/admin/realms/myrealm/groups" \
  -H "Authorization: Bearer $ADMIN_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{"name": "ops-team"}'

# Assign realm role to group
curl -X POST "https://keycloak/admin/realms/myrealm/groups/<group-id>/role-mappings/realm" \
  -H "Authorization: Bearer $ADMIN_TOKEN" \
  -H "Content-Type: application/json" \
  -d '[{"id": "<role-id>", "name": "app-admin"}]'

# Add user to group
curl -X PUT "https://keycloak/admin/realms/myrealm/users/<user-id>/groups/<group-id>" \
  -H "Authorization: Bearer $ADMIN_TOKEN"
```

### Group hierarchy

Groups can be nested:

```
/org
  /org/engineering
    /org/engineering/backend
    /org/engineering/frontend
  /org/operations
```

Child groups inherit roles from parent groups.

### Default group

Users are automatically added to the default group on registration:

`Realm settings → User registration → Default groups`

---

## Composite roles

A composite role bundles multiple roles. Assign the composite — user gets all included roles.

```bash
# Create composite role
curl -X POST "https://keycloak/admin/realms/myrealm/roles" \
  -H "Authorization: Bearer $ADMIN_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{"name": "super-admin", "composite": true}'

# Add sub-roles to composite
curl -X POST "https://keycloak/admin/realms/myrealm/roles/super-admin/composites" \
  -H "Authorization: Bearer $ADMIN_TOKEN" \
  -H "Content-Type: application/json" \
  -d '[{"id": "<role-id>", "name": "app-admin"}, {"id": "<role-id>", "name": "reporting-viewer"}]'
```

---

## Role evaluation — test what a user gets

`Realm → Clients → [client] → Client scopes → Evaluate`

Enter a username → see exactly what claims and roles will appear in the token. Invaluable for debugging authorization issues.

---

## Authorization Services (fine-grained)

For complex authorization beyond simple role checks, enable Authorization Services on a client:

`Clients → [client] → Authorization tab`

Concepts:
- **Resource** — something to protect (`/api/orders`)
- **Scope** — an action (`read`, `write`, `delete`)
- **Policy** — a rule (`user must have role X`, `time between 9–17`, `JavaScript expression`)
- **Permission** — binds a resource + scope to a policy

Clients call the `token` endpoint with `response_type=decision` to get authorization decisions.
