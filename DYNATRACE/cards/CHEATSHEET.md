# CHEATSHEET

## Env setup

```bash
export DT_ENV=https://<env-id>.live.dynatrace.com
export DT_TOKEN=dt0c01.XXXXXX
```

## API — most-used calls

```bash
# ── PROBLEMS ──────────────────────────────────────────────────────────────────
curl "$DT_ENV/api/v2/problems?problemSelector=status(open)" \
  -H "Authorization: Api-Token $DT_TOKEN" | jq '.totalCount'

curl "$DT_ENV/api/v2/problems/PROBLEM-XXX" \
  -H "Authorization: Api-Token $DT_TOKEN"

curl -X POST "$DT_ENV/api/v2/problems/PROBLEM-XXX/comments" \
  -H "Authorization: Api-Token $DT_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{"message":"investigating","context":"ops"}'

# ── ENTITIES ──────────────────────────────────────────────────────────────────
curl "$DT_ENV/api/v2/entities?entitySelector=type(SERVICE)" \
  -H "Authorization: Api-Token $DT_TOKEN" | jq '.entities[] | {name: .displayName, id: .entityId}'

curl "$DT_ENV/api/v2/entities?entitySelector=type(HOST),agentStatus(NOT_ACTIVE)" \
  -H "Authorization: Api-Token $DT_TOKEN" | jq '.totalCount'

# ── METRICS ───────────────────────────────────────────────────────────────────
curl "$DT_ENV/api/v2/metrics/query" \
  -H "Authorization: Api-Token $DT_TOKEN" \
  -G --data-urlencode "metricSelector=builtin:service.response.time:avg:auto" \
     --data-urlencode "from=now-2h" \
     --data-urlencode "entitySelector=type(SERVICE)" | jq .

# Ingest custom metric
curl -X POST "$DT_ENV/api/v2/metrics/ingest" \
  -H "Authorization: Api-Token $DT_TOKEN" \
  -H "Content-Type: text/plain" \
  -d "my.metric,env=prod 42.0"

# ── LOGS ──────────────────────────────────────────────────────────────────────
curl "$DT_ENV/api/v2/logs/search?query=loglevel%3D%22ERROR%22&from=now-1h&limit=50" \
  -H "Authorization: Api-Token $DT_TOKEN" | jq '.results[] | {timestamp, content}'

# Ingest log
curl -X POST "$DT_ENV/api/v2/logs/ingest" \
  -H "Authorization: Api-Token $DT_TOKEN" \
  -H "Content-Type: application/json" \
  -d '[{"content":"my log message","severity":"ERROR","dt.entity.service":"SERVICE-XXX"}]'

# ── EVENTS ────────────────────────────────────────────────────────────────────
# Push deployment marker
curl -X POST "$DT_ENV/api/v2/events/ingest" \
  -H "Authorization: Api-Token $DT_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{"eventType":"CUSTOM_DEPLOYMENT","title":"Deploy v1.0","entitySelector":"type(SERVICE),tag(\"app:my-service\")"}'

# ── SLOs ──────────────────────────────────────────────────────────────────────
curl "$DT_ENV/api/v2/slo" \
  -H "Authorization: Api-Token $DT_TOKEN" | jq '.slo[] | {name, status, evaluatedPercentage, target}'

# ── TAGS ──────────────────────────────────────────────────────────────────────
curl -X POST "$DT_ENV/api/v2/tags" \
  -H "Authorization: Api-Token $DT_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{"entitySelector":"type(SERVICE),entityName(\"my-service\")","tags":[{"key":"env","value":"prod"}]}'
```

## Key UI paths

| Task | Navigation |
|---|---|
| Deploy OneAgent | Settings → Deployment status → Deploy OneAgent |
| Create API token | Settings → Access tokens |
| Anomaly detection | Settings → Anomaly detection |
| Alerting profiles | Settings → Alerting → Alerting profiles |
| Notifications | Settings → Alerting → Problem notifications |
| Maintenance windows | Settings → Maintenance windows |
| Log sources | Settings → Log Monitoring → Log sources |
| Auto-tagging | Settings → Tags → Automatically applied tags |
| Management zones | Settings → Management zones |
| SLOs | Observe & Explore → SLOs |
| Problems | Observe & Explore → Problems |
| Dashboards | Observe & Explore → Dashboards |
| Notebooks | Observe & Explore → Notebooks |
| Log viewer | Observe & Explore → Logs |
| Smartscape | Observe & Explore → Smartscape |
| Synthetic monitors | Digital Experience → Synthetic |
