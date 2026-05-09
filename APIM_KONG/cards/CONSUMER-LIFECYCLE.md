# CONSUMER LIFECYCLE

## Create

```bash
curl -X POST http://kong:8001/consumers \
  --data username=my-client \
  --data custom_id=ext-001
```

## Add credentials

```bash
# JWT
curl -X POST http://kong:8001/consumers/my-client/jwt \
  --data algorithm=HS256 --data secret=<secret>

# API key
curl -X POST http://kong:8001/consumers/my-client/key-auth \
  --data key=<api-key>

# ACL group
curl -X POST http://kong:8001/consumers/my-client/acls \
  --data group=my-team
```

## Inspect

```bash
curl http://kong:8001/consumers/my-client/jwt
curl http://kong:8001/consumers/my-client/key-auth
curl http://kong:8001/consumers/my-client/acls
curl http://kong:8001/consumers/my-client/plugins
```

## Offboard (revoke access)

```bash
# Get the credential ID first
JWT_ID=$(curl -s http://kong:8001/consumers/my-client/jwt | jq -r '.data[0].id')

# Delete it — takes effect immediately, no restart
curl -X DELETE http://kong:8001/consumers/my-client/jwt/$JWT_ID

# Hard block if deletion isn't enough
curl -X POST http://kong:8001/consumers/my-client/plugins \
  -H "Content-Type: application/json" \
  -d '{"name":"request-termination","config":{"status_code":403,"message":"Access revoked"}}'
```

> Delete credentials, not just the consumer record. A deleted consumer with live credentials still lets requests through.
