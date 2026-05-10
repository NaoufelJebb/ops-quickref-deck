# LDAP QUICK REFERENCE

## AD vs OpenLDAP field mapping

| Setting | Active Directory | OpenLDAP |
|---|---|---|
| Vendor | Active Directory | Other |
| Username attr | `sAMAccountName` | `uid` |
| RDN attr | `cn` | `uid` |
| UUID attr | `objectGUID` | `entryUUID` |
| User object classes | `person, organizationalPerson, user` | `inetOrgPerson, organizationalPerson` |
| Email attr | `mail` | `mail` |
| First name attr | `givenName` | `givenName` |
| Last name attr | `sn` | `sn` |
| Group member attr | `member` | `member` or `memberUid` |
| Group object class | `group` | `groupOfNames` or `posixGroup` |

## Common LDAP filters

```ldap
# All users (AD)
(objectClass=user)

# Only enabled users (AD)
(&(objectClass=user)(!(userAccountControl:1.2.840.113556.1.4.803:=2)))

# Members of a specific group (AD)
(&(objectClass=user)(memberOf=CN=App-Users,OU=Groups,DC=corp,DC=local))

# Enabled members of a group (AD)
(&(objectClass=user)
  (memberOf=CN=App-Users,OU=Groups,DC=corp,DC=local)
  (!(userAccountControl:1.2.840.113556.1.4.803:=2)))

# All users (OpenLDAP)
(objectClass=inetOrgPerson)

# Users in a department (OpenLDAP)
(&(objectClass=inetOrgPerson)(department=Engineering))
```

## Bind DN formats

```
Active Directory:   CN=svc-keycloak,OU=Service Accounts,DC=corp,DC=local
OpenLDAP:           cn=admin,dc=example,dc=com
                or  uid=admin,ou=people,dc=example,dc=com
```

## Users DN examples

```
Active Directory:   OU=Users,DC=corp,DC=local
                or  OU=Users,OU=EMEA,DC=corp,DC=local
OpenLDAP:           ou=people,dc=example,dc=com
```

## Connection URLs

```
LDAP (plaintext):   ldap://dc1.corp.local:389
LDAPS (TLS):        ldaps://dc1.corp.local:636
StartTLS:           ldap://dc1.corp.local:389  + enable StartTLS toggle
```

## Test from CLI

```bash
# Test bind + user search
ldapsearch -H ldap://dc1.corp.local:389 \
  -D "CN=svc-keycloak,OU=Service Accounts,DC=corp,DC=local" \
  -w "$BIND_PASSWORD" \
  -b "OU=Users,DC=corp,DC=local" \
  "(sAMAccountName=testuser)" \
  sAMAccountName mail givenName sn memberOf

# Test group membership
ldapsearch -H ldap://dc1.corp.local:389 \
  -D "CN=svc-keycloak,OU=Service Accounts,DC=corp,DC=local" \
  -w "$BIND_PASSWORD" \
  -b "OU=Groups,DC=corp,DC=local" \
  "(cn=App-Users)" member
```

## Sync triggers (API)

```bash
# Full sync
curl -X POST "$KC/admin/realms/$REALM/user-storage/$PROVIDER_ID/sync?action=triggerFullSync" \
  -H "Authorization: Bearer $ADMIN_TOKEN"

# Delta sync (changed users only)
curl -X POST "$KC/admin/realms/$REALM/user-storage/$PROVIDER_ID/sync?action=triggerChangedUsersSync" \
  -H "Authorization: Bearer $ADMIN_TOKEN"

# Get LDAP provider ID
curl "$KC/admin/realms/$REALM/components?type=org.keycloak.storage.UserStorageProvider" \
  -H "Authorization: Bearer $ADMIN_TOKEN" | jq '.[0].id'
```
