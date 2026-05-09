# PLUGIN SCOPES

## Decision

```
Need it everywhere?            → Global
One service, all its routes?   → Service
One endpoint only?             → Route
One specific client?           → Consumer
```

## YAML positions

```yaml
# Global
plugins:
  - name: prometheus

# Service
services:
  - name: my-service
    plugins:
      - name: rate-limiting
        config:
          minute: 1000

# Route
services:
  - name: my-service
    routes:
      - name: my-route
        plugins:
          - name: jwt

# Consumer (Admin API only — not in YAML)
# POST /consumers/<name>/plugins
```

## Admin API — all four scopes

```bash
curl -X POST http://kong:8001/plugins                          # global
curl -X POST http://kong:8001/services/<n>/plugins             # service
curl -X POST http://kong:8001/routes/<n>/plugins               # route
curl -X POST http://kong:8001/consumers/<n>/plugins            # consumer
```

## Override order

```
Consumer > Route > Service > Global
```

Lower = higher priority. A Consumer plugin wins over everything.
