# LOCAL DEV

## Docker Compose — DB mode

```yaml
# docker-compose.yaml
version: "3.8"
services:
  kong-db:
    image: postgres:15
    environment:
      POSTGRES_USER: kong
      POSTGRES_DB: kong
      POSTGRES_PASSWORD: kongpass
    healthcheck:
      test: ["CMD", "pg_isready", "-U", "kong"]
      interval: 5s
      retries: 5

  kong-migrations:
    image: kong/kong-gateway:3.6
    command: kong migrations bootstrap
    environment:
      KONG_DATABASE: postgres
      KONG_PG_HOST: kong-db
      KONG_PG_USER: kong
      KONG_PG_PASSWORD: kongpass
    depends_on:
      kong-db:
        condition: service_healthy

  kong:
    image: kong/kong-gateway:3.6
    environment:
      KONG_DATABASE: postgres
      KONG_PG_HOST: kong-db
      KONG_PG_USER: kong
      KONG_PG_PASSWORD: kongpass
      KONG_PROXY_ACCESS_LOG: /dev/stdout
      KONG_ADMIN_ACCESS_LOG: /dev/stdout
      KONG_PROXY_ERROR_LOG: /dev/stderr
      KONG_ADMIN_ERROR_LOG: /dev/stderr
      KONG_ADMIN_LISTEN: "0.0.0.0:8001"
    ports:
      - "8000:8000"
      - "8443:8443"
      - "8001:8001"
    depends_on:
      kong-migrations:
        condition: service_completed_successfully

  redis:
    image: redis:7-alpine
    ports:
      - "6379:6379"
```

```bash
docker compose up -d
curl http://localhost:8001/status | jq .    # verify
docker compose down -v                      # teardown + wipe volumes
```

---

## Docker Compose — DB-less mode (lighter)

```yaml
# docker-compose.dbless.yaml
version: "3.8"
services:
  kong:
    image: kong/kong-gateway:3.6
    environment:
      KONG_DATABASE: "off"
      KONG_DECLARATIVE_CONFIG: /kong/declarative/kong.yaml
      KONG_PROXY_ACCESS_LOG: /dev/stdout
      KONG_ADMIN_ACCESS_LOG: /dev/stdout
      KONG_PROXY_ERROR_LOG: /dev/stderr
      KONG_ADMIN_ERROR_LOG: /dev/stderr
      KONG_ADMIN_LISTEN: "0.0.0.0:8001"
    volumes:
      - ./kong.yaml:/kong/declarative/kong.yaml
    ports:
      - "8000:8000"
      - "8001:8001"
```

```bash
docker compose -f docker-compose.dbless.yaml up -d

# Reload config after editing kong.yaml — no restart needed
curl -X POST http://localhost:8001/config -F config=@kong.yaml
```

---

## Stub an upstream for local testing

Use `request-termination` to return a fake response without a real upstream.

```bash
# 1. Create stub service (URL doesn't need to exist)
curl -X POST http://localhost:8001/services \
  --data name=stub --data url=http://stub:8080

# 2. Create route
curl -X POST http://localhost:8001/services/stub/routes \
  --data name=stub-route --data "paths[]=/api/test"

# 3. Return fake JSON response
curl -X POST http://localhost:8001/services/stub/plugins \
  -H "Content-Type: application/json" \
  -d '{
    "name": "request-termination",
    "config": {
      "status_code": 200,
      "body": "{\"status\":\"ok\",\"mock\":true}",
      "content_type": "application/json"
    }
  }'

# 4. Test
curl http://localhost:8000/api/test
# → {"status":"ok","mock":true}
```
