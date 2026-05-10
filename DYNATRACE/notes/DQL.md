# DQL

Dynatrace Query Language — used in Notebooks, Dashboards, Log viewer, and the API.

## Structure

```
fetch <data-type>
| filter <condition>
| summarize <aggregation>
| sort <field> [desc]
| limit <n>
```

## Data types

```
fetch logs
fetch metrics
fetch events
fetch bizevents
fetch dt.entity.host
fetch dt.entity.service
fetch dt.entity.process_group
fetch dt.entity.application
```

---

## Logs

```dql
// All logs from a service, last 1 hour
fetch logs
| filter matchesPhrase(content, "payment") and loglevel == "ERROR"
| fields timestamp, loglevel, content, dt.entity.service
| sort timestamp desc
| limit 100

// Error rate by service
fetch logs
| filter loglevel == "ERROR"
| summarize errors = count(), by: {service = dt.entity.service}
| sort errors desc

// Logs from a specific host
fetch logs
| filter dt.entity.host == "HOST-XXXXXXXX"
| fields timestamp, loglevel, content
| sort timestamp desc

// Keyword search with context
fetch logs
| filter matchesPhrase(content, "OutOfMemoryError")
| fields timestamp, content, dt.entity.host, dt.entity.process_group_instance
| sort timestamp desc

// Log volume over time
fetch logs
| summarize count = count(), by: {bin(timestamp, 5m)}
| sort timestamp asc
```

---

## Metrics

```dql
// Service response time — P95
fetch metrics
| filter metric.key == "builtin:service.response.time"
| summarize p95 = percentile(value, 95), by: {entity = dt.entity.service}
| sort p95 desc

// Host CPU usage
fetch metrics
| filter metric.key == "builtin:host.cpu.usage"
| summarize avg_cpu = avg(value), by: {host = dt.entity.host}
| sort avg_cpu desc

// Error rate per service
fetch metrics
| filter metric.key == "builtin:service.errors.total.rate"
| summarize avg = avg(value), by: {dt.entity.service}

// Custom metric
fetch metrics
| filter metric.key == "ext:my.custom.metric"
| summarize sum = sum(value), by: {bin(timestamp, 1m)}

// Metric over time (for charting)
fetch metrics
| filter metric.key == "builtin:service.response.time"
  and dt.entity.service == "SERVICE-XXXXXXXX"
| summarize avg = avg(value), by: {bin(timestamp, 5m)}
| sort timestamp asc
```

---

## Entities

```dql
// List all services with their names
fetch dt.entity.service
| fields entity.name, entityId
| sort entity.name asc

// Hosts with low memory
fetch dt.entity.host
| fields entity.name, memoryTotal, memoryUsed = memoryAvailable
| filter memoryTotal > 0

// Services tagged with a specific tag
fetch dt.entity.service
| filter in(entity.name, array("service-a", "service-b"))
| fields entity.name, entityId
```

---

## Events

```dql
// Deployment events in the last 24h
fetch events
| filter event.type == "CUSTOM_DEPLOYMENT"
| fields timestamp, event.name, event.provider, dt.entity.service
| sort timestamp desc

// All problem events
fetch events
| filter event.type == "DAVIS_PROBLEM"
| fields timestamp, event.name, status
| sort timestamp desc
```

---

## DQL patterns

```dql
// Time range (default is last 2h; override with from/to)
fetch logs, from: now()-24h, to: now()

// Conditional field
fetch logs
| fieldsAdd severity = if(loglevel == "ERROR", "high", else: "low")

// Parse a JSON field
fetch logs
| parse content, "JSON:parsed"
| fields timestamp, parsed[status], parsed[userId]

// String contains
fetch logs
| filter contains(content, "timeout")

// Regex match
fetch logs
| filter matchesPhrase(content, "exception") and
  content =~ ".*NullPointerException.*"

// Join — enrich logs with entity name
fetch logs
| lookup [fetch dt.entity.service | fields entityId, entity.name],
    sourceField: dt.entity.service,
    lookupField: entityId
| fields timestamp, content, entity.name
```

---

## Common built-in metric keys

```
builtin:service.response.time              ms — P50/P90/P95/P99 available
builtin:service.errors.total.rate          % error rate
builtin:service.requestCount.total         requests/min
builtin:host.cpu.usage                     % CPU
builtin:host.mem.usage                     % memory
builtin:host.disk.usedPct                  % disk
builtin:host.net.nic.trafficIn             bytes/s
builtin:host.net.nic.trafficOut            bytes/s
builtin:containers.cpu.usagePercent        container CPU
builtin:containers.memory.usagePercent     container memory
builtin:kubernetes.node.cpu.requests       K8s node CPU requested
builtin:kubernetes.workload.pods.running   running pod count
```
