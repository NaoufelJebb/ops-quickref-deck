# AUTH AND POLICIES

## Policies

Policies are attached to tokens and define what paths they can access.

```bash
# Write a policy from file
vault policy write myapp-policy myapp-policy.hcl

# Write inline
vault policy write myapp-policy - <<EOF
path "secret/data/myapp/*" {
  capabilities = ["read", "list"]
}
path "database/creds/app-role" {
  capabilities = ["read"]
}
path "auth/token/renew-self" {
  capabilities = ["update"]
}
EOF

# List policies
vault policy list

# Read a policy
vault policy read myapp-policy

# Delete
vault policy delete myapp-policy
```

---

## Token auth

```bash
# Create a token with a policy
vault token create \
  -policy=myapp-policy \
  -ttl=24h \
  -renewable=true \
  -display-name="myapp-prod"

# Create a token with multiple policies
vault token create -policy=myapp-policy -policy=pki-read

# Lookup current token
vault token lookup

# Lookup another token
vault token lookup <token>

# Renew token
vault token renew
vault token renew -increment=12h

# Revoke a token (and all children)
vault token revoke <token>

# Revoke all tokens with a specific policy (emergency)
vault token revoke -mode=path auth/token/create
```

---

## AppRole — machine-to-machine auth

AppRole is the standard auth method for services, CI/CD, and automation.

```bash
# Enable AppRole
vault auth enable approle

# Create a role
vault write auth/approle/role/myapp \
  token_policies="myapp-policy" \
  token_ttl=1h \
  token_max_ttl=4h \
  secret_id_ttl=24h \
  secret_id_num_uses=0        # 0 = unlimited uses

# Get the RoleID (semi-public — include in app config)
vault read auth/approle/role/myapp/role-id

# Generate a SecretID (treat like a password — inject at deploy time)
vault write -f auth/approle/role/myapp/secret-id

# Authenticate
vault write auth/approle/login \
  role_id=<role-id> \
  secret_id=<secret-id>
# Returns: token + token_policies + lease_duration

# Via API
curl -X POST "$VAULT_ADDR/v1/auth/approle/login" \
  -d '{"role_id":"<role-id>","secret_id":"<secret-id>"}' \
  | jq '.auth.client_token'
```

---

## Kubernetes auth

Pods authenticate using their service account JWT — no secrets to manage.

```bash
# Enable Kubernetes auth
vault auth enable kubernetes

# Configure (run from inside the cluster or provide values manually)
vault write auth/kubernetes/config \
  kubernetes_host=https://kubernetes.default.svc \
  kubernetes_ca_cert=@/var/run/secrets/kubernetes.io/serviceaccount/ca.crt \
  token_reviewer_jwt=@/var/run/secrets/kubernetes.io/serviceaccount/token

# Create a role — maps K8s service account to Vault policy
vault write auth/kubernetes/role/myapp \
  bound_service_account_names=myapp-sa \
  bound_service_account_namespaces=my-namespace \
  token_policies=myapp-policy \
  token_ttl=1h

# Authenticate (done by the app / Vault Agent)
curl -X POST "$VAULT_ADDR/v1/auth/kubernetes/login" \
  -d "{
    \"role\": \"myapp\",
    \"jwt\": \"$(cat /var/run/secrets/kubernetes.io/serviceaccount/token)\"
  }" | jq '.auth.client_token'
```

---

## LDAP / Active Directory auth

```bash
# Enable LDAP auth
vault auth enable ldap

# Configure AD
vault write auth/ldap/config \
  url="ldaps://dc1.corp.local:636" \
  binddn="CN=svc-vault,OU=Service Accounts,DC=corp,DC=local" \
  bindpass="service-account-password" \
  userdn="OU=Users,DC=corp,DC=local" \
  userattr="sAMAccountName" \
  groupdn="OU=Groups,DC=corp,DC=local" \
  groupattr="cn" \
  groupfilter="(&(objectClass=group)(member={{.UserDN}}))" \
  certificate=@/path/to/ldap-ca.crt \
  insecure_tls=false \
  starttls=false

# Map an AD group to a Vault policy
vault write auth/ldap/groups/vault-admins \
  policies=admin

vault write auth/ldap/groups/vault-readers \
  policies=readonly

# Map a specific user to a policy
vault write auth/ldap/users/john.doe \
  policies=myapp-policy

# Login
vault login -method=ldap username=john.doe
```

---

## OIDC auth (Keycloak, Okta, GitHub)

```bash
# Enable OIDC
vault auth enable oidc

# Configure
vault write auth/oidc/config \
  oidc_discovery_url="https://keycloak.example.com/realms/myrealm" \
  oidc_client_id="vault" \
  oidc_client_secret="client-secret" \
  default_role="default"

# Create a role
vault write auth/oidc/role/default \
  bound_audiences="vault" \
  user_claim="sub" \
  groups_claim="groups" \
  allowed_redirect_uris="https://vault.example.com/ui/vault/auth/oidc/oidc/callback" \
  allowed_redirect_uris="http://localhost:8250/oidc/callback" \
  token_policies="default" \
  token_ttl=1h

# Login via browser
vault login -method=oidc
```

---

## AWS auth (EC2 / IAM)

```bash
# Enable AWS auth
vault auth enable aws

vault write auth/aws/config/client \
  access_key=AKIAIOSFODNN7EXAMPLE \
  secret_key=wJalrXUtnFEMI/K7MDENG

# Bind an IAM role to a Vault policy
vault write auth/aws/role/myapp \
  auth_type=iam \
  bound_iam_principal_arn="arn:aws:iam::123456789:role/myapp-role" \
  token_policies=myapp-policy \
  token_ttl=1h
```

---

## Token hierarchy and orphan tokens

```bash
# Create an orphan token (not a child — survives parent revocation)
vault token create -orphan -policy=myapp-policy

# Create a periodic token (renews indefinitely — for long-running services)
vault token create \
  -policy=myapp-policy \
  -period=24h \
  -orphan

# Check token capabilities on a path
vault token capabilities secret/data/myapp/config
```
