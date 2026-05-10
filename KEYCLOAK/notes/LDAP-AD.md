# LDAP / ACTIVE DIRECTORY

## Overview

Keycloak connects to LDAP/AD as a **User Federation** provider. Users are stored in your directory — Keycloak syncs or proxies them into the realm. It does not copy passwords; it validates credentials against the LDAP server at login time.

```
User logs in → Keycloak → binds to LDAP/AD → validates credentials
                       → syncs user attributes → issues token
```

## Add LDAP federation

`Realm → User Federation → Add provider → LDAP`

---

## Core settings

### Connection

| Setting | Active Directory | OpenLDAP |
|---|---|---|
| **Vendor** | Active Directory | Other |
| **Connection URL** | `ldap://dc1.corp.local:389` | `ldap://ldap.example.com:389` |
| **Connection URL (TLS)** | `ldaps://dc1.corp.local:636` | `ldaps://ldap.example.com:636` |
| **Enable StartTLS** | Optional | Optional |
| **Use Truststore SPI** | Only if custom CA | Only if custom CA |
| **Bind DN** | `CN=svc-keycloak,OU=Service Accounts,DC=corp,DC=local` | `cn=admin,dc=example,dc=com` |
| **Bind Credential** | service account password | bind password |
| **Test connection** | ✅ Always test before saving | |
| **Test authentication** | ✅ Always test after connection | |

### Search

| Setting | Active Directory | OpenLDAP |
|---|---|---|
| **Edit mode** | `READ_ONLY` (recommended) or `WRITABLE` | `READ_ONLY` or `WRITABLE` |
| **Users DN** | `OU=Users,DC=corp,DC=local` | `ou=people,dc=example,dc=com` |
| **Username LDAP attribute** | `sAMAccountName` | `uid` |
| **RDN LDAP attribute** | `cn` | `uid` |
| **UUID LDAP attribute** | `objectGUID` | `entryUUID` |
| **User Object Classes** | `person, organizationalPerson, user` | `inetOrgPerson, organizationalPerson` |
| **User LDAP Filter** | `(memberOf=CN=App-Users,OU=Groups,DC=corp,DC=local)` | `(objectClass=inetOrgPerson)` |

### Sync

| Setting | Value | Notes |
|---|---|---|
| **Import Users** | On | Sync users into Keycloak DB |
| **Sync Registrations** | Off (read-only) | Allow creating users back to LDAP |
| **Batch Size** | 1000 | Max users per sync page |
| **Periodic Full Sync** | On, 86400s | Full sync daily |
| **Periodic Changed Sync** | On, 3600s | Delta sync hourly |

---

## Active Directory specific settings

```
Vendor:                     Active Directory
Username LDAP attribute:    sAMAccountName
UUID LDAP attribute:        objectGUID
User Object Classes:        person, organizationalPerson, user

# Filter to only sync members of a specific AD group:
User LDAP Filter:           (&(objectClass=user)(memberOf=CN=Keycloak-Users,OU=Groups,DC=corp,DC=local))

# Filter out disabled accounts:
User LDAP Filter:           (&(objectClass=user)(!(userAccountControl:1.2.840.113556.1.4.803:=2)))

# Both — only enabled members of group:
User LDAP Filter:           (&(objectClass=user)(memberOf=CN=Keycloak-Users,OU=Groups,DC=corp,DC=local)(!(userAccountControl:1.2.840.113556.1.4.803:=2)))
```

### AD password policy integration

Enable `Password Policy` mapper to enforce AD password change requirements:

`LDAP provider → Mappers → Add mapper → msad-user-account-control-mapper`

This handles:
- Expired passwords (forces change on next login)
- Locked accounts (shows appropriate error)
- Account disabled state

---

## OpenLDAP specific settings

```
Vendor:                     Other
Username LDAP attribute:    uid
RDN LDAP attribute:         uid
UUID LDAP attribute:        entryUUID
User Object Classes:        inetOrgPerson, organizationalPerson, person

# Filter all users
User LDAP Filter:           (objectClass=inetOrgPerson)
```

---

## LDAP mappers — sync attributes

`LDAP provider → Mappers`

Mappers define which LDAP attributes map to which Keycloak user attributes.

### Default mappers (auto-created)

| Mapper name | LDAP attribute | Keycloak attribute |
|---|---|---|
| username | sAMAccountName / uid | username |
| email | mail | email |
| first name | givenName | firstName |
| last name | sn | lastName |
| full name | cn | (combined) |

### Add a custom attribute mapper

`Mappers → Add mapper → user-attribute-ldap-mapper`

```
Name:                   department
Mapper Type:            user-attribute-ldap-mapper
User Model Attribute:   department
LDAP Attribute:         department
Read Only:              true
Always Read Value From LDAP: false
Is Mandatory In LDAP:   false
```

### Group mapper — sync AD groups as Keycloak groups

`Mappers → Add mapper → group-ldap-mapper`

```
Name:                   ad-groups
Mapper Type:            group-ldap-mapper
LDAP Groups DN:         OU=Groups,DC=corp,DC=local
Group Name LDAP Attribute: cn
Group Object Classes:   group
Preserve Group Inheritance: true
Ignore Missing Groups:  false
Membership LDAP Attribute: member
Membership Attribute Type: DN
Groups Path:            /        (root, or a specific group path)
Drop Non-Existing Groups During Sync: false
```

### Role mapper — map LDAP groups directly to realm roles

`Mappers → Add mapper → role-ldap-mapper`

```
Name:                   ldap-roles
Mapper Type:            role-ldap-mapper
LDAP Roles DN:          OU=Groups,DC=corp,DC=local
Role Name LDAP Attribute: cn
Role Object Classes:    group
Membership LDAP Attribute: member
Roles LDAP Filter:      (cn=App-*)
Use Realm Roles Mapping: true
```

---

## Triggering sync manually

```bash
# Via Admin Console
Realm → User Federation → [your provider] → Synchronize all users
Realm → User Federation → [your provider] → Synchronize changed users

# Via Admin API
curl -X POST "https://keycloak/admin/realms/myrealm/user-storage/<provider-id>/sync?action=triggerFullSync" \
  -H "Authorization: Bearer $ADMIN_TOKEN"

curl -X POST "https://keycloak/admin/realms/myrealm/user-storage/<provider-id>/sync?action=triggerChangedUsersSync" \
  -H "Authorization: Bearer $ADMIN_TOKEN"
```

---

## LDAPS / TLS configuration

### Using a custom CA

```bash
# 1. Import the LDAP server CA cert into the Keycloak truststore
keytool -importcert \
  -alias ldap-ca \
  -file /path/to/ldap-ca.crt \
  -keystore /opt/keycloak/conf/truststore.jks \
  -storepass changeit -noprompt

# 2. Reference in keycloak.conf
spi-truststore-file-file=/opt/keycloak/conf/truststore.jks
spi-truststore-file-password=changeit
spi-truststore-file-hostname-verification-policy=WILDCARD
```

Then in the LDAP provider: `Use Truststore SPI: Always`

---

## Kerberos / SPNEGO (Windows SSO)

Enable transparent Windows SSO so AD users are logged in automatically in browsers.

`LDAP provider → Kerberos Integration`

```
Allow Kerberos Authentication: On
Kerberos Realm:                CORP.LOCAL
Server Principal:              HTTP/auth.example.com@CORP.LOCAL
KeyTab:                        /etc/keycloak/keycloak.keytab
Debug:                         Off (set On for troubleshooting)
Use Kerberos For Password Auth: On
```

Generate the keytab on the AD server:
```cmd
ktpass /out keycloak.keytab /mapuser svc-keycloak@CORP.LOCAL ^
  /princ HTTP/auth.example.com@CORP.LOCAL ^
  /pass * /ptype KRB5_NT_PRINCIPAL /crypto AES256-SHA1
```

Clients need to add `auth.example.com` to their browser's trusted intranet zone for SPNEGO to work automatically.

---

## Troubleshooting LDAP

| Symptom | Check |
|---|---|
| "Test connection" fails | Firewall between Keycloak and LDAP, wrong port, wrong URL |
| "Test authentication" fails | Wrong bind DN format, wrong password, account locked |
| Users not syncing | Wrong Users DN, wrong User Object Classes, filter too restrictive |
| Login fails for federated user | Edit mode is READ_ONLY with WRITABLE needed, or credential validation failing |
| Groups not appearing | Group mapper not configured, wrong Groups DN |
| `objectGUID` errors (AD) | Use `objectGUID` not `entryUUID` for AD UUID attribute |
| Kerberos not triggering | Browser not in trusted intranet zone, wrong SPN, bad keytab |

```bash
# Enable LDAP debug logging temporarily
# keycloak.conf or env var:
log-level=org.keycloak.storage.ldap:debug

# Test LDAP connectivity from the Keycloak container
ldapsearch -H ldap://dc1.corp.local:389 \
  -D "CN=svc-keycloak,OU=Service Accounts,DC=corp,DC=local" \
  -w "password" \
  -b "OU=Users,DC=corp,DC=local" \
  "(sAMAccountName=testuser)"
```
