# ALERTING

## How Davis AI works

Davis does not use static thresholds. It:

1. **Baselines** each metric automatically (learns normal behavior per hour, day, week)
2. **Detects anomalies** when behavior deviates from the baseline
3. **Correlates** related events into a single **Problem** card
4. **Root-causes** by traversing the Smartscape topology

You configure *sensitivity*, not thresholds.

---

## Problems

A Problem groups all related anomalies, affected entities, and root cause into one card.

```
Problem card contains:
├── Root cause entity
├── Affected services / hosts
├── Impact (users, requests affected)
├── Timeline of events
├── Davis AI analysis
└── Linked alerts / events
```

### Problem lifecycle

```
Open → Resolved (auto, when anomaly clears) → Closed
```

Problems resolve automatically. You don't close them manually.

---

## Anomaly detection settings

### Service-level

`Settings → Anomaly detection → Services`

| Setting | What it controls |
|---|---|
| Response time degradation | Sensitivity: low / medium / high / custom |
| Error rate increase | Threshold or auto-baseline |
| Throughput drop | % drop from baseline |
| Custom thresholds | Override auto-baseline for a specific service |

### Infrastructure-level

`Settings → Anomaly detection → Infrastructure`

- CPU saturation
- Memory saturation
- High network drop/error rate
- Disk space low

### Custom metric events

`Settings → Anomaly detection → Metric events`

Create threshold-based or auto-adaptive alerts on any metric (built-in or custom):

```
Metric:    ext:my.queue.depth
Condition: Average > 1000 for 5 minutes
Severity:  Performance
```

---

## Alerting profiles

Filter which Problems trigger notifications. Chain multiple profiles.

`Settings → Alerting → Alerting profiles`

Key filters:
- Problem severity (Availability, Error, Slowdown, Resource contention, Custom)
- Management zone scope
- Entity tags
- Delay: only alert if Problem is open for N minutes (reduces noise)

---

## Notification integrations

`Settings → Alerting → Problem notifications`

| Integration | Use for |
|---|---|
| **Email** | Simple team alerts |
| **PagerDuty** | On-call routing |
| **Slack** | #ops-alerts channel |
| **Webhook** | Any custom system |
| **ServiceNow** | ITSM ticket creation |
| **Jira** | Dev team issue creation |
| **OpsGenie** | On-call management |
| **MS Teams** | Team notifications |

### Webhook payload (custom integration)

```json
{
  "problemTitle": "{ProblemTitle}",
  "problemId": "{ProblemID}",
  "problemUrl": "{ProblemURL}",
  "state": "{State}",
  "severity": "{ProblemSeverity}",
  "affectedEntities": "{ProblemImpact}",
  "rootCauseEntity": "{ProblemRootCause}",
  "tags": "{Tags}",
  "timestamp": "{Timestamp}"
}
```

---

## Maintenance windows

Suppress alerts during planned maintenance.

`Settings → Maintenance windows`

```
Name:       Deployment window prod
Schedule:   Recurring, Friday 22:00–02:00
Type:       Planned maintenance (suppresses alerts, still monitors)
Scope:      Management zone: production
```

Types:
- **Planned maintenance** — alerts suppressed, problems still detected
- **Unplanned maintenance** — everything suppressed

---

## Metric events (custom alerting) — example

```
Name:         High queue depth
Metric:       ext:rabbitmq.queue.messages_ready
Aggregation:  Average
Condition:    Static threshold > 5000
Alert when:   Condition met for 3 of 5 minutes
Severity:     Resource contention
Management zone: production
```

---

## Reduce alert noise — quick checklist

- [ ] Set alerting profile with minimum Problem duration (e.g. 5 min delay) to filter flaps
- [ ] Scope notification channels to Management zones — teams only see their own services
- [ ] Use `low` sensitivity for non-critical services, `high` for critical paths
- [ ] Configure maintenance windows for all known deployment windows
- [ ] Review and close frequent false-positive Problems — Davis learns from feedback
- [ ] Merge related alerting profiles rather than creating many overlapping ones
