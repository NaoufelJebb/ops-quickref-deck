# ARCHITECTURE

## Deployment modes

| Mode | Config storage | Best for |
|---|---|---|
| **DB mode** | PostgreSQL / Cassandra | Mutable, live config changes |
| **DB-less** | YAML file, loaded at startup | GitOps, immutable infra |
| **Hybrid** ⭐ | CP holds DB, DPs hold nothing | Production — secure separation |
| **Konnect** | SaaS CP, on-prem DPs | Managed control plane |

## Hybrid mode topology

```
Internal network                    DMZ / Edge
┌─────────────────────┐            ┌────────────────────┐
│  Control Plane (CP) │ ──mTLS──▶  │  Data Plane (DP)   │
│  - Admin API :8001  │            │  - Proxy :8000     │
│  - Kong DB          │            │  - No DB access    │
│  - Config mgmt      │            │  - No Admin API    │
└─────────────────────┘            └────────────────────┘
```

A compromised DP has no access to config or the database.

## DB-less reload (no restart needed)

```bash
curl -X POST http://localhost:8001/config -F config=@kong.yaml
```

## Check connected Data Planes

```bash
curl http://kong:8001/clustering/data-planes | jq '.data[] | {hostname, version, status}'
```
