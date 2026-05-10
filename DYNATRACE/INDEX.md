# DYNATRACE NOTES — INDEX

## Notes (deep reference)

| File | Open when you need to... |
|---|---|
| [CONCEPTS.md](notes/CONCEPTS.md) | Understand core entities, Davis AI, data model, PurePath |
| [ARCHITECTURE.md](notes/ARCHITECTURE.md) | Deployment modes, OneAgent install, ActiveGate, network requirements |
| [DQL.md](notes/DQL.md) | Write Dynatrace Query Language — logs, metrics, entities, patterns |
| [ALERTING.md](notes/ALERTING.md) | Configure Davis AI, anomaly detection, alerting profiles, notifications |
| [METRICS.md](notes/METRICS.md) | Ingest custom metrics, MINT format, OTLP, Prometheus scraping |
| [LOGS.md](notes/LOGS.md) | Log ingestion, processing rules, log viewer, log DQL |
| [KUBERNETES.md](notes/KUBERNETES.md) | Operator install, DynaKube CR, CRDs, pod injection, K8s coverage |
| [API.md](notes/API.md) | REST API reference, token scopes, common endpoints, rate limits |
| [SLOs.md](notes/SLOs.md) | Create SLOs, error budget, burn rate alerts |
| [DASHBOARDS.md](notes/DASHBOARDS.md) | Build dashboards, tile types, variables, Notebooks |
| [TROUBLESHOOTING.md](notes/TROUBLESHOOTING.md) | Debug OneAgent, missing data, problems, K8s injection, DQL errors |
| [OPS.md](notes/OPS.md) | Day-2 ops, tagging strategy, management zones, deployment events |

## Cards (fast access — open these daily)

| File | Open when you need to... |
|---|---|
| [cards/CHEATSHEET.md](cards/CHEATSHEET.md) | Any API call or UI navigation path |
| [cards/DQL-QUICKREF.md](cards/DQL-QUICKREF.md) | Paste-ready DQL for logs, metrics, entities |
| [cards/INCIDENT.md](cards/INCIDENT.md) | Incident triage and investigation runbook |
| [cards/METRIC-INGEST.md](cards/METRIC-INGEST.md) | Push a custom metric right now |
| [cards/TAGGING.md](cards/TAGGING.md) | Tag entities, write entity selectors, set up management zones |
| [cards/DEPLOYMENT-EVENT.md](cards/DEPLOYMENT-EVENT.md) | Push a deployment marker from CI/CD |
| [cards/DEBUG-NODATA.md](cards/DEBUG-NODATA.md) | Debug missing service, metric, log, or DQL result |

---

## Situation → file map

| I'm doing... | Open... |
|---|---|
| First day, learning Dynatrace | [CONCEPTS.md](notes/CONCEPTS.md) → [ARCHITECTURE.md](notes/ARCHITECTURE.md) |
| Installing OneAgent on a host | [ARCHITECTURE.md](notes/ARCHITECTURE.md) |
| Setting up K8s monitoring | [KUBERNETES.md](notes/KUBERNETES.md) |
| Writing a DQL query | [DQL.md](notes/DQL.md) or [cards/DQL-QUICKREF.md](cards/DQL-QUICKREF.md) |
| Pushing a custom metric | [cards/METRIC-INGEST.md](cards/METRIC-INGEST.md) + [METRICS.md](notes/METRICS.md) |
| Ingesting logs | [LOGS.md](notes/LOGS.md) |
| Setting up alerts | [ALERTING.md](notes/ALERTING.md) |
| Creating SLOs | [SLOs.md](notes/SLOs.md) |
| Building a dashboard | [DASHBOARDS.md](notes/DASHBOARDS.md) |
| Pushing a deployment marker | [cards/DEPLOYMENT-EVENT.md](cards/DEPLOYMENT-EVENT.md) |
| Tagging entities | [cards/TAGGING.md](cards/TAGGING.md) |
| Service missing in Dynatrace | [cards/DEBUG-NODATA.md](cards/DEBUG-NODATA.md) |
| Metric / log not appearing | [cards/DEBUG-NODATA.md](cards/DEBUG-NODATA.md) |
| Incident in progress | [cards/INCIDENT.md](cards/INCIDENT.md) |
| Problem won't close | [TROUBLESHOOTING.md](notes/TROUBLESHOOTING.md) |
| K8s pods not instrumented | [TROUBLESHOOTING.md](notes/TROUBLESHOOTING.md) + [KUBERNETES.md](notes/KUBERNETES.md) |
| Need a curl command | [cards/CHEATSHEET.md](cards/CHEATSHEET.md) |
| API token scopes | [API.md](notes/API.md) |
| Understanding a Problem card | [ALERTING.md](notes/ALERTING.md) + [cards/INCIDENT.md](cards/INCIDENT.md) |
| Day-to-day ops tasks | [OPS.md](notes/OPS.md) |
