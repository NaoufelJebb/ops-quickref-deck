# PLUGINS

## Adding a plugin — 3 methods

```bash
# 1. YAML (preferred) — under plugins:, services[].plugins, routes[].plugins
# 2. Admin API — POST /plugins, /services/<n>/plugins, /routes/<n>/plugins
# 3. curl JSON body — same endpoints, Content-Type: application/json

# Disable without deleting
curl -X PATCH http://kong:8001/plugins/<id> -d enabled=false

# Verify active plugins and their scope
curl http://kong:8001/plugins | jq '.data[] | {name, enabled, service, route, consumer}'
```

---

## Auth

### jwt

```yaml
- name: jwt
  config:
    claims_to_verify: [exp, nbf]
    key_claim_name: iss
    secret_is_base64: false
    run_on_preflight: false
    maximum_expiration: 3600
```

```bash
# Register consumer credential
curl -X POST http://kong:8001/consumers/my-app/jwt \
  --data algorithm=RS256 --data rsa_public_key=@public.pem
```

### key-auth

```yaml
- name: key-auth
  config:
    key_names: [apikey, X-API-Key]
    hide_credentials: true      # strip key before proxying
    key_in_query: false         # avoid keys in query strings
```

### mtls-auth *(Enterprise)*

```yaml
- name: mtls-auth
  config:
    ca_certificates: [<ca-uuid>]
    skip_consumer_lookup: false
    authenticated_group_by: DN
```

### acl

```yaml
- name: acl
  config:
    allow: [team-a, admins]
    hide_groups_header: true
```

```bash
curl -X POST http://kong:8001/consumers/my-app/acls --data group=team-a
```

---

## Traffic

### rate-limiting

```yaml
- name: rate-limiting
  config:
    second: 100
    minute: 1000
    hour: 50000
    limit_by: consumer           # ip | service | header | credential
    policy: redis                # ⚠️  always redis in multi-node
    redis_host: redis
    redis_port: 6379
    fault_tolerant: true
```

### proxy-cache

```yaml
- name: proxy-cache
  config:
    response_code: [200, 301]
    request_method: [GET, HEAD]
    content_type: [application/json]
    cache_ttl: 300
    strategy: memory
```

### request-size-limiting

```yaml
- name: request-size-limiting
  config:
    allowed_payload_size: 10    # MB
    size_unit: megabytes
```

---

## Observability

### prometheus

```yaml
- name: prometheus
  config:
    per_consumer: true
    status_code_metrics: true
    latency_metrics: true
    bandwidth_metrics: true
    upstream_health_metrics: true
```

### http-log

```yaml
- name: http-log
  config:
    http_endpoint: https://splunk:8088/services/collector
    method: POST
    timeout: 5000
    retry_count: 10
```

### opentelemetry

```yaml
- name: opentelemetry
  config:
    endpoint: http://otel-collector:4318/v1/traces
    resource_attributes:
      service.name: kong-gateway
    propagation:
      default_format: w3c
    sampling_rate: 1.0
```

---

## Transformation

### request-transformer

```yaml
- name: request-transformer
  config:
    add:
      headers:
        - "X-Consumer-ID:$(consumer.id)"
        - "X-Request-ID:$(request.id)"
    remove:
      headers: [Authorization]   # strip before proxying
```

### response-transformer

```yaml
- name: response-transformer
  config:
    add:
      headers:
        - "Strict-Transport-Security:max-age=31536000"
        - "X-Content-Type-Options:nosniff"
        - "X-Frame-Options:DENY"
    remove:
      headers: [Server, X-Powered-By]
```

### cors

```yaml
- name: cors
  config:
    origins: [https://app.example.com]
    methods: [GET, POST, PUT, DELETE, OPTIONS]
    headers: [Authorization, Content-Type]
    credentials: true
    max_age: 3600
```

### request-termination *(stub / kill switch)*

```yaml
- name: request-termination
  config:
    status_code: 503
    message: "Temporarily unavailable"
    content_type: application/json
    body: '{"error":"maintenance"}'
```
