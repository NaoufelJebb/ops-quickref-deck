# IDENTITY PROVIDERS

## What identity providers are

An Identity Provider (IdP) lets Keycloak delegate authentication to an external system — another OIDC provider, a SAML IdP, or a social login. Users log in via the IdP; Keycloak issues its own tokens.

```
User → Keycloak → redirects to external IdP → IdP authenticates → returns to Keycloak → Keycloak issues token
```

## Add an OIDC IdP

`Realm → Identity Providers → Add provider → OpenID Connect v1.0`

```
Alias:                  external-sso
Display name:           Corporate SSO
Discovery endpoint:     https://corporate-sso.example.com/realms/corp/.well-known/openid-configuration

Client ID:              keycloak-federation
Client secret:          <from the upstream IdP>
Default scopes:         openid profile email

Sync mode:              Force  (always update user from IdP token)
                     or Import (only on first login)

Trust Email:            On  (if upstream IdP verifies emails)
Store tokens:           Off (unless you need upstream tokens)
```

## Add a SAML IdP

`Realm → Identity Providers → Add provider → SAML v2.0`

```
Alias:                      saml-corporate
Display name:               Corporate SAML SSO
Service provider entity ID: https://keycloak/realms/myrealm
Single sign-on service URL: https://idp.corporate.com/sso/saml
Single logout service URL:  https://idp.corporate.com/slo/saml

NameID policy format:       Persistent
Principal type:             Subject NameID

Want AuthnRequests signed:  On
Want assertions signed:     On
Want assertions encrypted:  Off (enable if supported)

Validating X.509 certificate: <paste IdP cert>
```

Export Keycloak's SAML SP metadata (to give to the upstream IdP):
`Identity Providers → [provider] → Export SP metadata`

## Social logins

`Realm → Identity Providers → Add provider → [Google / GitHub / Microsoft / Apple / Facebook…]`

### Google

```
Client ID:     <from Google Cloud Console>
Client secret: <from Google Cloud Console>
Default scopes: openid profile email
```

Google Console setup: `APIs & Services → Credentials → OAuth 2.0 Client IDs`
- Authorized redirect URI: `https://keycloak/realms/{realm}/broker/google/endpoint`

### Microsoft (Entra ID / Azure AD)

```
Alias:              microsoft
Client ID:          <Azure App Registration client ID>
Client secret:      <client secret>
Discovery endpoint: https://login.microsoftonline.com/<tenant-id>/v2.0/.well-known/openid-configuration
Default scopes:     openid profile email
```

Azure App Registration redirect URI:
`https://keycloak/realms/{realm}/broker/microsoft/endpoint`

### GitHub

```
Client ID:          <GitHub OAuth App client ID>
Client secret:      <GitHub OAuth App client secret>
Default scopes:     user:email
```

GitHub OAuth App callback URL:
`https://keycloak/realms/{realm}/broker/github/endpoint`

---

## First broker login flow

Controls what happens when a user logs in via IdP for the first time.

`Identity Providers → [provider] → First login flow: first broker login`

Default behavior:
1. Review profile (confirm/update email, name)
2. Check for existing account with same email → link it or create new

Customize to:
- Auto-link accounts with same verified email (skip review if trust email is on)
- Require additional verification
- Auto-create accounts without review

---

## IdP mappers — import claims from upstream

`Identity Providers → [provider] → Mappers → Add mapper`

### Import a claim as a user attribute

```
Mapper type:           Attribute Importer
Name:                  department
Social Profile JSON Field Path: department
User Attribute:        department
```

### Import a claim and map to a role

```
Mapper type:           Role Importer
Claim:                 groups
Claim Value:           admins
Role:                  app-admin
```

### Hardcode a role for all IdP users

```
Mapper type:           Hardcoded Role
Role:                  external-user
```

---

## Account linking

Allow users to link multiple IdP accounts to one Keycloak account.

`Realm settings → Login tab → Link with broker`

Or trigger via the Account Console: `https://keycloak/realms/{realm}/account`

---

## Keycloak as a SAML SP to another Keycloak (realm chaining)

Useful for multi-realm federation:

`Realm B → Identity Providers → OpenID Connect → point to Realm A`

Realm A acts as the upstream IdP. Users from Realm A can log in to applications in Realm B.
