# INCIDENT

## 1. Triage — open Problems

```bash
# What's broken right now?
curl "$DT_ENV/api/v2/problems?problemSelector=status(open)" \
  -H "Authorization: Api-Token $DT_TOKEN" \
  | jq '.problems[] | {title, severity, affectedEntities, startTime}'
```

Or: **Menu → Problems** — Davis shows root cause entity and impact.

---

## 2. Understand the Problem card

```
Problem card contains:
├── Root cause — the entity Davis identified as source
├── Impact — downstream services/users affected
├── Events timeline — what changed before the problem
├── Deployment markers — was there a recent release?
└── Davis analysis — read this first before investigating manually
```

---

## 3. Investigate with DQL

```dql
// Spike in errors — which service?
fetch logs, from: now()-30m
| filter loglevel == "ERROR"
| summarize errors = count(), by: {dt.entity.service}
| sort errors desc

// What do the errors say?
fetch logs, from: now()-30m
| filter loglevel == "ERROR" and dt.entity.service == "SERVICE-XXX"
| fields timestamp, content
| sort timestamp desc | limit 20

// Latency spike?
fetch metrics, from: now()-1h
| filter metric.key == "builtin:service.response.time"
  and dt.entity.service == "SERVICE-XXX"
| summarize p99 = percentile(value, 99), by: {bin(timestamp, 1m)}
| sort timestamp asc
```

---

## 4. Check for recent deployments

```bash
# Was there a deploy just before the problem started?
curl "$DT_ENV/api/v2/events?eventSelector=type(\"CUSTOM_DEPLOYMENT\")&from=now-2h" \
  -H "Authorization: Api-Token $DT_TOKEN" | jq '.events[] | {title, timestamp}'
```

Or: look for deployment markers on the timeline in the Problem card.

---

## 5. Mitigate

```bash
# Comment on the problem to document actions
curl -X POST "$DT_ENV/api/v2/problems/PROBLEM-XXX/comments" \
  -H "Authorization: Api-Token $DT_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{"message":"Root cause: Redis timeout. Rolling back deploy v2.4.0","context":"ops"}'
```

Mitigation actions are outside Dynatrace (rollback, restart, scale up) — Dynatrace tells you *what* and *where*, not *how to fix it*.

---

## 6. Create maintenance window if needed

Suppress further alerts during remediation:

`Settings → Maintenance windows → Add`

---

## 7. Confirm resolution

- Problem auto-closes when the anomaly clears — watch the Problem card
- If it doesn't close: `curl -X DELETE "$DT_ENV/api/v2/problems/PROBLEM-XXX"`
- Post-incident: add a dashboard section showing the incident window

---

## Problem never resolves? Check

- Is the anomaly actually gone? Check the metric in Data explorer
- Is the sensitivity set too high? Lower it in anomaly detection settings
- Is a maintenance window masking a real issue?
- Davis may be waiting for enough data points — give it 5 minutes after fix
