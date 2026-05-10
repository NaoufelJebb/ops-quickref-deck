# CONCEPTS

## What Dynatrace is

Dynatrace is a full-stack observability and AIOps platform. It automatically discovers, instruments, and monitors your entire environment — infrastructure, applications, services, user experience — and uses AI (Davis) to detect anomalies and root-cause problems without manual configuration.

```
Browser / Mobile → Frontend (RUM) → Services → Infrastructure
                       ↓               ↓              ↓
                   [ Dynatrace OneAgent — auto-instrumented ]
                       ↓
                 [ Dynatrace Platform ]
                       ↓
               Davis AI — anomaly detection,
                          root cause analysis,
                          auto-baselining
```

## Core entities

| Entity | What it is |
|---|---|
| **Host** | A physical or virtual machine running OneAgent |
| **Process Group** | A set of similar processes (e.g. all instances of `my-service`) |
| **Service** | An auto-detected logical service (REST API, gRPC, DB calls) |
| **Application** | A frontend monitored via RUM (web or mobile) |
| **Synthetic Monitor** | Scheduled HTTP or browser test |
| **Problem** | An AI-detected anomaly — groups related events into one actionable item |
| **Management Zone** | A filtered view scoping entities by team, environment, or tag |
| **SLO** | A Service Level Objective — measured availability or performance target |

## Data model pillars

| Pillar | What it covers |
|---|---|
| **Metrics** | Time-series numbers — CPU, response time, error rate, custom |
| **Logs** | Ingested log lines, parsed and indexed |
| **Traces** | Distributed traces — PurePath technology |
| **Events** | Deployment markers, config changes, custom annotations |
| **Topology** | The Smartscape — entity relationships auto-mapped |

## Key concepts

**OneAgent** — a single agent deployed per host that auto-instruments everything: JVM, Node.js, .NET, Go, PHP, Nginx, Kubernetes, databases. No code changes needed.

**PurePath** — Dynatrace's distributed tracing. Every transaction is captured end-to-end, from browser click to database query, with full context.

**Davis AI** — the anomaly detection and root-cause engine. Baselines automatically, correlates events, surfaces a single Problem card instead of flooding you with alerts.

**Smartscape** — real-time topology map. Shows relationships between hosts, processes, services, and applications automatically.

**Grail** — the unified data lakehouse (DPS/SaaS). Stores metrics, logs, traces, events in one queryable store via DQL.

**DQL** — Dynatrace Query Language. Used in Notebooks, Dashboards, and Log viewer to query Grail data.
