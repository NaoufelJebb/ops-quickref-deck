# AUTHENTICATION AND IDENTITY

## OpenShift OAuth server

OpenShift has a built-in OAuth server that issues tokens to users and service accounts. It integrates with external identity providers — users authenticate with the IdP, OpenShift issues an OAuth token.

```
oc login → OAuth server → IdP (LDAP/AD, OIDC, GitHub…) → issues token → oc commands
```

## Configure identity providers

`oc edit oauth cluster`

Or patch via YAML:

```bash
oc apply -f oauth-config.yaml
```

---

## LDAP / Active Directory

```yaml
apiVersion: config.openshift.io/v1
kind: OAuth
metadata:
  name: cluster
spec:
  identityProviders:
    - name: ldap-ad
      mappingMethod: claim              # claim | lookup | generate | add
      type: LDAP
      ldap:
        url: "ldap://dc1.corp.local:389/OU=Users,DC=corp,DC=local?sAMAccountName"
        # Format: ldap://<host>/<base-dn>?<attribute>?<scope>?<filter>
        # scope: sub (subtree), one (one level), base
        bindDN: "CN=svc-openshift,OU=Service Accounts,DC=corp,DC=local"
        bindPassword:
          name: ldap-bind-password      # Secret in openshift-config namespace
        insecure: false                 # true only for dev (disables TLS)
        ca:
          name: ldap-ca-cert            # ConfigMap in openshift-config namespace
        attributes:
          id: ["sAMAccountName"]        # unique identifier
          email: ["mail"]
          name: ["cn"]
          preferredUsername: ["sAMAccountName"]
```

```bash
# Create the bind password secret (openshift-config namespace)
oc create secret generic ldap-bind-password \
  --from-literal=bindPassword='your-service-account-password' \
  -n openshift-config

# Create CA cert configmap (if LDAPS with custom CA)
oc create configmap ldap-ca-cert \
  --from-file=ca.crt=/path/to/ldap-ca.crt \
  -n openshift-config
```

### LDAP URL format

```
ldap://<host>:<port>/<base-dn>?<attribute>?<scope>?<filter>

Active Directory examples:
  ldap://dc1.corp.local:389/OU=Users,DC=corp,DC=local?sAMAccountName
  ldaps://dc1.corp.local:636/OU=Users,DC=corp,DC=local?sAMAccountName?sub?(objectClass=user)
  
  # Only enabled users:
  ldap://dc1.corp.local:389/OU=Users,DC=corp,DC=local?sAMAccountName?sub?(&(objectClass=user)(!(userAccountControl:1.2.840.113556.1.4.803:=2)))

OpenLDAP example:
  ldap://ldap.example.com:389/ou=people,dc=example,dc=com?uid
```

---

## OIDC (Keycloak, Okta, Azure AD, Google)

```yaml
spec:
  identityProviders:
    - name: keycloak
      mappingMethod: claim
      type: OpenID
      openID:
        clientID: openshift
        clientSecret:
          name: openshift-oidc-secret   # Secret in openshift-config
        claims:
          id: ["sub"]
          email: ["email"]
          name: ["name"]
          preferredUsername: ["preferred_username"]
        issuer: https://keycloak.example.com/realms/myrealm
        extraScopes: ["profile", "email"]
```

```bash
# Create OIDC client secret
oc create secret generic openshift-oidc-secret \
  --from-literal=clientSecret='your-keycloak-client-secret' \
  -n openshift-config
```

Keycloak client config:
- Client ID: `openshift`
- Valid redirect URIs: `https://oauth-openshift.apps.<cluster-domain>/oauth2callback/keycloak`
- Standard flow: On, Direct access grants: Off

---

## GitHub / GitLab

```yaml
spec:
  identityProviders:
    - name: github
      mappingMethod: claim
      type: GitHub
      github:
        clientID: <github-oauth-app-client-id>
        clientSecret:
          name: github-secret
        organizations:                # restrict to specific GitHub orgs
          - myorg
        teams:                        # or specific teams
          - myorg/platform-team
```

---

## HTPasswd (local users — dev/break-glass)

```bash
# Create htpasswd file
htpasswd -c -B -b htpasswd.tmp admin 'SecurePassword123!'
htpasswd -B -b htpasswd.tmp developer 'DevPassword456!'

# Create/update secret
oc create secret generic htpasswd-secret \
  --from-file=htpasswd=htpasswd.tmp \
  -n openshift-config

# Add htpasswd IdP
oc patch oauth cluster --type=merge -p '
spec:
  identityProviders:
  - name: htpasswd
    mappingMethod: claim
    type: HTPasswd
    htPasswd:
      fileData:
        name: htpasswd-secret'
```

---

## Sync LDAP groups to OpenShift groups

OpenShift can sync AD/LDAP groups for use in RBAC.

```yaml
# ldap-sync-config.yaml
kind: LDAPSyncConfig
apiVersion: v1
url: ldap://dc1.corp.local:389
bindDN: CN=svc-openshift,OU=Service Accounts,DC=corp,DC=local
bindPassword: your-password
insecure: false
ca: /path/to/ldap-ca.crt
groupUIDNameMapping:
  "CN=OCP-Admins,OU=Groups,DC=corp,DC=local": ocp-admins
  "CN=OCP-Developers,OU=Groups,DC=corp,DC=local": ocp-developers
activeDirectory:
  usersQuery:
    baseDN: OU=Users,DC=corp,DC=local
    scope: sub
    derefAliases: never
    filter: (objectClass=user)
    pageSize: 0
  userNameAttributes: [sAMAccountName]
  groupMembershipAttributes: [memberOf]
```

```bash
# Run group sync (dry-run first)
oc adm groups sync --sync-config=ldap-sync-config.yaml

# Apply
oc adm groups sync --sync-config=ldap-sync-config.yaml --confirm

# Prune groups removed from LDAP
oc adm groups prune --sync-config=ldap-sync-config.yaml --confirm

# Automate with a CronJob
oc create -f ldap-sync-cronjob.yaml
```

After sync, assign cluster roles to the synced groups:

```bash
oc adm policy add-cluster-role-to-group cluster-admin ocp-admins
oc adm policy add-role-to-group edit ocp-developers -n my-app
```

---

## Mapping methods

| Method | Behaviour |
|---|---|
| `claim` | Use identity provider's username claim as-is (default) |
| `lookup` | Only allow pre-provisioned users — no auto-create |
| `generate` | Auto-generate unique username on conflict |
| `add` | Add new identity to existing user if username matches |

---

## Remove the kubeadmin default user (post-setup)

```bash
# Only do this AFTER configuring at least one working IdP
# and granting cluster-admin to at least one user

oc delete secret kubeadmin -n kube-system
```
