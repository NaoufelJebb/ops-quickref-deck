# OPS

## Daily health checks

```bash
# Node status
curl http://kong:8001/status | jq .

# Upstream health — anything red?
curl http://kong:8001/upstreams/<name>/health | jq '.data[] | {target, health}'

# Error rate (if Prometheus running)
curl -s "http://prometheus:9090/api/v1/query?query=sum(rate(kong_http_requests_total{status=~\"5..\"}[5m]))/sum(rate(kong_http_requests_total[5m]))*100" \
  | jq '.data.result[0].value[1]'

# Recent errors in logs
docker logs kong 2>&1 | grep -c "\[error\]"
```

---

## Block a consumer immediately

```bash
# Option 1 — delete JWT credential (surgical)
JWT_ID=$(curl -s http://kong:8001/consumers/bad-actor/jwt | jq -r '.data[0].id')
curl -X DELETE http://kong:8001/consumers/bad-actor/jwt/$JWT_ID

# Option 2 — remove ACL group
ACL_ID=$(curl -s http://kong:8001/consumers/bad-actor/acls | jq -r '.data[0].id')
curl -X DELETE http://kong:8001/consumers/bad-actor/acls/$ACL_ID

# Option 3 — terminate all their requests (reversible)
curl -X POST http://kong:8001/consumers/bad-actor/plugins \
  -H "Content-Type: application/json" \
  -d '{"name":"request-termination","config":{"status_code":403,"message":"Access denied"}}'
```

No restart needed — all changes are immediate.

---

## Kill switch — stop traffic to a route

```bash
curl -X POST http://kong:8001/routes/<name>/plugins \
  -H "Content-Type: application/json" \
  -d '{"name":"request-termination","config":{"status_code":503,"message":"Temporarily unavailable"}}'

# Undo
curl -X DELETE http://kong:8001/plugins/<plugin-id>
```

---

## Upstream traffic control

```bash
# Stop traffic to a specific target
curl -X PUT http://kong:8001/upstreams/<name>/targets/<host:port>/unhealthy

# Add an emergency fallback target
curl -X POST http://kong:8001/upstreams/<name>/targets \
  --data target=fallback:8080 --data weight=100

# Restore a target
curl -X PUT http://kong:8001/upstreams/<name>/targets/<host:port>/healthy
```

---

## Rolling upgrade (hybrid mode, zero-downtime)

```bash
# 1. Upgrade Control Plane first
# 2. Run DB migrations on CP only — DPs keep serving
kong migrations up
kong migrations finish

# 3. Validate config survives the upgrade
deck validate --config kong.yaml
deck diff --config kong.yaml

# 4. Upgrade Data Planes one by one
# LB drains each DP during restart; others continue serving

# 5. Confirm all DPs reconnected
curl http://kong:8001/clustering/data-planes | jq '.data[] | {hostname, version, status}'
```

Kong maintains backward compatibility between CP and DP across one major version.

---

## Config promotion (Dev → Staging → Prod)

```bash
# Recommended file structure
kong/
  kong-base.yaml       # shared across environments
  kong-prod.yaml       # prod-only overrides
  kong-staging.yaml    # staging-only overrides

# Staging
deck sync --config kong-base.yaml --config kong-staging.yaml \
  --kong-addr https://kong-staging:8444 \
  --header "Kong-Admin-Token: $STAGING_TOKEN"

# Production
deck sync --config kong-base.yaml --config kong-prod.yaml \
  --kong-addr https://kong-prod:8444 \
  --header "Kong-Admin-Token: $PROD_TOKEN" \
  --select-tag env:prod
```

---

## Consumer lifecycle

```bash
# Create
curl -X POST http://kong:8001/consumers --data username=my-client --data custom_id=ext-001

# Add credentials
curl -X POST http://kong:8001/consumers/my-client/jwt   --data algorithm=HS256 --data secret=<s>
curl -X POST http://kong:8001/consumers/my-client/key-auth --data key=<api-key>
curl -X POST http://kong:8001/consumers/my-client/acls  --data group=my-team

# Inspect
curl http://kong:8001/consumers/my-client/jwt
curl http://kong:8001/consumers/my-client/acls

# Offboard — delete credentials first, then optionally the consumer
curl -X DELETE http://kong:8001/consumers/my-client/jwt/<id>
curl -X DELETE http://kong:8001/consumers/my-client/key-auth/<id>
curl -X DELETE http://kong:8001/consumers/my-client     # optional
```
