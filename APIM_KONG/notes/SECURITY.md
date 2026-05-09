# SECURITY

## Admin API lockdown

```bash
# kong.conf — bind only to internal/loopback
admin_listen = 127.0.0.1:8001, 127.0.0.1:8444 ssl

# Or via env var
KONG_ADMIN_LISTEN="127.0.0.1:8001"
```

Enable token auth for decK and CI/CD access *(Enterprise)*:

```bash
# Create RBAC user
curl -X POST http://localhost:8001/rbac/users \
  --data name=ci-user --data user_token=my-ci-token

# Use with decK
deck sync --config kong.yaml --header "Kong-Admin-Token: my-ci-token"
```

## Trusted IPs — required for rate-limiting by IP

Without this, Kong sees your load balancer's IP, not the real client.

```bash
# kong.conf
trusted_ips = 10.0.0.0/8,172.16.0.0/12,192.168.0.0/16
real_ip_header = X-Forwarded-For
real_ip_recursive = on
```

## Restrict loaded plugins

Reduce attack surface — only load what you use:

```bash
# kong.conf
plugins = jwt,rate-limiting,prometheus,http-log,acl,request-transformer,cors,opentelemetry
```

## Security response headers (global plugin)

```yaml
plugins:
  - name: response-transformer
    config:
      add:
        headers:
          - "Strict-Transport-Security:max-age=31536000; includeSubDomains"
          - "X-Content-Type-Options:nosniff"
          - "X-Frame-Options:DENY"
          - "X-XSS-Protection:1; mode=block"
          - "Referrer-Policy:strict-origin-when-cross-origin"
          - "Cache-Control:no-store"
      remove:
        headers:
          - "Server"
          - "X-Powered-By"
          - "Via"
```

## Checklist

- [ ] Admin API bound to `127.0.0.1` or internal network — not `0.0.0.0`
- [ ] Admin API protected by RBAC token or mTLS *(Enterprise)*
- [ ] `trusted_ips` set to load balancer CIDR(s)
- [ ] `plugins` config allowlists only what is actively used
- [ ] Security response headers applied globally
- [ ] All proxy traffic on TLS (`8443`) — HTTP either redirected or blocked
- [ ] Certificates monitored for expiry — alert at ≤ 30 days
- [ ] Consumer offboarding: **credentials deleted**, not just the consumer record
- [ ] `deck reset` gated behind manual approval in CI/CD
- [ ] Kong Manager port `8002` not reachable from the internet
