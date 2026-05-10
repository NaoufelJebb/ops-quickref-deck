# DQL QUICK REFERENCE

## Logs

```dql
// All errors, last hour
fetch logs, from: now()-1h
| filter loglevel == "ERROR"
| fields timestamp, content, dt.entity.service
| sort timestamp desc
| limit 100

// Keyword search
fetch logs
| filter matchesPhrase(content, "timeout")
| sort timestamp desc | limit 50

// Error count by service
fetch logs
| filter loglevel == "ERROR"
| summarize errors = count(), by: {dt.entity.service}
| sort errors desc

// Log volume over time (for chart)
fetch logs
| summarize count = count(), by: {bin(timestamp, 5m)}
| sort timestamp asc

// Parse JSON log content
fetch logs
| filter matchesPhrase(content, "payment")
| parse content, "JSON:body"
| fields timestamp, body[transactionId], body[status]
```

## Metrics

```dql
// Service response time P95
fetch metrics
| filter metric.key == "builtin:service.response.time"
| summarize p95 = percentile(value, 95), by: {dt.entity.service}
| sort p95 desc

// Host CPU over time
fetch metrics
| filter metric.key == "builtin:host.cpu.usage"
  and dt.entity.host == "HOST-XXXXXXXX"
| summarize avg = avg(value), by: {bin(timestamp, 5m)}
| sort timestamp asc

// Error rate per service
fetch metrics
| filter metric.key == "builtin:service.errors.total.rate"
| summarize avg = avg(value), by: {dt.entity.service}
| sort avg desc
```

## Entities

```dql
// All services
fetch dt.entity.service
| fields entity.name, entityId
| sort entity.name asc

// Services with a specific tag
fetch dt.entity.service
| filter in(entity.name, array("service-a", "service-b"))
| fields entity.name, entityId
```

## Events

```dql
// Deployment events last 24h
fetch events, from: now()-24h
| filter event.type == "CUSTOM_DEPLOYMENT"
| fields timestamp, event.name, dt.entity.service
| sort timestamp desc
```

## DQL syntax reminders

```dql
from: now()-1h                    -- time range
from: now()-24h, to: now()-1h     -- specific window
| limit 100                       -- cap results
| sort field desc                 -- sort descending
| fields f1, f2                   -- select fields
| fields *                        -- all fields (debug)
| filter a == "x" and b > 100
| summarize count(), by: {field}
| summarize avg = avg(value), by: {bin(timestamp, 5m)}
| fieldsAdd new_col = expression
matchesPhrase(content, "keyword")  -- full-text search
contains(field, "substring")       -- substring match
```
