# TOKENS

## Token endpoints

```bash
# Discovery — all endpoints for a realm
GET https://keycloak/realms/{realm}/.well-known/openid-configuration

# JWKS — public keys for token validation
GET https://keycloak/realms/{realm}/protocol/openid-connect/certs

# Token endpoint — get tokens
POST https://keycloak/realms/{realm}/protocol/openid-connect/token

# Userinfo — get claims for an access token
GET https://keycloak/realms/{realm}/protocol/openid-connect/userinfo
  Authorization: Bearer <access_token>

# Introspect — validate a token and get its metadata
POST https://keycloak/realms/{realm}/protocol/openid-connect/token/introspect
  client_id=my-client&client_secret=xxx&token=<token>

# Logout — end session
POST https://keycloak/realms/{realm}/protocol/openid-connect/logout
  client_id=my-client&refresh_token=<refresh_token>
```

---

## Get tokens — common grant types

### Authorization Code (web app — backend handles code exchange)

```bash
# Step 1: redirect user to:
GET https://keycloak/realms/{realm}/protocol/openid-connect/auth
  ?client_id=my-client
  &redirect_uri=https://app.example.com/callback
  &response_type=code
  &scope=openid profile email
  &state=<random>
  &code_challenge=<pkce-challenge>
  &code_challenge_method=S256

# Step 2: exchange code for tokens
curl -X POST https://keycloak/realms/{realm}/protocol/openid-connect/token \
  -d "grant_type=authorization_code" \
  -d "client_id=my-client" \
  -d "client_secret=my-secret" \
  -d "code=<auth-code>" \
  -d "redirect_uri=https://app.example.com/callback" \
  -d "code_verifier=<pkce-verifier>"
```

### Client Credentials (machine-to-machine)

```bash
curl -X POST https://keycloak/realms/{realm}/protocol/openid-connect/token \
  -d "grant_type=client_credentials" \
  -d "client_id=my-service" \
  -d "client_secret=my-secret"
```

### Refresh token

```bash
curl -X POST https://keycloak/realms/{realm}/protocol/openid-connect/token \
  -d "grant_type=refresh_token" \
  -d "client_id=my-client" \
  -d "client_secret=my-secret" \
  -d "refresh_token=<refresh_token>"
```

---

## Validate a token (API side)

APIs should validate access tokens locally using the JWKS — no network call per request.

### JWT structure

```
header.payload.signature

Decode payload (base64):
{
  "exp": 1704070800,          ← expiry — check this
  "iat": 1704067200,          ← issued at
  "iss": "https://keycloak/realms/myrealm",  ← issuer — verify this
  "aud": "my-api",            ← audience — verify this
  "sub": "user-uuid",         ← subject (user ID)
  "preferred_username": "john.doe",
  "email": "john@example.com",
  "realm_access": {
    "roles": ["app-admin"]
  },
  "resource_access": {
    "my-client": {
      "roles": ["read"]
    }
  }
}
```

### Validation checklist (your API must check all of these)

- [ ] Signature valid (verify with JWKS public key)
- [ ] `exp` not in the past
- [ ] `iss` matches your Keycloak realm URL
- [ ] `aud` contains your client/API identifier
- [ ] `nbf` (if present) is not in the future
- [ ] Token type is `Bearer` access token (not refresh or ID token)

### Quick local decode (debugging only)

```bash
# Decode payload without verification (debug only)
echo "<jwt-payload-part>" | base64 -d | jq .

# Full decode with verification via curl introspect
curl -X POST https://keycloak/realms/{realm}/protocol/openid-connect/token/introspect \
  -u "my-client:my-secret" \
  -d "token=<access_token>"
# Returns: {"active": true, ...claims...}
# active: false = invalid/expired token
```

---

## Token exchange (service-to-service impersonation)

Exchange a user's token for a token scoped to a different service/realm.

```bash
curl -X POST https://keycloak/realms/{realm}/protocol/openid-connect/token \
  -d "grant_type=urn:ietf:params:oauth:grant-type:token-exchange" \
  -d "client_id=service-a" \
  -d "client_secret=secret" \
  -d "subject_token=<user-access-token>" \
  -d "subject_token_type=urn:ietf:params:oauth:token-type:access_token" \
  -d "requested_token_type=urn:ietf:params:oauth:token-type:access_token" \
  -d "audience=service-b"
```

Enable: `Realm settings → General → Token exchange policy`

---

## Offline tokens (long-lived refresh)

Request an offline token by including `offline_access` scope:

```bash
curl -X POST https://keycloak/realms/{realm}/protocol/openid-connect/token \
  -d "grant_type=authorization_code" \
  -d "client_id=my-client" \
  -d "scope=openid offline_access" \
  -d "code=<code>" \
  -d "redirect_uri=..."
```

Offline tokens survive server restarts and don't expire until explicitly revoked or the offline session max is reached. Revoke via:

```bash
# Revoke all offline sessions for a user
curl -X DELETE "https://keycloak/admin/realms/myrealm/users/<user-id>/offline-sessions/<client-id>" \
  -H "Authorization: Bearer $ADMIN_TOKEN"
```
