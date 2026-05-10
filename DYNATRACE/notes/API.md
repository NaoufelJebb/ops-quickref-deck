# API

## Base URL

```
https://<env-id>.live.dynatrace.com/api/v2/
```

## Authentication

All requests use an API token in the `Authorization` header:

```bash
-H "Authorization: Api-Token <your-token>"
```

Create tokens: `Settings → Access tokens → Generate new token`

---

## Token scopes — most used

| Scope | What it allows |
|---|---|
| `metrics.read` | Query metrics |
| `metrics.ingest` | Push custom metrics |
| `metrics.write` | Create metric descriptors |
| `logs.read` | Query logs |
| `logs.ingest` | Push logs |
| `entities.read` | Query entities (hosts, services…) |
| `problems.read` | Read problems |
| `problems.write` | Comment / close problems |
| `settings.read` | Read settings |
| `settings.write` | Modify settings |
| `events.read` | Query events |
| `events.ingest` | Push custom events |
| `DataExport` | Export raw data |
| `InstallerDownload` | Download OneAgent / ActiveGate installers |

> Principle of least privilege — create separate tokens per use case.

---

## Common endpoints

### Metrics

```bash
# List available metrics
GET /api/v2/metrics

# Query metric data
GET /api/v2/metrics/query
  ?metricSelector=builtin:service.response.time:avg
  &from=now-2h
  &to=now
  &resolution=5m
  &entitySelector=type(SERVICE)

# Ingest custom metric
POST /api/v2/metrics/ingest
Content-Type: text/plain
Body: my.metric,env=prod 42.0
```

### Entities

```bash
# List entities
GET /api/v2/entities
  ?entitySelector=type(SERVICE)
  &fields=+tags,+properties
  &from=now-3h

# Get a specific entity
GET /api/v2/entities/SERVICE-XXXXXXXX

# Entity selector examples
type(HOST)
type(SERVICE),tag("env:prod")
type(PROCESS_GROUP),entityName("my-service")
type(SERVICE),mzName("production")
```

### Problems

```bash
# List open problems
GET /api/v2/problems
  ?problemSelector=status(open)
  &from=now-24h

# Get a specific problem
GET /api/v2/problems/PROBLEM-XXXXXXXX

# Add comment to a problem
POST /api/v2/problems/PROBLEM-XXXXXXXX/comments
{
  "message": "Investigating — on-call team notified",
  "context": "ops"
}
```

### Events

```bash
# Push a deployment event
POST /api/v2/events/ingest
{
  "eventType": "CUSTOM_DEPLOYMENT",
  "title": "Deploy v2.4.1",
  "entitySelector": "type(SERVICE),tag(\"app:checkout\")",
  "properties": {
    "deploymentVersion": "2.4.1",
    "deployedBy": "CI/CD"
  }
}

# Push a custom annotation event
POST /api/v2/events/ingest
{
  "eventType": "CUSTOM_ANNOTATION",
  "title": "Config change: timeout increased",
  "entitySelector": "type(SERVICE),entityName(\"payments\")"
}
```

### Logs

```bash
# Query logs
GET /api/v2/logs/search
  ?query=loglevel%3D%22ERROR%22
  &from=now-1h
  &limit=100

# Ingest logs
POST /api/v2/logs/ingest
[{"content":"message","severity":"INFO","dt.entity.service":"SERVICE-XXX"}]
```

### Settings

```bash
# List setting schemas
GET /api/v2/settings/schemas

# Get settings for a schema
GET /api/v2/settings/objects
  ?schemaIds=builtin:anomaly-detection.metric-events
  &scopes=environment

# Create/update a setting
POST /api/v2/settings/objects
[{
  "schemaId": "builtin:anomaly-detection.metric-events",
  "scope": "environment",
  "value": { ... }
}]
```

---

## Useful curl patterns

```bash
# Set env vars to reduce repetition
export DT_ENV=https://abc12345.live.dynatrace.com
export DT_TOKEN=dt0c01.XXXXXX

# List all services
curl "$DT_ENV/api/v2/entities?entitySelector=type(SERVICE)&fields=+tags" \
  -H "Authorization: Api-Token $DT_TOKEN" | jq '.entities[] | {name: .displayName, id: .entityId}'

# Get open problems count
curl "$DT_ENV/api/v2/problems?problemSelector=status(open)" \
  -H "Authorization: Api-Token $DT_TOKEN" | jq '.totalCount'

# Push a deployment marker
curl -X POST "$DT_ENV/api/v2/events/ingest" \
  -H "Authorization: Api-Token $DT_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
    "eventType": "CUSTOM_DEPLOYMENT",
    "title": "Deploy '"$VERSION"'",
    "entitySelector": "type(SERVICE),tag(\"app:my-service\")"
  }'

# Ingest a custom metric
curl -X POST "$DT_ENV/api/v2/metrics/ingest" \
  -H "Authorization: Api-Token $DT_TOKEN" \
  -H "Content-Type: text/plain" \
  -d "my.queue.depth,queue=orders,env=prod 42"
```

---

## Rate limits

| Endpoint group | Limit |
|---|---|
| Metrics query | 50 requests/minute |
| Metrics ingest | 1000 data points/minute per tenant |
| Log ingest | 1000 requests/minute |
| Entities | 50 requests/minute |
| Problems | 50 requests/minute |

429 response = rate limited. Implement exponential backoff.
