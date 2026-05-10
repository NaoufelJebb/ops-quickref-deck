# AD GROUPS CONFIGURATION

Map Active Directory groups into Keycloak groups or roles so users inherit permissions automatically at login.

---

## Prerequisites

- LDAP/AD federation already connected and working
- Test connection + test authentication passing
- At least one user syncing correctly

If not: [LDAP-AD.md](../LDAP-AD.md) first.

---

## Step 1 — Add a Group Mapper to your LDAP provider

`Realm → User Federation → [your AD provider] → Mappers → Add mapper`

```
Name:                           ad-groups
Mapper Type:                    group-ldap-mapper

LDAP Groups DN:                 OU=Groups,DC=corp,DC=local
Group Name LDAP Attribute:      cn
Group Object Classes:           group
Membership LDAP Attribute:      member
Membership Attribute Type:      DN
Membership User LDAP Attribute: sAMAccountName (leave default)

Preserve Group Inheritance:     On   ← keeps nested OU structure
Ignore Missing Groups:          Off
Drop Non-Existing Groups During Sync: Off  ← safe default

Groups Path:                    /    ← root, or /AD if you want a subfolder
Mode:                           LDAP_ONLY  ← groups only from AD, not local KC groups
```

Save → click **Sync LDAP Groups to Keycloak** button that appears.

---

## Step 2 — Verify groups imported

`Realm → Groups`

You should see your AD groups listed. If nested groups exist and `Preserve Group Inheritance` is on, they appear as a tree.

```bash
# Or via API
curl "$KC/admin/realms/$REALM/groups" \
  -H "Authorization: Bearer $ADMIN_TOKEN" | jq '.[].name'
```

---

## Step 3 — Assign realm roles to AD groups

This is what gives AD group members permissions in Keycloak.

```bash
# Get the group ID
GROUP_ID=$(curl -s "$KC/admin/realms/$REALM/groups?search=App-Admins" \
  -H "Authorization: Bearer $ADMIN_TOKEN" | jq -r '.[0].id')

# Get the role
ROLE=$(curl -s "$KC/admin/realms/$REALM/roles/app-admin" \
  -H "Authorization: Bearer $ADMIN_TOKEN")

# Assign realm role to the group
curl -X POST "$KC/admin/realms/$REALM/groups/$GROUP_ID/role-mappings/realm" \
  -H "Authorization: Bearer $ADMIN_TOKEN" \
  -H "Content-Type: application/json" \
  -d "[$ROLE]"
```

Or in the UI: `Realm → Groups → [group] → Role mappings → Assign role`

---

## Step 4 — Add groups claim to tokens

By default, group membership is not in the token. Add a mapper to include it.

`Realm → Client scopes → [your client's dedicated scope] → Mappers → Add mapper → By configuration → Group Membership`

```
Name:                   groups
Token Claim Name:       groups
Full group path:        Off   ← just the group name, not /AD/App-Admins
Add to ID token:        On
Add to access token:    On
Add to userinfo:        On
```

Token will now contain:
```json
{
  "groups": ["App-Admins", "Dev-Team"]
}
```

---

## Step 5 — Test it

`Clients → [client] → Client scopes → Evaluate → enter AD username → Generate access token`

Check:
- `groups` claim lists the user's AD groups
- `realm_access.roles` contains roles assigned to those groups

---

## Sync modes

| Mode | Behaviour |
|---|---|
| `LDAP_ONLY` | Groups only created/updated from AD — never create local-only groups |
| `READ_ONLY` | Same — safe for production |
| `IMPORT` | One-time import, then managed locally |

Stick with `LDAP_ONLY` if AD is your source of truth.

---

## Nested groups (recursive membership)

AD supports nested groups (`memberOf` transitively). Keycloak's group mapper follows direct `member` attributes only by default.

To handle nested groups, add a second mapper:

```
Mapper Type:    msad-group-membership-ldap-mapper
```

Or use an LDAP filter that expands membership recursively:

```ldap
(member:1.2.840.113556.1.4.1941:=CN=username,OU=Users,DC=corp,DC=local)
```

---

## Map specific AD groups to specific Keycloak roles

Use a **Role Mapper** instead of (or in addition to) the Group Mapper:

`LDAP provider → Mappers → Add mapper → role-ldap-mapper`

```
Name:                        ad-role-map
Mapper Type:                 role-ldap-mapper
LDAP Roles DN:               OU=Groups,DC=corp,DC=local
Role Name LDAP Attribute:    cn
Role Object Classes:         group
Membership LDAP Attribute:   member
Membership Attribute Type:   DN
Roles LDAP Filter:           (cn=KC-*)   ← only sync groups prefixed KC-
Use Realm Roles Mapping:     On
Client ID:                   (leave empty for realm roles)
```

AD groups prefixed `KC-` map directly to Keycloak realm roles of the same name.

---

## Trigger sync manually

```bash
# Sync groups from AD to Keycloak
curl -X POST "$KC/admin/realms/$REALM/user-storage/$PROVIDER_ID/sync?action=triggerFullSync" \
  -H "Authorization: Bearer $ADMIN_TOKEN"

# Get LDAP provider ID if you don't have it
curl "$KC/admin/realms/$REALM/components?type=org.keycloak.storage.UserStorageProvider" \
  -H "Authorization: Bearer $ADMIN_TOKEN" | jq '.[0] | {id, name}'
```

---

## Troubleshooting

| Symptom | Check |
|---|---|
| No groups appear after sync | Wrong Groups DN — verify with `ldapsearch` |
| Groups appear but users not in them | `member` attribute uses DN format — ensure `Membership Attribute Type: DN` |
| Nested groups not followed | Add `msad-group-membership-ldap-mapper` or expand filter |
| Groups in token but roles missing | Role not assigned to the group in `Groups → Role mappings` |
| Groups claim missing from token | Group Membership mapper not added to client scope |
| Old deleted AD groups still in Keycloak | Enable `Drop Non-Existing Groups During Sync` and re-sync |

```bash
# Test group membership in AD directly
ldapsearch -H ldap://dc1.corp.local:389 \
  -D "CN=svc-keycloak,OU=Service Accounts,DC=corp,DC=local" \
  -w "$BIND_PASSWORD" \
  -b "OU=Groups,DC=corp,DC=local" \
  "(cn=App-Admins)" member cn
```
