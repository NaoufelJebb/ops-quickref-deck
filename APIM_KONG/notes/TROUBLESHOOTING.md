# TROUBLESHOOTING

## Status codes — what they mean

| Code | Cause |
|---|---|
| `401` | Missing or invalid credential (JWT, API key) |
| `403` | ACL denied — consumer not in allowed group |
| `404` | No route matched the request |
| `429` | Rate limit exceeded |
| `494` | Request headers too large |
| `502` | Upstream returned invalid/empty response |
| `503` | No healthy upstream targets |
| `504` | Upstream exceeded `read_timeout` |

---

## 404 — Route not matching

```bash
# 1. List routes and their match criteria
curl http://kong:8001/routes | jq '.data[] | {name, paths, methods, hosts}'

# 2. Use the debug header (Kong 3.x)
curl -i http://kong:8000/api/v1/test -H "Kong-Debug: 1"
# → X-Kong-Matched-Route in response

# 3. Check the service is reachable
curl http://kong:8001/services/my-service
```

| Symptom | Likely cause |
|---|---|
| 404 on all routes | Kong started before config loaded; check `deck sync` |
| 404 on one route | Path mismatch — check trailing slash, case, regex |
| Wrong service matched | Route priority — Kong matches by specificity |

---

## Plugin not applying

```bash
# 1. Confirm plugin exists and is enabled at the right scope
curl http://kong:8001/plugins | jq '.data[] | select(.name=="jwt") | {name, enabled, service, route, consumer}'

# 2. Check for consumer-level override (overrides service/route)
curl http://kong:8001/consumers/<name>/plugins | jq .

# 3. Verify plugin config is valid
curl http://kong:8001/plugins/<id> | jq '.config'
```

---

## Rate limiting allowing too much traffic

```bash
# Check the policy
curl http://kong:8001/plugins | jq '.data[] | select(.name=="rate-limiting") | .config.policy'

# "local" in multi-node = bug. Fix:
curl -X PATCH http://kong:8001/plugins/<id> \
  -H "Content-Type: application/json" \
  -d '{"config":{"policy":"redis","redis_host":"redis","redis_port":6379}}'

# Verify Redis has the keys
redis-cli keys "kong_rate_limiting*"
```

`policy: local` = each node enforces limits independently → N nodes × limit passes through.

---

## 502 / 504 — Upstream errors

```bash
# 1. Check upstream health
curl http://kong:8001/upstreams/my-upstream/health | jq .

# 2. Test upstream reachability from inside Kong container
docker exec kong curl -i http://my-api:8080/health

# 3. Check timeouts
curl http://kong:8001/services/my-service | jq '{connect_timeout, read_timeout, write_timeout}'

# 4. Check error logs
docker logs kong 2>&1 | grep -E "\[error\]|\[crit\]" | tail -30

# 5. Stop traffic to a bad target immediately
curl -X PUT http://kong:8001/upstreams/my-upstream/targets/bad-host:8080/unhealthy
```

---

## JWT auth failures

```bash
# Decode payload to inspect claims (non-prod only)
echo "<jwt-payload-part>" | base64 -d | jq .

# Check consumer's registered credentials
curl http://kong:8001/consumers/<username>/jwt | jq '.data[] | {algorithm, key}'

# Test with explicit header
curl http://kong:8000/api/v1 -H "Authorization: Bearer <token>"
```

| Error message | Cause |
|---|---|
| `No mandatory 'iss' found` | `iss` claim missing from token |
| `Invalid signature` | Wrong secret or algorithm mismatch |
| `Token is expired` | `exp` claim is in the past |
| `Invalid credentials` | No consumer found for that `iss` value |

---

## Reading logs

```bash
# Docker
docker logs kong 2>&1 | tail -100

# Systemd
journalctl -u kong -f

# Files
tail -f /usr/local/kong/logs/error.log | grep -E "error|warn|crit"

# Access log (every proxied request)
tail -f /usr/local/kong/logs/access.log | jq .
```

### Common log patterns

```
failed to connect to upstream          → upstream unreachable, check DNS/network
no consumer for this request           → JWT/key-auth consumer not found
could not determine current worker     → Redis connection issue
init_by_lua error                      → plugin config error at startup
upstream prematurely closed connection → upstream closed before full response
SSL handshake failed                   → mTLS cert issue with upstream
```

## Temporary debug logging

```bash
# Enable (not for production)
curl -X PATCH http://kong:8001/ -d "configuration.log_level=debug"

# Reset
curl -X PATCH http://kong:8001/ -d "configuration.log_level=notice"
```
