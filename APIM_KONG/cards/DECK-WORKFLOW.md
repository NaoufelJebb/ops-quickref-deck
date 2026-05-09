# DECK WORKFLOW

## The sequence — every time, no exceptions

```
1. Edit kong.yaml
2. deck validate --config kong.yaml     ← catches schema errors locally
3. deck diff    --config kong.yaml     ← read like a PR, verify intent
4. deck sync    --config kong.yaml     ← apply
```

## Common flags

```bash
# Point at a specific Kong instance
--kong-addr https://kong-prod:8444

# Authenticate
--header "Kong-Admin-Token: $TOKEN"

# Scope to tagged resources only
--select-tag env:prod

# Multiple config files (merged in order)
--config kong-base.yaml --config kong-prod.yaml
```

## ~/.deck.yaml (set defaults, avoid repetition)

```yaml
kong-addr: https://kong-prod:8444
headers:
  - "Kong-Admin-Token: my-token"
```

## Export current state

```bash
deck dump --output-file kong-$(date +%Y%m%d).yaml
```

Run this before any major change. Keep the backup.

## What diff output looks like

```
creating service my-service
creating route my-route
creating plugin rate-limiting for service my-service
Summary:
  Created: 3
  Updated: 0
  Deleted: 0
```

Deletions in the diff are the ones to scrutinize carefully.
