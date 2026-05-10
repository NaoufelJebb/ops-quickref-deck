# TOKEN DEBUG

## Decode a token fast

```bash
# Decode payload (no verification — debug only)
echo "<paste-payload-part>" | base64 -d 2>/dev/null | jq .

# Or with padding fix:
TOKEN_PAYLOAD="<paste-payload-part>"
python3 -c "
import base64, json, sys
p = sys.argv[1] + '=' * (4 - len(sys.argv[1]) % 4)
print(json.dumps(json.loads(base64.urlsafe_b64decode(p)), indent=2))
" "$TOKEN_PAYLOAD"
```

## What to check in a decoded token

```json
{
  "exp": 1704070800,     ← in the future? (epoch seconds)
  "iss": "https://keycloak/realms/myrealm",  ← exact match with your realm URL?
  "aud": "my-api",       ← contains your client/API identifier?
  "sub": "uuid",         ← user ID
  "realm_access": {
    "roles": ["app-admin"]  ← expected roles present?
  },
  "resource_access": {
    "my-client": {
      "roles": ["read"]  ← client roles present?
    }
  }
}
```

## Validate expiry

```bash
# Check if exp is in the future
EXP=$(echo "$TOKEN_PAYLOAD" | base64 -d 2>/dev/null | jq -r '.exp')
NOW=$(date +%s)
echo "Expires in: $((EXP - NOW)) seconds"
# Negative = expired
```

## Introspect via API

```bash
curl -X POST "$KC/realms/$REALM/protocol/openid-connect/token/introspect" \
  -u "my-client:$CLIENT_SECRET" \
  -d "token=$ACCESS_TOKEN" | jq '{active, sub, exp, realm_access}'
# active: false = invalid or expired
```

## Common errors and fixes

| Error | Cause | Fix |
|---|---|---|
| `401 Unauthorized` | Token expired or missing | Refresh the token |
| `403 Forbidden` | Token valid but wrong role/scope | Check role mapping, token evaluator |
| `iss mismatch` | Wrong Keycloak URL in API config | Match exactly including trailing slash |
| `aud mismatch` | Audience claim missing | Add audience mapper to client scope |
| `invalid_client` | Wrong client ID or secret | Verify in Keycloak console |
| `invalid_grant` | Refresh token expired or revoked | User must re-authenticate |
| `redirect_uri_mismatch` | URI not in allowed list | Add exact URI to client config |

## Use the token evaluator (no guesswork)

`Clients → [client] → Client scopes → Evaluate`

1. Enter username
2. Click "Generate access token"
3. Inspect claims directly — shows exactly what the token will contain
