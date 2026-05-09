# DEBUG — 404 ROUTE NOT MATCHED

## Step by step

```bash
# 1. List routes and their matchers
curl http://kong:8001/routes | jq '.data[] | {name, paths, methods, hosts, headers}'

# 2. Use the debug header (Kong 3.x)
curl -i http://kong:8000/my/path -H "Kong-Debug: 1"
# Look for: X-Kong-Matched-Route in response headers

# 3. Check the service is registered
curl http://kong:8001/services/my-service | jq .
```

## Common causes

| Symptom | Fix |
|---|---|
| 404 on every route | `deck sync` not applied; check `deck diff` |
| 404 on one route | Trailing slash mismatch, wrong method, regex error |
| 404 after path change | Route not updated — `deck diff` to verify |
| Wrong service matched | Route specificity — more specific path wins |

## strip_path behaviour

```
Route path: /api/v1   strip_path: true
Client:     GET /api/v1/users
Upstream:   GET /users          ← prefix stripped

Route path: /api/v1   strip_path: false
Client:     GET /api/v1/users
Upstream:   GET /api/v1/users   ← full path forwarded
```

## Route priority (when multiple routes could match)

Kong selects the most specific match:
1. Longest path prefix wins
2. Routes with `hosts` beat routes without
3. Routes with `methods` beat routes without
