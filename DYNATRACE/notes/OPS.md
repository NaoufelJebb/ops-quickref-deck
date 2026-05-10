# OPS

## Daily health checks

```bash
# 1. Open problems
curl "$DT_ENV/api/v2/problems?problemSelector=status(open)" \
  -H "Authorization: Api-Token $DT_TOKEN" | jq '{total: .totalCount, problems: [.problems[] | {title, severity, status}]}'

# 2. OneAgent host connectivity — how many are disconnected?
curl "$DT_ENV/api/v2/entities?entitySelector=type(HOST),agentStatus(NOT_ACTIVE)" \
  -H "Authorization: Api-Token $DT_TOKEN" | jq '.totalCount'

# 3. SLO compliance
curl "$DT_ENV/api/v2/slo" \
  -H "Authorization: Api-Token $DT_TOKEN" | jq '.slo[] | {name, status, evaluatedPercentage, target}'
```

---

## Tagging strategy

Tags are the foundation of scoping, filtering, and management zones. Define a convention and enforce it.

### Recommended tag schema

```
env:prod / env:staging / env:dev
team:platform / team:checkout / team:payments
app:my-service
version:2.4.1
criticality:high / criticality:low
```

### Apply tags via API

```bash
# Tag a service
curl -X POST "$DT_ENV/api/v2/tags" \
  -H "Authorization: Api-Token $DT_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
    "entitySelector": "type(SERVICE),entityName(\"my-service\")",
    "tags": [
      {"key": "env", "value": "prod"},
      {"key": "team", "value": "checkout"}
    ]
  }'
```

### Auto-tagging rules

`Settings → Tags → Automatically applied tags`

```
Rule name:   Environment from Kubernetes label
Conditions:
  - Kubernetes label "env" exists
Tag:         env:{Kubernetes label: env}
```

---

## Management zones

Scope entities by team or environment so each team only sees their services.

`Settings → Management zones → Add management zone`

```
Name: Production — Checkout Team
Rules:
  - Type: Service, condition: tag("team:checkout") AND tag("env:prod")
  - Type: Host, condition: tag("team:checkout")
  - Type: Process group, condition: tag("team:checkout")
```

Apply to:
- Dashboards (auto-filter)
- Alerting profiles (scope notifications)
- User permissions (restrict visibility)
- SLOs

---

## Deployment events — mark releases in Dynatrace

Push a deployment event on every release so Davis can correlate problems with deploys.

```bash
# Push deployment event (add to CI/CD pipeline)
curl -X POST "$DT_ENV/api/v2/events/ingest" \
  -H "Authorization: Api-Token $DT_TOKEN" \
  -H "Content-Type: application/json" \
  -d "{
    \"eventType\": \"CUSTOM_DEPLOYMENT\",
    \"title\": \"Deploy $SERVICE v$VERSION\",
    \"entitySelector\": \"type(SERVICE),tag(\\\"app:$SERVICE\\\")\",
    \"properties\": {
      \"deploymentVersion\": \"$VERSION\",
      \"deployedBy\": \"$CI_USER\",
      \"ciLink\": \"$CI_BUILD_URL\"
    }
  }"
```

---

## Maintenance windows — planned operations

```bash
# Create via API
curl -X POST "$DT_ENV/api/config/v1/maintenanceWindows" \
  -H "Authorization: Api-Token $DT_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
    "name": "Prod deployment window",
    "description": "Weekly deployment window",
    "type": "PLANNED",
    "suppression": "DONT_DETECT_PROBLEMS",
    "schedule": {
      "recurrenceType": "WEEKLY",
      "recurrence": {
        "dayOfWeek": "FRIDAY",
        "startTime": "22:00",
        "durationMinutes": 120
      },
      "start": "2024-01-05 22:00",
      "end": "2025-12-31 23:59",
      "zoneId": "Europe/Paris"
    },
    "scope": {
      "entities": [],
      "matches": [{"type": "SERVICE", "tags": [{"key": "env", "value": "prod"}]}]
    }
  }'
```

---

## Problem management

```bash
# Get all open problems with detail
curl "$DT_ENV/api/v2/problems?problemSelector=status(open)&fields=+evidenceDetails,+recentComments" \
  -H "Authorization: Api-Token $DT_TOKEN" | jq .

# Add a comment (document investigation)
curl -X POST "$DT_ENV/api/v2/problems/PROBLEM-XXXXXXXX/comments" \
  -H "Authorization: Api-Token $DT_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{"message": "Root cause identified: Redis connection pool exhausted. Fix deployed.", "context": "ops"}'

# Close a problem manually (if Davis doesn't auto-close)
curl -X DELETE "$DT_ENV/api/v2/problems/PROBLEM-XXXXXXXX" \
  -H "Authorization: Api-Token $DT_TOKEN"
```

---

## Ops checklist

- [ ] Open problems reviewed daily — comment and assign
- [ ] Disconnected OneAgent hosts checked — investigate if count rises
- [ ] SLO compliance checked weekly — error budget trending
- [ ] Deployment events pushed on every release — Davis correlates correctly
- [ ] Maintenance windows created for all planned changes
- [ ] Management zones reviewed quarterly — new services tagged and included
- [ ] API tokens audited quarterly — unused tokens revoked
- [ ] Dashboard reviewed after major incidents — add coverage gaps
