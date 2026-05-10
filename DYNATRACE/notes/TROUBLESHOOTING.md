# TROUBLESHOOTING

## OneAgent not reporting

```bash
# 1. Check OneAgent service
systemctl status oneagent
journalctl -u oneagent --since "30 minutes ago"

# 2. Verify connectivity to Dynatrace
/opt/dynatrace/oneagent/agent/tools/oneagentctl --get-server
curl -I https://<env-id>.live.dynatrace.com/communication

# 3. Check OneAgent version and status
/opt/dynatrace/oneagent/agent/tools/oneagentctl --get-watchdog-portrange

# 4. Restart if needed
systemctl restart oneagent
```

Common causes:

| Symptom | Likely cause |
|---|---|
| Host visible but no processes | Process not yet discovered — wait 5 min after start |
| No connectivity | Firewall blocking outbound 443 to `*.live.dynatrace.com` |
| "Disconnected" in UI | Token expired or revoked |
| Missing data after restart | OneAgent restarting — check `journalctl -u oneagent` |

---

## Service not appearing in Dynatrace

```
1. Confirm OneAgent is injected into the process/container
2. Check the technology is supported and enabled:
   Settings → Monitoring → Monitored technologies
3. Check if excluded via process group exclusion rule:
   Settings → Processes and containers → Process group detection
4. For K8s: verify pod has injection annotation or namespace is not excluded
5. Wait 5–10 minutes after first request — service detection is not instant
```

---

## No data / gaps in metrics

```bash
# Check if the metric exists at all
curl "$DT_ENV/api/v2/metrics/builtin:service.response.time" \
  -H "Authorization: Api-Token $DT_TOKEN" | jq .

# Check entity exists and is active
curl "$DT_ENV/api/v2/entities/SERVICE-XXXXXXXX" \
  -H "Authorization: Api-Token $DT_TOKEN" | jq '.lastSeenTms'
```

Common causes:
- **No traffic** — metric only appears when there are requests
- **Entity stale** — entity went offline; data stops after last activity
- **Metric resolution** — querying at too fine a resolution returns no data; try `resolution=5m`
- **Time range** — entity or metric may not have existed in the queried range

---

## Custom metrics not appearing

```bash
# Test ingest — check HTTP response code (should be 202)
curl -v -X POST "$DT_ENV/api/v2/metrics/ingest" \
  -H "Authorization: Api-Token $DT_TOKEN" \
  -H "Content-Type: text/plain" \
  -d "test.metric,env=debug 1.0"

# Check the metric descriptor was created
curl "$DT_ENV/api/v2/metrics/test.metric" \
  -H "Authorization: Api-Token $DT_TOKEN"
```

Common causes:
- `401` — token missing `metrics.ingest` scope
- `400` — malformed MINT line (check format: `key,dim=val value`)
- Metric appears but shows no data — dimension cardinality too high (max 50 values per dim key)
- Wrong Content-Type — must be `text/plain; charset=utf-8`

---

## Problem not triggering expected alert

```
1. Check alerting profile — is the Problem severity/type included?
   Settings → Alerting → Alerting profiles → review filters
2. Check if Problem is in a maintenance window
   Settings → Maintenance windows
3. Check Management zone scope — does the notification channel have the right MZ?
4. Check notification integration itself:
   Settings → Alerting → Problem notifications → test the integration
5. Check Problem delay setting — alerting profile may require Problem to be open N min
```

---

## DQL query returns no results

```dql
-- Debug: remove filters one by one to find the blocking condition
fetch logs                    -- start with no filters
| limit 5                     -- verify data exists at all

-- Check your time range
fetch logs, from: now()-24h   -- expand the window

-- Check field names — case sensitive
fetch logs
| fields *                    -- see all available fields

-- Verify entity ID is correct
fetch dt.entity.service
| filter entity.name == "my-service"
| fields entityId             -- copy the ID, use in filter
```

---

## K8s: pods not instrumented

```bash
# 1. Check operator is running
kubectl get pods -n dynatrace

# 2. Check webhook is active
kubectl get mutatingwebhookconfigurations | grep dynatrace

# 3. Check DynaKube status for errors
kubectl describe dynakube dynakube -n dynatrace | grep -A 20 Status

# 4. Check if namespace is excluded
kubectl get namespace my-namespace -o yaml | grep -i inject

# 5. Check pod for injection evidence
kubectl describe pod <pod-name> | grep -i dynatrace
```

Common causes:
- Namespace not labeled for monitoring (if `namespaceSelector` is set in DynaKube)
- Pod has `oneagent.dynatrace.com/inject: "false"` annotation
- Operator webhook not running — operator pod crashed
- `cloudNativeFullStack` CSI driver not installed

---

## Log ingestion not working

```bash
# Test the ingest endpoint directly
curl -v -X POST "$DT_ENV/api/v2/logs/ingest" \
  -H "Authorization: Api-Token $DT_TOKEN" \
  -H "Content-Type: application/json" \
  -d '[{"content":"test log message","severity":"INFO"}]'
# Expect: 204 No Content
```

- `401` — token missing `logs.ingest` scope
- `413` — payload too large (max 1MB per request)
- `429` — rate limited — batch logs and reduce frequency
- Logs ingested but not visible — check storage rules and index settings
