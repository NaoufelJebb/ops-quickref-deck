# DEBUG — NO DATA

Use this when a metric, log, service, or entity is not appearing in Dynatrace.

## Service not appearing

```
1. Is OneAgent running on the host?
   systemctl status oneagent

2. Is the technology enabled?
   Settings → Monitoring → Monitored technologies

3. Has traffic hit the service? (Services only appear after first request)

4. Is the process excluded?
   Settings → Processes and containers → Process group detection → exclusion rules

5. K8s only: is the pod injected?
   kubectl describe pod <name> | grep -i dynatrace
```

---

## Metric exists but shows no data

```bash
# 1. Confirm the metric descriptor exists
curl "$DT_ENV/api/v2/metrics/builtin:service.response.time" \
  -H "Authorization: Api-Token $DT_TOKEN" | jq '{key: .metricId, lastWritten: .lastWritten}'

# 2. Entity active? (lastSeenTms should be recent)
curl "$DT_ENV/api/v2/entities/SERVICE-XXX" \
  -H "Authorization: Api-Token $DT_TOKEN" | jq '.lastSeenTms'
```

Common causes:
- **No traffic** — metric only exists when requests flow
- **Resolution too fine** — try `resolution=5m` or `resolution=auto`
- **Time range** — entity didn't exist in the queried period
- **Wrong entity** — double-check entity ID

---

## Custom metric not appearing after ingest

```bash
# Test — check HTTP response (should be 202)
curl -v -X POST "$DT_ENV/api/v2/metrics/ingest" \
  -H "Authorization: Api-Token $DT_TOKEN" \
  -H "Content-Type: text/plain; charset=utf-8" \
  -d "test.debug.metric,env=test 1.0"

# Confirm descriptor was created (wait 30s after first ingest)
curl "$DT_ENV/api/v2/metrics/test.debug.metric" \
  -H "Authorization: Api-Token $DT_TOKEN"
```

| Response | Cause |
|---|---|
| `401` | Token missing `metrics.ingest` scope |
| `400` | Malformed MINT line — check format |
| `202` but no data | Dimension cardinality > 50 unique values |
| `202` but no data | Wrong Content-Type — must be `text/plain; charset=utf-8` |

---

## Logs not appearing

```bash
# Test ingest (expect 204)
curl -v -X POST "$DT_ENV/api/v2/logs/ingest" \
  -H "Authorization: Api-Token $DT_TOKEN" \
  -H "Content-Type: application/json" \
  -d '[{"content":"debug test","severity":"INFO"}]'
```

| Response | Cause |
|---|---|
| `401` | Token missing `logs.ingest` scope |
| `413` | Payload > 1MB — split into smaller batches |
| `429` | Rate limited — reduce frequency |
| `204` but no logs | Check log storage rules — log may not be indexed |

---

## DQL returns nothing

```dql
-- Step 1: try with no filters
fetch logs | limit 5

-- Step 2: widen the time window
fetch logs, from: now()-24h | limit 5

-- Step 3: check field names (case-sensitive)
fetch logs | fields * | limit 1

-- Step 4: verify entity ID
fetch dt.entity.service
| filter entity.name == "my-service"
| fields entityId
```

---

## OneAgent connected but host shows no data

```bash
/opt/dynatrace/oneagent/agent/tools/oneagentctl --get-server
journalctl -u oneagent --since "10 minutes ago" | grep -E "error|warn"
```

- Token revoked or expired → regenerate and update
- Connectivity lost → check firewall, DNS, proxy settings
- Agent update in progress → wait 2–3 minutes
