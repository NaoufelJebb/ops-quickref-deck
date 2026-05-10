# SECURITY

## Realm hardening checklist

- [ ] `master` realm used only for Keycloak administration — no application clients
- [ ] `master` realm admin account uses strong password + MFA
- [ ] Brute force protection enabled: `Realm settings → Security defenses`
- [ ] SSL required: `Realm settings → General → Require SSL: All requests`
- [ ] Registration disabled unless needed: `Realm settings → Login → User registration: Off`
- [ ] Email as username disabled unless intentional
- [ ] Forgot password flow enabled only with verified email
- [ ] Admin events enabled and retained: `Realm settings → Events`
- [ ] Login events enabled and retained

## Token settings

```bash
# Recommended token lifetimes (Realm settings → Tokens)
Access token lifespan:           300s    (5 min)
SSO session idle:                1800s   (30 min)
SSO session max:                 36000s  (10 hours)
Offline session idle:            2592000 (30 days)
Offline session max enabled:     On
Offline session max:             5184000 (60 days)
```

Short access tokens reduce the window of token misuse. Always use refresh tokens.

## Client hardening

```bash
# Public clients (SPAs, mobile)
Direct access grants: Off      # disable Password grant
Implicit flow:        Off      # deprecated
PKCE:                 Required # enforce S256

# Confidential clients
Rotate client secret regularly
Service accounts: assign minimum required roles only
Valid redirect URIs: exact URLs only, no wildcards in production
Web origins: explicit list, not "+"
```

## Disable the Password grant globally

`Realm settings → Client policies → Add policy`

Create a policy that enforces no `direct_access_grants` on all clients, or review each client manually and disable it.

## Secrets management

```bash
# Never hardcode secrets in config files — use env vars or a vault
# keycloak.conf example:
db-password=${KC_DB_PASSWORD}

# In K8s, use Secrets (ideally managed by Vault / External Secrets Operator)
```

## CORS configuration

Configure per-client via `Web Origins`:

```
Specific origins: https://app.example.com
All origins:      +  (use only in dev — avoid in prod)
```

CORS preflight is handled automatically by Keycloak when `Web Origins` is set.

## Content Security Policy

`Realm settings → Security defenses → Headers`

```
X-Frame-Options:   SAMEORIGIN
Content-Security-Policy: frame-src 'self'; object-src 'none';
X-Robots-Tag:      none
X-Content-Type-Options: nosniff
X-XSS-Protection:  1; mode=block
Strict-Transport-Security: max-age=31536000; includeSubDomains
```

## Network hardening

```bash
# Bind admin port only to internal network (if separate admin port is supported)
# Use a reverse proxy that only exposes /realms/* and /resources/*
# Block direct access to /admin from external networks at the proxy level:

# Nginx example — block /admin from non-internal IPs
location /admin {
    allow 10.0.0.0/8;
    allow 172.16.0.0/12;
    deny all;
    proxy_pass http://keycloak:8080;
}
```

## Signing key rotation

Rotate realm signing keys annually or on suspected compromise:

1. Add a new RSA key provider with higher priority (see OPS.md → Cert/key rotation)
2. New key becomes active immediately — new tokens use it
3. Old key stays enabled for validation until existing tokens expire (max = access token lifespan)
4. Disable old key after tokens expire

JWKS endpoint (`/realms/{realm}/protocol/openid-connect/certs`) automatically serves all enabled keys — clients validate against any of them.

## Audit trail

Enable both event types:
`Realm settings → Events`

```
Save login events:  On
Save admin events:  On
Expiration:         90 days (minimum for most compliance requirements)
```

Events to monitor:
- `LOGIN_ERROR` with `error=invalid_client_credentials` → credential stuffing
- `BRUTE_FORCE_FAILURE` → account under attack
- Admin events: `CREATE_CLIENT`, `UPDATE_REALM` → unexpected config changes
