# DEPLOYMENT EVENT

Push this in every CI/CD pipeline so Davis can correlate problems with releases.

## Minimal push

```bash
curl -X POST "$DT_ENV/api/v2/events/ingest" \
  -H "Authorization: Api-Token $DT_TOKEN" \
  -H "Content-Type: application/json" \
  -d "{
    \"eventType\": \"CUSTOM_DEPLOYMENT\",
    \"title\": \"Deploy $SERVICE v$VERSION\",
    \"entitySelector\": \"type(SERVICE),tag(\\\"app:$SERVICE\\\")\"
  }"
```

## Full payload with metadata

```bash
curl -X POST "$DT_ENV/api/v2/events/ingest" \
  -H "Authorization: Api-Token $DT_TOKEN" \
  -H "Content-Type: application/json" \
  -d "{
    \"eventType\": \"CUSTOM_DEPLOYMENT\",
    \"title\": \"Deploy $SERVICE v$VERSION\",
    \"entitySelector\": \"type(SERVICE),tag(\\\"app:$SERVICE\\\"),tag(\\\"env:prod\\\")\",
    \"properties\": {
      \"deploymentVersion\": \"$VERSION\",
      \"deploymentProject\": \"$SERVICE\",
      \"deployedBy\": \"$CI_USER\",
      \"ciLink\": \"$CI_BUILD_URL\",
      \"gitCommit\": \"$GIT_SHA\"
    }
  }"
```

Required token scope: `events.ingest`

## GitHub Actions example

```yaml
- name: Notify Dynatrace
  run: |
    curl -X POST "${{ secrets.DT_ENV }}/api/v2/events/ingest" \
      -H "Authorization: Api-Token ${{ secrets.DT_TOKEN }}" \
      -H "Content-Type: application/json" \
      -d "{
        \"eventType\": \"CUSTOM_DEPLOYMENT\",
        \"title\": \"Deploy ${{ env.SERVICE }} v${{ github.ref_name }}\",
        \"entitySelector\": \"type(SERVICE),tag(\\\"app:${{ env.SERVICE }}\\\")\",
        \"properties\": {
          \"deploymentVersion\": \"${{ github.ref_name }}\",
          \"deployedBy\": \"${{ github.actor }}\",
          \"ciLink\": \"${{ github.server_url }}/${{ github.repository }}/actions/runs/${{ github.run_id }}\"
        }
      }"
```

## Why it matters

- Deployment markers appear on the Davis timeline inside Problems
- Davis automatically checks if a deployment preceded an anomaly
- Enables "did this deploy cause the issue?" root-cause attribution without manual correlation
