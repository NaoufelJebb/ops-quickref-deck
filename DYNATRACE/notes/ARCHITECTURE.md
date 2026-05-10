# ARCHITECTURE

## Deployment models

| Model | Description | Best for |
|---|---|---|
| **SaaS (DPS)** | Fully hosted by Dynatrace. Grail lakehouse. You manage agents only. | Most teams — zero infra overhead |
| **Managed** | Dynatrace software runs on your own servers. Full data control. | Air-gapped / strict data residency |
| **Hybrid** | Managed cluster + SaaS features via Mission Control | Legacy managed moving to SaaS |

## Component map

```
Your environment                    Dynatrace Platform
┌────────────────────────────┐      ┌─────────────────────────┐
│  Hosts / Containers / K8s  │      │  SaaS Tenant            │
│  ┌─────────────────┐       │      │  - Smartscape topology  │
│  │   OneAgent      │──────────────▶  - Davis AI             │
│  └─────────────────┘       │      │  - Dashboards           │
│                            │      │  - Problems             │
│  ┌─────────────────┐       │      │  - Grail (DQL)          │
│  │   ActiveGate    │──────────────▶  - APIs                 │
│  └─────────────────┘       │      └─────────────────────────┘
└────────────────────────────┘
```

## OneAgent

- **One agent per host** — auto-instruments all processes, no restart usually needed
- Communicates outbound only (HTTPS 443) — no inbound firewall rules needed
- Updates automatically (configurable)
- Modes: `full-stack` (default), `infra-only` (no APM), `cloud-native` (K8s sidecar/operator)

### Install (Linux)

```bash
# Download from: Settings → Deployment status → Deploy OneAgent
wget -O Dynatrace-OneAgent.sh "<installer-url>"
sudo /bin/sh Dynatrace-OneAgent.sh \
  --set-server=https://<env-id>.live.dynatrace.com/communication \
  --set-tenant=<env-id> \
  --set-tenant-token=<token>
```

### Check OneAgent status

```bash
/opt/dynatrace/oneagent/agent/tools/oneagentctl --get-server
systemctl status oneagent
journalctl -u oneagent -f
```

## ActiveGate

A proxy/relay between your environment and the Dynatrace platform.

| Use case | Required? |
|---|---|
| Routing OneAgent traffic through a single egress point | Optional but common |
| Monitoring remote environments, cloud APIs | Required |
| Synthetic private locations | Required |
| Extension execution (EEC) | Required |
| Managed deployments | Required |

### Install (Linux)

```bash
wget -O Dynatrace-ActiveGate.sh "<activegate-installer-url>"
sudo /bin/sh Dynatrace-ActiveGate.sh
```

## Kubernetes architecture

```
K8s cluster
├── dynatrace namespace
│   ├── dynakube (DynaKube CRD — defines monitoring config)
│   ├── dynatrace-operator
│   ├── activegate (for API monitoring, routing)
│   └── oneagent (DaemonSet — one pod per node)
└── your workloads (auto-instrumented via webhook injection)
```

Operator manages the full lifecycle. See KUBERNETES.md.

## Network requirements

| Destination | Port | Purpose |
|---|---|---|
| `*.live.dynatrace.com` | 443 | SaaS — OneAgent and ActiveGate communication |
| `*.dynatracelabs.com` | 443 | SaaS — alternative endpoints |
| Your ActiveGate | 443 | If routing through ActiveGate |

Outbound only. OneAgent never accepts inbound connections.
