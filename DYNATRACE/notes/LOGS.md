# LOGS

## Ingestion methods

| Method | How |
|---|---|
| **OneAgent** | Auto-collects from monitored processes — zero config |
| **Log Monitoring API** | Push logs via HTTP (sidecar-less, external sources) |
| **Fluentd / Fluent Bit** | Dynatrace output plugin |
| **Logstash** | Dynatrace output plugin |
| **OpenTelemetry** | OTLP logs endpoint |
| **ActiveGate** | Log sources polled by extensions |

---

## OneAgent — automatic log collection

OneAgent captures logs from monitored processes automatically. To add a custom log file:

`Settings → Log Monitoring → Log sources and storage`

```
Custom log source:
  Path:    /var/log/myapp/*.log
  Process: myapp
```

---

## Log Ingest API

```bash
# POST a log batch
curl -X POST "https://<env>.live.dynatrace.com/api/v2/logs/ingest" \
  -H "Authorization: Api-Token $DT_API_TOKEN" \
  -H "Content-Type: application/json; charset=utf-8" \
  -d '[
    {
      "timestamp": "2024-01-01T10:00:00.000Z",
      "content": "Payment processed successfully",
      "severity": "INFO",
      "dt.entity.service": "SERVICE-XXXXXXXX",
      "custom.attribute": "checkout"
    },
    {
      "timestamp": "2024-01-01T10:00:01.000Z",
      "content": "NullPointerException in PaymentService",
      "severity": "ERROR",
      "dt.entity.service": "SERVICE-XXXXXXXX"
    }
  ]'
```

Required token scope: `logs.ingest`

### Fluent Bit config

```ini
[OUTPUT]
    Name              dynatrace
    Match             *
    Host              <env>.live.dynatrace.com
    Port              443
    API_Token         <token>
    Log_Strict_SSL    On
    Log_Tag_Attribute app.name
```

---

## Log processing rules

Parse and enrich logs server-side before storage.

`Settings → Log Monitoring → Log processing`

### DQL-based processing rule

```
# Parse JSON logs
PARSE(content, "JSON:parsed")
| FIELDS_ADD status = parsed[status], user = parsed[userId]

# Extract fields with a pattern
PARSE(content, "LD 'status=' INT:status_code ' ' LD")

# Mask sensitive data
FIELDS_REPLACE content = REPLACE_PATTERN(content, "'password':'[^']*'", "'password':'***'")
```

### Log metric extraction

Convert log patterns into metrics (no extra metric ingest cost):

`Settings → Log Monitoring → Log metrics`

```
Name:     error_count_by_service
DQL:      fetch logs | filter loglevel == "ERROR"
Metric:   log.error_count
Dimension: dt.entity.service
```

---

## Log storage and retention

`Settings → Log Monitoring → Log storage`

- Define which log attributes to index (indexed = faster search, costs DDUs)
- Set retention per log source (default: 35 days SaaS, configurable)
- Non-indexed logs still stored but slower to query

---

## Log viewer — quick navigation

| Action | How |
|---|---|
| Open log viewer | Menu → Logs |
| Filter by service | Click a service in Smartscape, then "View logs" |
| Filter by severity | Click ERROR / WARN / INFO pills |
| Switch to DQL | Toggle "Advanced mode" |
| Export logs | Overflow menu → Export |
| Create metric from query | "Create metric" button in Log viewer |

---

## Common log DQL queries

```dql
// Errors in the last hour
fetch logs, from: now()-1h
| filter loglevel == "ERROR"
| fields timestamp, loglevel, content, dt.entity.service
| sort timestamp desc

// Specific exception
fetch logs
| filter matchesPhrase(content, "NullPointerException")
| fields timestamp, content, dt.entity.host
| sort timestamp desc

// Log volume by service (for dashboard)
fetch logs
| summarize count = count(), by: {service = dt.entity.service, bin(timestamp, 5m)}
| sort timestamp asc

// Errors by service — ranked
fetch logs
| filter loglevel == "ERROR"
| summarize errors = count(), by: {dt.entity.service}
| sort errors desc

// Parse a structured log field
fetch logs
| filter matchesPhrase(content, "payment")
| parse content, "JSON:body"
| fields timestamp, body[transactionId], body[amount], body[status]
```
