# INCIDENT

## Identify

```bash
# Where are the errors coming from?
curl http://kong:8001/metrics | grep 'kong_http_requests_total.*status="5'

# Is an upstream down?
curl http://kong:8001/upstreams/<name>/health | jq '.data[] | {target, health}'

# What plugins are on the troubled route?
curl http://kong:8001/routes/<name>/plugins | jq '.data[] | {name, enabled}'

# Recent errors
docker logs kong 2>&1 | grep "\[error\]" | tail -20
```

## Mitigate

```bash
# Stop traffic to a bad target
curl -X PUT http://kong:8001/upstreams/<name>/targets/<host:port>/unhealthy

# Add a fallback target
curl -X POST http://kong:8001/upstreams/<name>/targets \
  --data target=fallback:8080 --data weight=100

# Kill switch — stop all traffic to a route
curl -X POST http://kong:8001/routes/<name>/plugins \
  -H "Content-Type: application/json" \
  -d '{"name":"request-termination","config":{"status_code":503,"message":"Temporarily unavailable"}}'

# Emergency throttle a flooding client
curl -X POST http://kong:8001/consumers/<name>/plugins \
  -H "Content-Type: application/json" \
  -d '{"name":"rate-limiting","config":{"second":1,"policy":"redis","redis_host":"redis","redis_port":6379}}'

# Disable a misbehaving plugin (without deleting it)
curl -X PATCH http://kong:8001/plugins/<id> -d enabled=false
```

## Restore

```bash
# Re-enable plugin
curl -X PATCH http://kong:8001/plugins/<id> -d enabled=true

# Mark target healthy again
curl -X PUT http://kong:8001/upstreams/<name>/targets/<host:port>/healthy

# Remove kill switch
curl -X DELETE http://kong:8001/plugins/<plugin-id>

# Remove emergency rate limit
curl -X DELETE http://kong:8001/consumers/<name>/plugins/<plugin-id>
```

## Nothing requires a restart. All changes are immediate.
