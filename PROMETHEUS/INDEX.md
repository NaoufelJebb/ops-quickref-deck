# PROMETHEUS NOTES — INDEX

## Notes (deep reference)

| File | Open when you need to... |
|---|---|
| [CONCEPTS.md](./notes/CONCEPTS.md) | Understand the data model, metric types, components, exposition format |
| [SETUP.md](./notes/SETUP.md) | Install with Docker, write prometheus.yml, configure scrape + service discovery, relabeling |
| [PROMQL.md](./notes/PROMQL.md) | Write PromQL — selectors, functions, aggregations, histogram quantiles, SLO queries |
| [ALERTING.md](./notes/ALERTING.md) | Write alert rules, configure Alertmanager routing, inhibition, silences |
| [OPERATOR.md](./notes/OPERATOR.md) | kube-prometheus-stack Helm install, ServiceMonitor, PodMonitor, PrometheusRule, AlertmanagerConfig |
| [FEDERATION-AND-SCALING.md](./notes/FEDERATION-AND-SCALING.md) | Federation config, remote write, Thanos sidecar/receive, Cortex/Mimir, recording rules, HA |
| [OPS-AND-TROUBLESHOOTING.md](./notes/OPS-AND-TROUBLESHOOTING.md) | Common exporters, cardinality, performance tuning, debug targets/alerts, ops checklist |

## Cards (fast access)

| File | Open when you need to... |
|---|---|
| [CHEATSHEET.md](./cards/CHEATSHEET.md) | Any Prometheus/Alertmanager API call, promtool, amtool, kubectl for operator |
| [PROMQL-QUICKREF.md](./cards/PROMQL-QUICKREF.md) | Paste-ready PromQL — RED, USE, K8s, SLO, cardinality |
| [INCIDENT.md](./cards/INCIDENT.md) | Alerts not firing, targets down, high memory, Alertmanager not delivering |

---

## Situation → file map

| I'm doing... | Open... |
|---|---|
| First day, learning Prometheus | [CONCEPTS.md](./notes/CONCEPTS.md) → [SETUP.md](./notes/SETUP.md) |
| Writing a scrape config | [SETUP.md](./notes/SETUP.md) |
| Writing a PromQL query | [PROMQL-QUICKREF.md](./cards/PROMQL-QUICKREF.md) + [PROMQL.md](./notes/PROMQL.md) |
| Writing alert rules | [ALERTING.md](./notes/ALERTING.md) |
| Configuring Alertmanager routing | [ALERTING.md](./notes/ALERTING.md) |
| Deploying on Kubernetes | [OPERATOR.md](./notes/OPERATOR.md) |
| Adding a ServiceMonitor | [OPERATOR.md](./notes/OPERATOR.md) → ServiceMonitor section |
| ServiceMonitor not picking up targets | [INCIDENT.md](./cards/INCIDENT.md) + [OPERATOR.md](./notes/OPERATOR.md) |
| Setting up federation | [FEDERATION-AND-SCALING.md](./notes/FEDERATION-AND-SCALING.md) → Federation section |
| Setting up remote write / Thanos | [FEDERATION-AND-SCALING.md](./notes/FEDERATION-AND-SCALING.md) |
| Alerts not firing | [INCIDENT.md](./cards/INCIDENT.md) |
| Alertmanager not delivering | [INCIDENT.md](./cards/INCIDENT.md) |
| Prometheus high memory / slow | [INCIDENT.md](./cards/INCIDENT.md) + [OPS-AND-TROUBLESHOOTING.md](./notes/OPS-AND-TROUBLESHOOTING.md) |
| Cardinality explosion | [OPS-AND-TROUBLESHOOTING.md](./notes/OPS-AND-TROUBLESHOOTING.md) → Cardinality section |
| Any API call or CLI command | [CHEATSHEET.md](./cards/CHEATSHEET.md) |
| Setting up blackbox / uptime checks | [OPS-AND-TROUBLESHOOTING.md](./notes/OPS-AND-TROUBLESHOOTING.md) → Blackbox exporter |
