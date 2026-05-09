# CONFIG

## Golden rule

```
deck validate → deck diff → deck sync
Never skip the diff in production.
```

## Minimal kong.yaml

```yaml
_format_version: "3.0"

services:
  - name: my-service
    url: http://my-api:8080
    connect_timeout: 5000
    read_timeout: 60000
    routes:
      - name: my-route
        paths:
          - /api/v1
        methods: [GET, POST]
        strip_path: true          # /api/v1/users → /users upstream
    plugins:
      - name: jwt
      - name: rate-limiting
        config:
          minute: 1000
          policy: redis           # never "local" in multi-node
          redis_host: redis
          redis_port: 6379

plugins:
  - name: prometheus              # global — all traffic
    config:
      per_consumer: true
      latency_metrics: true
      status_code_metrics: true
```

## Essential decK commands

```bash
deck validate --config kong.yaml            # lint, no Kong connection needed
deck diff     --config kong.yaml            # preview — read like a PR
deck sync     --config kong.yaml            # apply
deck dump     --output-file backup.yaml     # export current state
deck reset                                  # ⚠️  wipe all — never in prod
```

## Multi-environment sync

```bash
deck sync --config kong.yaml \
  --kong-addr https://kong-prod:8444 \
  --header "Kong-Admin-Token: $KONG_TOKEN" \
  --select-tag env:prod
```

## Secret injection (never hardcode)

```yaml
config:
  redis_password: ${{ env "REDIS_PASSWORD" }}
  # Kong Enterprise vaults:
  redis_password: "{vault://hcv/redis/password}"
```

## Upstream with health checks

```yaml
upstreams:
  - name: my-upstream
    algorithm: round-robin        # or: least-connections, consistent-hashing
    healthchecks:
      active:
        http_path: /health
        interval: 5
        healthy:
          successes: 2
        unhealthy:
          http_failures: 3
      passive:
        unhealthy:
          http_failures: 5
    targets:
      - target: api-1:8080
        weight: 100
      - target: api-2:8080
        weight: 100

services:
  - name: my-service
    host: my-upstream             # reference upstream by name
    port: 80
    protocol: http
```

## Admin API — imperative (exploration only)

```bash
# Service
curl -X POST http://localhost:8001/services \
  --data name=my-service --data url=http://my-api:8080

# Route
curl -X POST http://localhost:8001/services/my-service/routes \
  --data name=my-route --data "paths[]=/api/v1" --data strip_path=true

# Plugin on a service
curl -X POST http://localhost:8001/services/my-service/plugins \
  -H "Content-Type: application/json" \
  -d '{"name":"rate-limiting","config":{"minute":1000,"policy":"redis"}}'
```
