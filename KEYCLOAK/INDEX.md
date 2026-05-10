# KEYCLOAK NOTES — INDEX

## Notes (deep reference)

| File | Open when you need to... |
|---|---|
| [CONCEPTS.md](./notes/CONCEPTS.md) | Understand realms, clients, roles, flows, token types, grant types |
| [ARCHITECTURE.md](./notes/ARCHITECTURE.md) | Deployment modes, HA setup, DB config, reverse proxy, Docker Compose |
| [LDAP-AD.md](./notes/LDAP-AD.md) | LDAP/Active Directory federation, mappers, sync, Kerberos, LDAPS |
| [CLIENTS.md](./notes/CLIENTS.md) | Create and configure clients, scopes, PKCE, service accounts, OIDC endpoints |
| [ROLES-AND-GROUPS.md](./notes/ROLES-AND-GROUPS.md) | Realm/client roles, groups, composites, fine-grained authorization |
| [FLOWS.md](./notes/FLOWS.md) | Authentication flows, MFA, WebAuthn, step-up auth, brute force protection |
| [IDENTITY-PROVIDERS.md](./notes/IDENTITY-PROVIDERS.md) | SAML, OIDC, Google, Microsoft, GitHub, IdP mappers, account linking |
| [TOKENS.md](./notes/TOKENS.md) | Token validation, grant types, introspection, JWKS, token exchange |
| [KUBERNETES.md](./notes/KUBERNETES.md) | Operator, Keycloak CR, KeycloakRealmImport, Ingress, kubectl commands |
| [API.md](./notes/API.md) | Admin REST API — users, roles, groups, sessions, realm operations |
| [TROUBLESHOOTING.md](./notes/TROUBLESHOOTING.md) | Login failures, LDAP issues, MFA, token errors, log patterns |
| [OPS.md](./notes/OPS.md) | Day-2 ops, export/import, key rotation, monitoring, audit |
| [SECURITY.md](./notes/SECURITY.md) | Hardening checklist, token settings, CORS, CSP, audit trail |

## Cards (fast access — open these daily)

| File | Open when you need to... |
|---|---|
| [CHEATSHEET.md](./cards/CHEATSHEET.md) | Any Admin API call or UI navigation path |
| [LDAP-QUICKREF.md](./cards/LDAP-QUICKREF.md) | AD vs OpenLDAP field mapping, LDAP filters, bind DN formats |
| [TOKEN-DEBUG.md](./cards/TOKEN-DEBUG.md) | Decode a token, validate claims, fix common token errors |
| [INCIDENT.md](./cards/INCIDENT.md) | Login outage, account compromise, brute force, key rotation |
| [CLIENT-SETUP.md](./cards/CLIENT-SETUP.md) | Create a client fast — public, confidential, service account |
| [USER-OPS.md](./cards/USER-OPS.md) | Create, disable, reset, unlock, assign roles — all user operations |
| [AD-GROUPS.md](./cards/AD-GROUPS.md) | Map Active Directory groups into Keycloak groups or roles |

---

## Situation → file map

| I'm doing... | Open... |
|---|---|
| First day, learning Keycloak | [CONCEPTS.md](./notes/CONCEPTS.md) → [ARCHITECTURE.md](./notes/ARCHITECTURE.md) |
| Setting up Keycloak for the first time | [ARCHITECTURE.md](./notes/ARCHITECTURE.md) |
| Connecting LDAP / Active Directory | [LDAP-AD.md](./notes/LDAP-AD.md) + [LDAP-QUICKREF.md](./cards/LDAP-QUICKREF.md) |
| LDAP users not syncing | [LDAP-AD.md](./notes/LDAP-AD.md) → Troubleshooting section + [TROUBLESHOOTING.md](./notes/TROUBLESHOOTING.md) |
| Creating a new client (app or API) | [CLIENT-SETUP.md](./cards/CLIENT-SETUP.md) + [CLIENTS.md](./notes/CLIENTS.md) |
| Setting up SSO with Google / Microsoft | [IDENTITY-PROVIDERS.md](./notes/IDENTITY-PROVIDERS.md) |
| Setting up SAML with a corporate IdP | [IDENTITY-PROVIDERS.md](./notes/IDENTITY-PROVIDERS.md) |
| Adding MFA / WebAuthn | [FLOWS.md](./notes/FLOWS.md) |
| Token validation failing in my API | [TOKEN-DEBUG.md](./cards/TOKEN-DEBUG.md) + [TOKENS.md](./notes/TOKENS.md) |
| Users getting wrong roles in token | [ROLES-AND-GROUPS.md](./notes/ROLES-AND-GROUPS.md) + [TOKEN-DEBUG.md](./cards/TOKEN-DEBUG.md) |
| Create / disable / reset a user | [USER-OPS.md](./cards/USER-OPS.md) |
| User locked out (brute force) | [USER-OPS.md](./cards/USER-OPS.md) → unlock section |
| Incident — users can't log in | [INCIDENT.md](./cards/INCIDENT.md) |
| Incident — account compromised | [INCIDENT.md](./cards/INCIDENT.md) |
| Rotating signing keys | [OPS.md](./notes/OPS.md) → Cert/key rotation |
| Deploying on Kubernetes | [KUBERNETES.md](./notes/KUBERNETES.md) |
| Exporting / importing realm config | [OPS.md](./notes/OPS.md) → Realm export/import |
| Any Admin API call | [CHEATSHEET.md](./cards/CHEATSHEET.md) + [API.md](./notes/API.md) |
| Hardening a new environment | [SECURITY.md](./notes/SECURITY.md) |
| Setting up monitoring | [OPS.md](./notes/OPS.md) → Monitoring section |
