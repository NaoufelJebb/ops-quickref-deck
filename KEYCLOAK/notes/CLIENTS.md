# CLIENTS

## Client types

| Type | Access type | Use for |
|---|---|---|
| **Public** | No secret | SPAs, mobile apps — cannot keep a secret |
| **Confidential** | Client secret or mTLS | Web apps with backend, APIs |
| **Bearer-only** | No login flow | APIs that only validate tokens, never initiate auth |

## Create a client — web app (Authorization Code + PKCE)

`Realm → Clients → Create client`

```
Client ID:              my-frontend
Client type:            OpenID Connect
Client authentication:  Off (public)

Settings tab:
  Valid redirect URIs:  https://app.example.com/*
                        http://localhost:3000/*  (dev only)
  Valid post logout redirect URIs: https://app.example.com/
  Web origins:          https://app.example.com  (CORS)

Capability config:
  Standard flow:        On   (Authorization Code)
  Direct access grants: Off  (never enable Password grant in prod)
  Implicit flow:        Off  (deprecated)
```

## Create a client — API / backend (Client Credentials)

```
Client ID:              my-api-service
Client type:            OpenID Connect
Client authentication:  On (confidential)

Settings tab:
  Valid redirect URIs:  (leave empty — not used for client credentials)

Capability config:
  Standard flow:        Off
  Service accounts:     On  (enables client credentials grant)
```

Get the client secret: `Clients → my-api-service → Credentials tab`

## Client scopes

Scopes control what claims appear in the access and ID tokens.

| Scope | Claims added |
|---|---|
| `openid` | Required for OIDC — adds `sub`, `iss`, `aud` |
| `profile` | `name`, `given_name`, `family_name`, `preferred_username` |
| `email` | `email`, `email_verified` |
| `roles` | `realm_access.roles`, `resource_access.<client>.roles` |
| `address` | Address claims |
| `phone` | Phone number |
| `offline_access` | Issues offline (long-lived) refresh tokens |
| `microprofile-jwt` | Adds `groups` claim from roles |

### Add a custom claim to tokens

`Realm → Client scopes → Create client scope`

Then add a mapper:

```
Mapper type:         User Attribute
Name:                department
User Attribute:      department
Token Claim Name:    department
Claim JSON Type:     String
Add to ID token:     On
Add to access token: On
Add to userinfo:     On
```

Or map a role to a claim:

```
Mapper type:         User Realm Role
Token Claim Name:    roles
Multivalued:         On
Add to access token: On
```

## Assign scopes to a client

`Clients → [client] → Client scopes tab`

- **Default scopes** — always included in tokens
- **Optional scopes** — included only when the client requests them

## Service account roles

For clients with service accounts (Client Credentials grant), assign roles to the service account:

`Clients → [client] → Service accounts roles tab → Assign role`

---

## PKCE configuration (SPAs / mobile)

PKCE is automatic for public clients in Keycloak 20+. To enforce it:

`Clients → [client] → Advanced tab → Proof Key for Code Exchange Code Challenge Method: S256`

---

## Token lifetimes (per client override)

`Clients → [client] → Advanced tab → Token settings`

| Setting | Default | Override for |
|---|---|---|
| Access token lifespan | 5 min | Extend for less-frequent refresh scenarios |
| Client session idle | 30 min | How long before idle session expires |
| Client session max | 10 hours | Absolute session max |
| Offline session idle | 30 days | Offline token expiry |

Global defaults: `Realm settings → Tokens tab`

---

## Audience mapper (API clients)

When an API needs to validate that a token was issued for it, add an audience mapper:

`Clients → [client] → Client scopes → [dedicated scope] → Mappers → Add mapper → Audience`

```
Name:           my-api-audience
Included Client Audience: my-api-service
Add to access token: On
```

The API validates: `aud` claim contains `my-api-service`.

---

## Useful OIDC endpoints

All discoverable at: `https://keycloak/realms/{realm}/.well-known/openid-configuration`

```
Authorization:   https://keycloak/realms/{realm}/protocol/openid-connect/auth
Token:           https://keycloak/realms/{realm}/protocol/openid-connect/token
Userinfo:        https://keycloak/realms/{realm}/protocol/openid-connect/userinfo
Logout:          https://keycloak/realms/{realm}/protocol/openid-connect/logout
JWKS:            https://keycloak/realms/{realm}/protocol/openid-connect/certs
Introspect:      https://keycloak/realms/{realm}/protocol/openid-connect/token/introspect
```
