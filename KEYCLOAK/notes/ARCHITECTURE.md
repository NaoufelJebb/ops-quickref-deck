# ARCHITECTURE

## Deployment modes

| Mode | Description | Use for |
|---|---|---|
| **Development** | `start-dev` — H2 in-memory DB, no TLS | Local dev only |
| **Production** | `start` — requires external DB, TLS, hostname config | All real environments |
| **Operator (K8s)** | Managed via Keycloak Operator CRD | Kubernetes deployments |

## Production stack

```
Internet
   │
   ▼
[ Load Balancer / Reverse Proxy ]   ← terminates TLS
   │           │
   ▼           ▼
[ Keycloak ] [ Keycloak ]           ← stateless, horizontally scalable
   │           │
   └─────┬─────┘
         ▼
   [ PostgreSQL ]                   ← single source of truth
         │
   [ Infinispan / JGroups ]         ← distributed session cache (optional external)
```

## Database support

| DB | Supported | Recommended |
|---|---|---|
| PostgreSQL | ✅ | ✅ Yes |
| MySQL / MariaDB | ✅ | OK |
| MSSQL | ✅ | OK |
| Oracle | ✅ | OK |
| H2 | Dev only | ❌ Never prod |

## Key configuration — `keycloak.conf`

```bash
# Database
db=postgres
db-url=jdbc:postgresql://postgres:5432/keycloak
db-username=keycloak
db-password=${KC_DB_PASSWORD}

# HTTP / TLS
http-enabled=false
https-certificate-file=/etc/keycloak/tls.crt
https-certificate-key-file=/etc/keycloak/tls.key
https-port=8443

# Hostname (what clients see)
hostname=auth.example.com
hostname-strict=true
hostname-strict-https=true

# Proxy (if behind a reverse proxy)
proxy=edge           # reverse proxy terminates TLS
# or:
proxy=reencrypt      # reverse proxy re-encrypts to Keycloak
# or:
proxy=passthrough    # Keycloak handles TLS itself

# Cache (for HA clustering)
cache=ispn
cache-stack=kubernetes   # or: tcp, udp, ec2, azure, google

# Health / metrics
health-enabled=true
metrics-enabled=true
```

## Docker — production-ready

```yaml
# docker-compose.yaml
version: "3.8"
services:
  postgres:
    image: postgres:15
    environment:
      POSTGRES_DB: keycloak
      POSTGRES_USER: keycloak
      POSTGRES_PASSWORD: ${KC_DB_PASSWORD}
    volumes:
      - postgres_data:/var/lib/postgresql/data
    healthcheck:
      test: ["CMD", "pg_isready", "-U", "keycloak"]
      interval: 5s
      retries: 5

  keycloak:
    image: quay.io/keycloak/keycloak:24.0
    command: start
    environment:
      KC_DB: postgres
      KC_DB_URL: jdbc:postgresql://postgres:5432/keycloak
      KC_DB_USERNAME: keycloak
      KC_DB_PASSWORD: ${KC_DB_PASSWORD}
      KC_HOSTNAME: auth.example.com
      KC_PROXY: edge
      KC_HTTP_ENABLED: "true"           # only if behind TLS-terminating proxy
      KEYCLOAK_ADMIN: admin
      KEYCLOAK_ADMIN_PASSWORD: ${KC_ADMIN_PASSWORD}
    ports:
      - "8080:8080"
      - "8443:8443"
    depends_on:
      postgres:
        condition: service_healthy

volumes:
  postgres_data:
```

## Reverse proxy (Nginx)

```nginx
server {
    listen 443 ssl;
    server_name auth.example.com;

    ssl_certificate     /etc/ssl/certs/auth.example.com.crt;
    ssl_certificate_key /etc/ssl/private/auth.example.com.key;

    location / {
        proxy_pass          http://keycloak:8080;
        proxy_set_header    Host               $host;
        proxy_set_header    X-Real-IP          $remote_addr;
        proxy_set_header    X-Forwarded-For    $proxy_add_x_forwarded_for;
        proxy_set_header    X-Forwarded-Proto  $scheme;
        proxy_set_header    X-Forwarded-Host   $host;
        proxy_set_header    X-Forwarded-Port   $server_port;
        proxy_buffer_size   128k;
        proxy_buffers       4 256k;
    }
}
```

## HA clustering

Keycloak uses JGroups for node discovery and Infinispan for distributed session cache.

```bash
# K8s — use DNS ping for discovery
cache-stack=kubernetes

# Bare metal / VMs — TCP ping
cache-stack=tcp
jgroups-discovery-protocol=TCPPING
jgroups-discovery-properties=initial_hosts="kc-1[7800],kc-2[7800]"
```

Nodes are stateless — scale horizontally freely. Sessions are stored in the shared DB and/or Infinispan cluster.

## Health endpoints

```bash
GET http://keycloak:8080/health        # overall health
GET http://keycloak:8080/health/ready  # ready to serve traffic
GET http://keycloak:8080/health/live   # liveness probe
GET http://keycloak:8080/metrics       # Prometheus metrics (if enabled)
```
