# SECRETS

## KV v2 — versioned static secrets

```bash
# Enable KV v2 engine
vault secrets enable -path=secret kv-v2

# Write a secret
vault kv put secret/myapp/config \
  db_host=postgres db_port=5432 db_password=supersecret

# Read a secret
vault kv get secret/myapp/config
vault kv get -field=db_password secret/myapp/config

# Read as JSON
vault kv get -format=json secret/myapp/config | jq '.data.data'

# Update (adds version, keeps history)
vault kv patch secret/myapp/config db_password=newpassword

# List secrets at a path
vault kv list secret/myapp

# Get version history
vault kv metadata get secret/myapp/config

# Read a specific version
vault kv get -version=2 secret/myapp/config

# Delete (soft — creates delete marker, recoverable)
vault kv delete secret/myapp/config

# Undelete a version
vault kv undelete -versions=2 secret/myapp/config

# Permanently destroy a version
vault kv destroy -versions=1 secret/myapp/config

# Delete all versions and metadata
vault kv metadata delete secret/myapp/config
```

### KV v2 via API

```bash
# Write
curl -X POST "$VAULT_ADDR/v1/secret/data/myapp/config" \
  -H "X-Vault-Token: $VAULT_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{"data": {"db_password": "supersecret", "api_key": "abc123"}}'

# Read
curl "$VAULT_ADDR/v1/secret/data/myapp/config" \
  -H "X-Vault-Token: $VAULT_TOKEN" | jq '.data.data'
```

---

## Database engine — dynamic credentials

Vault generates short-lived credentials on demand. The app never sees a static password.

```bash
# Enable database engine
vault secrets enable database

# Configure a PostgreSQL connection
vault write database/config/mydb \
  plugin_name=postgresql-database-plugin \
  connection_url="postgresql://{{username}}:{{password}}@postgres:5432/mydb" \
  allowed_roles="app-role" \
  username="vault-admin" \
  password="admin-password"

# Create a role (defines what credentials are created)
vault write database/roles/app-role \
  db_name=mydb \
  creation_statements="CREATE ROLE \"{{name}}\" WITH LOGIN PASSWORD '{{password}}' VALID UNTIL '{{expiration}}'; GRANT SELECT, INSERT, UPDATE ON ALL TABLES IN SCHEMA public TO \"{{name}}\";" \
  default_ttl="1h" \
  max_ttl="24h"

# Request credentials
vault read database/creds/app-role
# Returns: username=v-app-role-xxxx, password=<random>, lease_duration=1h

# Renew lease
vault lease renew database/creds/app-role/<lease-id>

# Revoke immediately (e.g. after incident)
vault lease revoke database/creds/app-role/<lease-id>

# Rotate the static admin password Vault uses
vault write -force database/rotate-root/mydb
```

---

## PKI engine — certificate authority

```bash
# Enable PKI engine
vault secrets enable -path=pki pki

# Generate root CA (or import your own)
vault write pki/root/generate/internal \
  common_name="My Root CA" \
  ttl=87600h                # 10 years

# Configure CRL and issuer URLs
vault write pki/config/urls \
  issuing_certificates="$VAULT_ADDR/v1/pki/ca" \
  crl_distribution_points="$VAULT_ADDR/v1/pki/crl"

# Enable intermediate CA (recommended — don't issue from root directly)
vault secrets enable -path=pki_int pki
vault write pki_int/intermediate/generate/internal \
  common_name="My Intermediate CA" | tee /tmp/pki_int.csr

vault write pki/root/sign-intermediate \
  csr=@/tmp/pki_int.csr format=pem_bundle \
  ttl=43800h | tee /tmp/signed_int.pem

vault write pki_int/intermediate/set-signed \
  certificate=@/tmp/signed_int.pem

# Create a role (defines cert parameters)
vault write pki_int/roles/myapp \
  allowed_domains="example.com,internal.example.com" \
  allow_subdomains=true \
  max_ttl=72h

# Issue a certificate
vault write pki_int/issue/myapp \
  common_name="api.example.com" \
  ttl=24h \
  alt_names="api-v2.example.com"

# Revoke a certificate
vault write pki_int/revoke \
  serial_number="<serial>"
```

---

## Transit engine — encryption as a service

Vault encrypts/decrypts data without storing it. Your app stores ciphertext; Vault holds the key.

```bash
# Enable transit engine
vault secrets enable transit

# Create an encryption key
vault write -f transit/keys/myapp-key

# Encrypt data
vault write transit/encrypt/myapp-key \
  plaintext=$(echo "my sensitive data" | base64)
# Returns: ciphertext=vault:v1:abc123...

# Decrypt data
vault write transit/decrypt/myapp-key \
  ciphertext="vault:v1:abc123..."
# Returns: plaintext (base64) → decode to get original

# Rotate the key (old ciphertext still decryptable — rewrap for new key version)
vault write -f transit/keys/myapp-key/rotate

# Rewrap ciphertext with latest key version
vault write transit/rewrap/myapp-key \
  ciphertext="vault:v1:abc123..."
```

---

## AWS secrets engine

```bash
# Enable AWS engine
vault secrets enable aws

# Configure with IAM credentials
vault write aws/config/root \
  access_key=AKIAIOSFODNN7EXAMPLE \
  secret_key=wJalrXUtnFEMI/K7MDENG/bPxRfiCYEXAMPLEKEY \
  region=eu-west-1

# Create a role (Vault-managed IAM user)
vault write aws/roles/my-role \
  credential_type=iam_user \
  policy_document='{
    "Version":"2012-10-17",
    "Statement":[{"Effect":"Allow","Action":["s3:GetObject"],"Resource":"arn:aws:s3:::my-bucket/*"}]
  }'

# Request temporary AWS credentials
vault read aws/creds/my-role
# Returns: access_key, secret_key, lease_duration=768h
```
