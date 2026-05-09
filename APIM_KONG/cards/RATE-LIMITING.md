# RATE LIMITING

## The one rule

```
policy: redis    ← always, in any multi-node deployment
policy: local    ← per-node; allows N × limit across N nodes
```

## Minimal config

```yaml
- name: rate-limiting
  config:
    minute: 1000
    policy: redis
    redis_host: redis
    redis_port: 6379
    fault_tolerant: true     # allow traffic if Redis is down
    limit_by: consumer       # ip | service | header | credential
```

## Per-consumer override

```bash
curl -X POST http://kong:8001/consumers/premium-client/plugins \
  -H "Content-Type: application/json" \
  -d '{"name":"rate-limiting","config":{"minute":10000,"policy":"redis","redis_host":"redis","redis_port":6379}}'
```

## Response headers exposed to client

```
X-RateLimit-Limit-Minute: 1000
X-RateLimit-Remaining-Minute: 847
RateLimit-Reset: 1704067200
```

Set `hide_client_headers: true` to suppress these.

## Debug — verify Redis is being used

```bash
# Check the active policy
curl http://kong:8001/plugins | jq '.data[] | select(.name=="rate-limiting") | .config.policy'

# Confirm Redis has keys
redis-cli keys "kong_rate_limiting*"
```
