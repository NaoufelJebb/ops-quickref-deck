# OPENSHIFT NOTES — INDEX

## Notes (deep reference)

| File | Open when you need to... |
|---|---|
| [CONCEPTS.md](./notes/CONCEPTS.md) | Understand OpenShift vs Kubernetes, core objects, oc CLI |
| [ARCHITECTURE.md](./notes/ARCHITECTURE.md) | Node types, cluster topology, networking, storage, flavors (OCP/ROSA/ARO) |
| [PROJECTS-AND-RBAC.md](./notes/PROJECTS-AND-RBAC.md) | Projects, quotas, LimitRange, RBAC, service accounts, SCCs |
| [WORKLOADS.md](./notes/WORKLOADS.md) | Deployments, Routes, ImageStreams, BuildConfig, S2I, ConfigMaps |
| [NETWORKING.md](./notes/NETWORKING.md) | Routes, NetworkPolicy, EgressNetworkPolicy, DNS, IngressController, Multus |
| [OPERATORS.md](./notes/OPERATORS.md) | OLM, installing operators, Subscriptions, custom resources, troubleshooting |
| [AUTH.md](./notes/AUTH.md) | OAuth server, LDAP/AD login, OIDC, GitHub, HTPasswd, group sync |
| [MONITORING.md](./notes/MONITORING.md) | Built-in Prometheus stack, ServiceMonitor, custom alerts, Alertmanager |
| [TROUBLESHOOTING.md](./notes/TROUBLESHOOTING.md) | Pod failures, SCC errors, route issues, node problems, build failures |
| [OPS.md](./notes/OPS.md) | Upgrades, certificate rotation, etcd backup, node management, registry |
| [SECURITY.md](./notes/SECURITY.md) | SCCs, image security, secrets, etcd encryption, audit logging, checklist |

## Cards (fast access — open these daily)

| File | Open when you need to... |
|---|---|
| [CHEATSHEET.md](./cards/CHEATSHEET.md) | Any `oc` command — pods, deployments, routes, RBAC, nodes |
| [AD-LDAP-SYNC.md](./cards/AD-LDAP-SYNC.md) | Configure LDAP/AD login and group sync end-to-end |
| [RBAC-QUICKREF.md](./cards/RBAC-QUICKREF.md) | Assign roles, check access, grant SCCs — fast reference |
| [DEBUG-POD.md](./cards/DEBUG-POD.md) | Pod not starting — match state to cause and fix |
| [INCIDENT.md](./cards/INCIDENT.md) | Triage a cluster incident — app down, node NotReady, operator degraded |
| [UPGRADE.md](./cards/UPGRADE.md) | Cluster upgrade runbook — pre-checks, start, monitor, post-checks |

---

## Situation → file map

| I'm doing... | Open... |
|---|---|
| First day, learning OpenShift | [CONCEPTS.md](./notes/CONCEPTS.md) → [ARCHITECTURE.md](./notes/ARCHITECTURE.md) |
| Deploying an application | [WORKLOADS.md](./notes/WORKLOADS.md) |
| Exposing an app externally | [WORKLOADS.md](./notes/WORKLOADS.md) → Routes section |
| Pod won't start | [DEBUG-POD.md](./cards/DEBUG-POD.md) |
| SCC / permission error | [DEBUG-POD.md](./cards/DEBUG-POD.md) + [PROJECTS-AND-RBAC.md](./notes/PROJECTS-AND-RBAC.md) |
| Setting up LDAP / AD login | [AD-LDAP-SYNC.md](./cards/AD-LDAP-SYNC.md) + [AUTH.md](./notes/AUTH.md) |
| Syncing AD groups to OpenShift | [AD-LDAP-SYNC.md](./cards/AD-LDAP-SYNC.md) |
| Assigning RBAC roles | [RBAC-QUICKREF.md](./cards/RBAC-QUICKREF.md) |
| Setting up OIDC (Keycloak/Okta) | [AUTH.md](./notes/AUTH.md) → OIDC section |
| Incident in progress | [INCIDENT.md](./cards/INCIDENT.md) |
| Cluster operator degraded | [INCIDENT.md](./cards/INCIDENT.md) + [TROUBLESHOOTING.md](./notes/TROUBLESHOOTING.md) |
| Node NotReady | [INCIDENT.md](./cards/INCIDENT.md) + [TROUBLESHOOTING.md](./notes/TROUBLESHOOTING.md) |
| Installing an operator | [OPERATORS.md](./notes/OPERATORS.md) |
| Setting up monitoring / alerts | [MONITORING.md](./notes/MONITORING.md) |
| Upgrading the cluster | [UPGRADE.md](./cards/UPGRADE.md) + [OPS.md](./notes/OPS.md) |
| Rotating certificates | [OPS.md](./notes/OPS.md) → Certificate management |
| etcd backup | [OPS.md](./notes/OPS.md) → etcd backup |
| Hardening a cluster | [SECURITY.md](./notes/SECURITY.md) |
| Network policy / isolation | [NETWORKING.md](./notes/NETWORKING.md) |
| Build from source (S2I) | [WORKLOADS.md](./notes/WORKLOADS.md) → BuildConfig section |
| Need a quick oc command | [CHEATSHEET.md](./cards/CHEATSHEET.md) |

