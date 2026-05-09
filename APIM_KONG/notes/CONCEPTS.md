# CONCEPTS

## The 6 objects

| Object | One-line definition |
|---|---|
| **Service** | Your upstream API — `http://my-api:8080` |
| **Route** | A matcher that forwards requests to a Service |
| **Plugin** | Middleware attached to Global / Service / Route / Consumer |
| **Consumer** | A client identity — app, user, partner |
| **Upstream** | A load-balanced pool of targets behind a Service |
| **Workspace** | Config isolation boundary *(Enterprise only)* |

## Plugin hierarchy

```
Global → Service → Route → Consumer
```

Lower overrides higher. A Consumer-level rate limit beats a Service-level one.

## Request lifecycle

```
1. access         auth, ACL, rate-limiting — can reject here
2. rewrite        mutate request before proxying
3. proxy          forward to upstream
4. header_filter  mutate response headers
5. body_filter    mutate response body
6. log            async — zero latency impact
```

## Scope quick-pick

| I want to... | Attach plugin to... |
|---|---|
| Apply to everything | Global |
| Apply to all routes of one API | Service |
| Apply to one endpoint only | Route |
| Apply to one client only | Consumer |

## Key ports

| Port | Role |
|---|---|
| `8000` | HTTP proxy |
| `8443` | HTTPS proxy |
| `8001` | Admin API — **never expose externally** |
| `8444` | Admin API HTTPS |
| `8002` | Kong Manager UI *(Enterprise)* |
| `8005` | Hybrid CP↔DP clustering |
