# TAGGING

## Recommended tag schema

```
env:prod | env:staging | env:dev
team:platform | team:checkout | team:payments
app:my-service
criticality:high | criticality:low
version:2.4.1
```

Tags are case-sensitive. Define the schema once, enforce it via auto-tagging rules.

## Apply tags via API

```bash
curl -X POST "$DT_ENV/api/v2/tags" \
  -H "Authorization: Api-Token $DT_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
    "entitySelector": "type(SERVICE),entityName(\"my-service\")",
    "tags": [
      {"key": "env",  "value": "prod"},
      {"key": "team", "value": "checkout"}
    ]
  }'
```

## Auto-tagging from K8s labels

`Settings → Tags → Automatically applied tags`

```
Rule name:    env from K8s label
Entity type:  All
Condition:    Kubernetes label "env" exists
Tag:          env:{Kubernetes label: env}
```

Same pattern for namespace, deployment name, or any K8s annotation.

## Management zones — scope by tag

`Settings → Management zones → Add`

```
Name:  Production — Checkout
Rules:
  - Entity type: SERVICE
    Condition: tag("team:checkout") AND tag("env:prod")
  - Entity type: HOST
    Condition: tag("team:checkout")
```

Apply zones to alerting profiles and dashboards to isolate team visibility.

## Filter entities by tag in API

```bash
# Entity selector with tag filter
?entitySelector=type(SERVICE),tag("env:prod"),tag("team:checkout")

# All hosts not yet tagged
?entitySelector=type(HOST),not(tag("env"))
```

## Tag-based entity selector patterns

```
tag("key:value")           exact match
tag("key")                 key exists, any value
not(tag("key"))            key absent
tag("[AWS]InstanceType")   cloud provider tags
```
