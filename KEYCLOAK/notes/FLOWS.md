# AUTHENTICATION FLOWS

## What flows are

Flows are configurable pipelines of authentication steps. Keycloak ships with built-in flows you copy and customize — never edit the built-in flows directly.

`Realm → Authentication → Flows`

## Built-in flows

| Flow | Used for |
|---|---|
| `browser` | Standard web login |
| `direct grant` | Resource Owner Password grant (avoid) |
| `registration` | User self-registration |
| `reset credentials` | Password reset |
| `clients` | Client authentication |
| `first broker login` | First login via identity provider |

## Add MFA (TOTP) to browser flow

1. `Authentication → Flows → browser → Copy` → name it `browser-mfa`
2. In the copy, find `Browser - Conditional OTP`
3. Set it to `Required` (always) or keep `Conditional` (policy-driven)
4. Bind the flow: `Authentication → Bindings → Browser flow → browser-mfa`

### Conditional OTP policy

`Realm settings → OTP Policy`

```
OTP type:          TOTP (time-based)
OTP algorithm:     SHA1
Number of digits:  6
Look ahead window: 1
OTP token period:  30 seconds

Supported apps: Google Authenticator, FreeOTP, Authy
```

## Passwordless (WebAuthn)

`Authentication → Required Actions → Register → WebAuthn Register`

Flow setup:

1. Copy `browser` flow → `browser-webauthn`
2. Replace `Username Password Form` with `WebAuthn Authenticator`
3. Set `WebAuthn Authenticator` as `Required`
4. Bind as browser flow

WebAuthn policy: `Realm settings → WebAuthn Policy`

```
Relying Party Name:           My App
Relying Party ID:             example.com
Signature Algorithms:         ES256
Attestation Conveyance:       none
Authenticator Attachment:     platform  (Touch ID, Windows Hello)
Require Resident Key:         No
User Verification Requirement: preferred
```

## Step-up authentication

Require stronger auth for sensitive operations without full re-login.

1. Create a separate flow with the stronger step (OTP, WebAuthn)
2. Configure ACR (Authentication Context Class Reference) values:
   `Realm → Authentication → Policies → ACR to LoA Mapping`

```
1 → password only
2 → password + OTP
3 → password + WebAuthn
```

Client requests step-up by including `acr_values=2` in the auth request.

## Brute force protection

`Realm settings → Security defenses → Brute force detection`

```
Enabled:                    On
Permanent lockout:          Off (prefer temporary)
Max login failures:         5
Wait increment:             30 seconds
Max wait:                   900 seconds (15 min)
Failure reset time:         43200 seconds (12 hours)
```

Manually unlock a user: `Users → [user] → Credentials tab → Reset`

Or via API:
```bash
curl -X DELETE "https://keycloak/admin/realms/myrealm/attack-detection/brute-force/users/<user-id>" \
  -H "Authorization: Bearer $ADMIN_TOKEN"
```

## Required actions

Actions users must complete before their first login completes.

`Realm → Authentication → Required actions`

| Action | When to enable |
|---|---|
| `Verify Email` | On registration — send verification email |
| `Update Password` | Force password change on next login |
| `Configure OTP` | Require TOTP setup |
| `Terms and Conditions` | Require ToS acceptance |
| `WebAuthn Register` | Require passkey setup |

Assign to a specific user:
```bash
curl -X PUT "https://keycloak/admin/realms/myrealm/users/<user-id>" \
  -H "Authorization: Bearer $ADMIN_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{"requiredActions": ["UPDATE_PASSWORD", "CONFIGURE_TOTP"]}'
```

## Custom flow — example: skip OTP for internal IPs

1. Copy `browser` → `browser-conditional-otp`
2. Add `Condition - IP address` authenticator before OTP step
3. Configure allowed IP ranges
4. Set OTP step as `Conditional` under the IP condition

No code required — pure UI configuration.
