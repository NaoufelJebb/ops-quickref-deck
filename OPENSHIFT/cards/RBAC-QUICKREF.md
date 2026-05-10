# RBAC QUICK REFERENCE

## Built-in roles

| Role | Access |
|---|---|
| `cluster-admin` | Full cluster — use sparingly |
| `admin` | Full within a project |
| `edit` | Create/update/delete resources, not RBAC |
| `view` | Read-only within a project |
| `self-provisioner` | Create new projects |
| `basic-user` | See own identity, create projects |

## Assign roles

```bash
# Project-scoped
oc adm policy add-role-to-user admin    <user>  -n <project>
oc adm policy add-role-to-user edit     <user>  -n <project>
oc adm policy add-role-to-user view     <user>  -n <project>
oc adm policy add-role-to-group edit    <group> -n <project>

# Cluster-scoped
oc adm policy add-cluster-role-to-user cluster-admin <user>
oc adm policy add-cluster-role-to-group view <group>

# Service account (-z = service account in namespace)
oc adm policy add-role-to-user edit -z <sa> -n <project>

# Remove
oc adm policy remove-role-from-user edit <user> -n <project>
oc adm policy remove-cluster-role-from-user cluster-admin <user>
```

## Check access

```bash
# List role bindings in a project
oc get rolebindings -n <project>
oc describe rolebinding admin -n <project>

# List cluster role bindings
oc get clusterrolebindings | grep <user>

# Can a user do something?
oc auth can-i create pods -n <project> --as=<user>
oc auth can-i '*' '*' --as=<user>      # check cluster-admin

# What can the current user do?
oc auth can-i --list -n <project>
```

## SCCs — quick grants

```bash
# Check pod's SCC
oc get pod <pod> -n <ns> \
  -o jsonpath='{.metadata.annotations.openshift\.io/scc}'

# Grant (service account)
oc adm policy add-scc-to-user nonroot    -z <sa> -n <ns>
oc adm policy add-scc-to-user anyuid     -z <sa> -n <ns>
oc adm policy add-scc-to-user privileged -z <sa> -n <ns>

# Grant to a user
oc adm policy add-scc-to-user anyuid <user>

# Remove
oc adm policy remove-scc-from-user anyuid -z <sa> -n <ns>

# Simulate admission
oc adm policy scc-review -f pod.yaml
oc adm policy review -z <sa> -n <ns>
```

## SCC hierarchy (most → least restrictive)

```
restricted-v2 → restricted → nonroot-v2 → nonroot → anyuid → privileged
```

Try the most restrictive first. Grant up only as needed.

## LDAP group → role (after group sync)

```bash
# Assign cluster role to synced LDAP group
oc adm policy add-cluster-role-to-group cluster-admin ocp-admins
oc adm policy add-role-to-group edit ocp-developers -n my-app
oc adm policy add-role-to-group view ocp-viewers -n my-app
```
