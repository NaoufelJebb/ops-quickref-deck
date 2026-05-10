# CLIENT SETUP

## Which client type?

```
User logs in via browser (SPA, web app)?   → Public + Auth Code + PKCE
Backend web app with server-side session?  → Confidential + Auth Code
Service / daemon / CI-CD?                  → Confidential + Client Credentials
API that only validates tokens?            → Bearer-only (no login flow)
```

## Public client (SPA / mobile) — minimum config

```
Client ID:              my-spa
Client authentication:  Off (public)
Standard flow:          On
Direct access grants:   Off
Valid redirect URIs:    https://app.example.com/*
Web origins:            https://app.example.com
PKCE:                   S256 (Advanced tab → Code Challenge Method)
```

## Confidential client (backend / service) — minimum config

```
Client ID:              my-service
Client authentication:  On
Standard flow:          Off (if API only)
Service accounts:       On (if using client credentials)
Valid redirect URIs:    (empty if service account only)
```

Client secret: `Clients → my-service → Credentials tab`

## Token endpoint — client credentials

```bash
curl -X POST "$KC/realms/$REALM/protocol/openid-connect/token" \
  -d "grant_type=client_credentials" \
  -d "client_id=my-service" \
  -d "client_secret=$SECRET" | jq '{access_token, expires_in}'
```

## Add a custom claim to all tokens for a client

1. `Clients → [client] → Client scopes → [client-name]-dedicated → Mappers → Add mapper`
2. Choose mapper type:
   - `User Attribute` — map a user attribute to a claim
   - `User Realm Role` — add realm roles to token
   - `Hardcoded claim` — always add a static value
   - `Audience` — add `aud` claim for API validation

## Discovery URL (give this to your developers)

```
https://keycloak/realms/{realm}/.well-known/openid-configuration
```

## OIDC endpoints quick reference

```
Auth:        $KC/realms/$REALM/protocol/openid-connect/auth
Token:       $KC/realms/$REALM/protocol/openid-connect/token
Userinfo:    $KC/realms/$REALM/protocol/openid-connect/userinfo
Logout:      $KC/realms/$REALM/protocol/openid-connect/logout
JWKS:        $KC/realms/$REALM/protocol/openid-connect/certs
Introspect:  $KC/realms/$REALM/protocol/openid-connect/token/introspect
```
