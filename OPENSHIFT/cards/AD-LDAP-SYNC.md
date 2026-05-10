# AD / LDAP — LOGIN AND GROUP SYNC

## Step 1 — Create the bind password secret

```bash
oc create secret generic ldap-bind-password \
  --from-literal=bindPassword='your-service-account-password' \
  -n openshift-config
```

If using LDAPS with a custom CA:

```bash
oc create configmap ldap-ca-cert \
  --from-file=ca.crt=/path/to/ldap-ca.crt \
  -n openshift-config
```

---

## Step 2 — Configure OAuth identity provider

```bash
oc apply -f - <<EOF
apiVersion: config.openshift.io/v1
kind: OAuth
metadata:
  name: cluster
spec:
  identityProviders:
    - name: corp-ad
      mappingMethod: claim
      type: LDAP
      ldap:
        url: "ldap://dc1.corp.local:389/OU=Users,DC=corp,DC=local?sAMAccountName?sub?(objectClass=user)"
        bindDN: "CN=svc-openshift,OU=Service Accounts,DC=corp,DC=local"
        bindPassword:
          name: ldap-bind-password
        insecure: false
        ca:
          name: ldap-ca-cert
        attributes:
          id:                ["sAMAccountName"]
          email:             ["mail"]
          name:              ["cn"]
          preferredUsername: ["sAMAccountName"]
EOF
```

Wait 1–2 min for OAuth pods to restart, then test:

```bash
oc login -u testuser -p testpassword https://api.<cluster>:6443
```

---

## Step 3 — Create LDAP group sync config

```yaml
# ldap-sync-config.yaml
kind: LDAPSyncConfig
apiVersion: v1
url: ldap://dc1.corp.local:389
bindDN: CN=svc-openshift,OU=Service Accounts,DC=corp,DC=local
bindPassword: your-password
insecure: false
ca: /etc/ldap-ca/ca.crt

groupUIDNameMapping:
  "CN=OCP-Admins,OU=Groups,DC=corp,DC=local":     ocp-admins
  "CN=OCP-Developers,OU=Groups,DC=corp,DC=local": ocp-developers
  "CN=OCP-Viewers,OU=Groups,DC=corp,DC=local":    ocp-viewers

activeDirectory:
  usersQuery:
    baseDN: "OU=Users,DC=corp,DC=local"
    scope: sub
    derefAliases: never
    filter: "(objectClass=user)"
    pageSize: 0
  userNameAttributes:         [sAMAccountName]
  groupMembershipAttributes:  [memberOf]
```

---

## Step 4 — Run group sync

```bash
# Dry run first — see what would change
oc adm groups sync --sync-config=ldap-sync-config.yaml

# Apply
oc adm groups sync --sync-config=ldap-sync-config.yaml --confirm

# Verify groups were created
oc get groups
oc describe group ocp-admins
```

---

## Step 5 — Assign cluster roles to synced groups

```bash
# Cluster admins
oc adm policy add-cluster-role-to-group cluster-admin ocp-admins

# Developers — edit access in their namespace
oc adm policy add-role-to-group edit ocp-developers -n my-app

# Viewers — read-only
oc adm policy add-role-to-group view ocp-viewers -n my-app

# Verify
oc get rolebindings -n my-app
oc get clusterrolebindings | grep ocp-admins
```

---

## Step 6 — Automate group sync with a CronJob

```yaml
apiVersion: batch/v1
kind: CronJob
metadata:
  name: ldap-group-sync
  namespace: openshift-config
spec:
  schedule: "0 * * * *"               # every hour
  jobTemplate:
    spec:
      template:
        spec:
          serviceAccountName: ldap-group-syncer
          restartPolicy: OnFailure
          containers:
            - name: ldap-group-sync
              image: registry.redhat.io/openshift4/ose-cli:latest
              command:
                - oc
                - adm
                - groups
                - sync
                - --sync-config=/etc/ldap-sync/ldap-sync-config.yaml
                - --confirm
              volumeMounts:
                - name: ldap-sync-config
                  mountPath: /etc/ldap-sync
                - name: ldap-ca
                  mountPath: /etc/ldap-ca
          volumes:
            - name: ldap-sync-config
              configMap:
                name: ldap-sync-config
            - name: ldap-ca
              configMap:
                name: ldap-ca-cert
```

```bash
# Create the service account and give it permission to sync groups
oc create sa ldap-group-syncer -n openshift-config
oc adm policy add-cluster-role-to-user system:auth-delegator \
  -z ldap-group-syncer -n openshift-config
oc create clusterrole ldap-group-syncer \
  --verb=get,list,create,update --resource=groups.user.openshift.io
oc adm policy add-cluster-role-to-user ldap-group-syncer \
  -z ldap-group-syncer -n openshift-config
```

---

## Prune removed groups

```bash
# Remove OpenShift groups that no longer exist in LDAP
oc adm groups prune --sync-config=ldap-sync-config.yaml --confirm
```

---

## LDAP URL format reference

```
ldap://<host>/<base-dn>?<attribute>?<scope>?<filter>

scope:  sub (subtree) | one (one level) | base

# All users (AD)
ldap://dc1.corp.local:389/OU=Users,DC=corp,DC=local?sAMAccountName

# Enabled users only (AD)
ldap://dc1.corp.local:389/OU=Users,DC=corp,DC=local?sAMAccountName?sub?(&(objectClass=user)(!(userAccountControl:1.2.840.113556.1.4.803:=2)))

# LDAPS
ldaps://dc1.corp.local:636/OU=Users,DC=corp,DC=local?sAMAccountName

# OpenLDAP
ldap://ldap.example.com:389/ou=people,dc=example,dc=com?uid
```

---

## Troubleshooting

| Symptom | Check |
|---|---|
| Login fails with "invalid credentials" | Wrong Users DN, wrong attribute, test with `ldapsearch` |
| Login works but no group membership | Group sync not run, or wrong `groupMembershipAttributes` |
| Group sync shows no groups | Wrong Groups DN in sync config |
| CA error on LDAPS | Wrong CA cert in ConfigMap, or cert not uploaded to openshift-config |
| OAuth pods not restarting | `oc get pods -n openshift-authentication` — check logs |

```bash
# Test LDAP connectivity from inside the cluster
oc debug -n openshift-authentication -- \
  ldapsearch -H ldap://dc1.corp.local:389 \
  -D "CN=svc-openshift,OU=Service Accounts,DC=corp,DC=local" \
  -w "$BIND_PASS" \
  -b "OU=Users,DC=corp,DC=local" "(sAMAccountName=testuser)"

# Check OAuth operator logs
oc logs -n openshift-authentication-operator \
  deployment/authentication-operator | tail -30

# Check authentication pods
oc get pods -n openshift-authentication
oc logs -n openshift-authentication <pod> | grep -i error
```
