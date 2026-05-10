# CONCEPTS

## What Keycloak is

Keycloak is an open-source Identity and Access Management (IAM) platform. It provides authentication, authorization, SSO, and federation out of the box — so your applications delegate all identity concerns to Keycloak instead of implementing them individually.

```
Browser / App → [ Keycloak ] → issues tokens → App validates token
                     ↑
              LDAP / AD / SAML / Social (federated identities)
```

## Core objects

| Object | What it is |
|---|---|
| **Realm** | Isolated tenant — users, clients, and config are scoped to a realm |
| **Client** | An application that delegates auth to Keycloak (web app, API, CLI) |
| **User** | An identity — local or federated (LDAP, AD, social) |
| **Group** | A collection of users — assign roles to groups |
| **Role** | A permission label — realm-level or client-level |
| **Scope** | An OAuth2 scope — controls what claims appear in tokens |
| **Identity Provider (IdP)** | An external auth source (SAML, OIDC, Google, GitHub) |
| **User Federation** | A directory sync source (LDAP, Active Directory) |
| **Flow** | An authentication pipeline — sequences of steps and checks |
| **Session** | An active login — SSO session shared across clients |

## Protocol support

| Protocol | Use for |
|---|---|
| **OpenID Connect (OIDC)** | Modern web apps, APIs, mobile — JWT-based |
| **OAuth 2.0** | API authorization, machine-to-machine |
| **SAML 2.0** | Enterprise SSO with legacy systems |
| **Kerberos** | Windows/AD integrated auth (via LDAP federation) |

## Token types

| Token | Lifetime | Use |
|---|---|---|
| **Access token** | Short (minutes) | Sent to APIs in `Authorization: Bearer` header |
| **Refresh token** | Longer (hours/days) | Exchanges for new access tokens without re-login |
| **ID token** | Short | Contains user identity claims for the client |
| **Offline token** | Very long / no expiry | Long-lived refresh token for background jobs |

## Realm hierarchy

```
Master realm (admin only — manage other realms)
├── Realm A (your app)
│   ├── Clients (your-frontend, your-api, your-mobile)
│   ├── Users
│   ├── Groups
│   ├── Roles
│   └── Identity Providers / User Federation
└── Realm B (another tenant)
```

Never use the `master` realm for your applications. Create a dedicated realm per environment or tenant.

## OAuth2 grant types — which to use

| Grant type | Use for |
|---|---|
| **Authorization Code + PKCE** | Web apps, SPAs, mobile apps — always prefer this |
| **Client Credentials** | Machine-to-machine, service accounts, CI/CD |
| **Device Authorization** | CLI tools, smart TVs |
| **Refresh Token** | Renew access tokens silently |
| ~~Password grant~~ | **Avoid** — deprecated, bypasses MFA |
| ~~Implicit~~ | **Avoid** — deprecated |
