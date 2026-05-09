# CHEATSHEET

## decK

```bash
deck validate --config kong.yaml
deck diff     --config kong.yaml
deck sync     --config kong.yaml
deck dump     --output-file backup.yaml
deck reset                                   # ⚠️  wipes everything
```

## Admin API

```bash
# Health
curl http://kong:8001/status | jq .

# Services
curl http://kong:8001/services
curl http://kong:8001/services/<name>
curl http://kong:8001/services/<name>/plugins
curl http://kong:8001/services/<name>/routes

# Routes
curl http://kong:8001/routes
curl http://kong:8001/routes/<name>/plugins

# Plugins
curl http://kong:8001/plugins
curl -X PATCH http://kong:8001/plugins/<id> -d enabled=false    # disable
curl -X PATCH http://kong:8001/plugins/<id> -d enabled=true     # re-enable
curl -X DELETE http://kong:8001/plugins/<id>

# Consumers
curl http://kong:8001/consumers
curl http://kong:8001/consumers/<name>
curl http://kong:8001/consumers/<name>/jwt
curl http://kong:8001/consumers/<name>/key-auth
curl http://kong:8001/consumers/<name>/acls
curl http://kong:8001/consumers/<name>/plugins

# Upstreams
curl http://kong:8001/upstreams
curl http://kong:8001/upstreams/<name>/health
curl http://kong:8001/upstreams/<name>/targets
curl -X PUT http://kong:8001/upstreams/<name>/targets/<host:port>/unhealthy
curl -X PUT http://kong:8001/upstreams/<name>/targets/<host:port>/healthy

# Certificates
curl http://kong:8001/certificates
curl http://kong:8001/snis
curl http://kong:8001/ca_certificates

# Hybrid
curl http://kong:8001/clustering/data-planes
```

## Emergency actions

```bash
# Block a consumer
curl -X DELETE http://kong:8001/consumers/<name>/jwt/<id>

# Kill switch on a route
curl -X POST http://kong:8001/routes/<name>/plugins \
  -H "Content-Type: application/json" \
  -d '{"name":"request-termination","config":{"status_code":503}}'

# Stop traffic to a bad upstream target
curl -X PUT http://kong:8001/upstreams/<name>/targets/<host:port>/unhealthy

# Emergency rate limit a client
curl -X POST http://kong:8001/consumers/<name>/plugins \
  -H "Content-Type: application/json" \
  -d '{"name":"rate-limiting","config":{"second":1,"policy":"redis","redis_host":"redis","redis_port":6379}}'
```

## Status codes

| Code | Meaning |
|---|---|
| `401` | Bad / missing credential |
| `403` | ACL denied |
| `404` | No route matched |
| `429` | Rate limit hit |
| `502` | Upstream bad response |
| `503` | No healthy targets |
| `504` | Upstream timeout |

## Key files

| Path | What |
|---|---|
| `/etc/kong/kong.conf` | Main config |
| `/usr/local/kong/logs/error.log` | Error log |
| `/usr/local/kong/logs/access.log` | Access log |
| `~/.deck.yaml` | decK CLI defaults |
| `kong.yaml` | Declarative config (keep in Git) |

## Env vars for scripting

```bash
export KONG_ADMIN=http://localhost:8001
export KONG_TOKEN=your-token

curl -H "Kong-Admin-Token: $KONG_TOKEN" $KONG_ADMIN/services
deck sync --config kong.yaml --kong-addr $KONG_ADMIN --header "Kong-Admin-Token: $KONG_TOKEN"
```
