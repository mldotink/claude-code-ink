---
name: ink-troubleshoot
description: Debug a failing Ink (ml.ink) deployment. Use when a deploy fails, a service crashes, or something isn't working.
argument-hint: "[service-name]"
allowed-tools: Bash, Read, Grep
---

# Troubleshoot Ink Deployment

Debug failing services on Ink (ml.ink) using the GraphQL API.

## Prerequisites

Verify `$INK_API_KEY` is set.

## Step 1: Get service status

```bash
curl -s https://api.ml.ink/graphql \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer $INK_API_KEY" \
  -d '{"query": "{ listServices(first: 50) { nodes { id name status errorMessage fqdn port memory vcpus repo branch gitProvider } } }"}' | jq
```

Find the failing service (use `$ARGUMENTS` if provided). Note its `id`, `status`, and `errorMessage`.

## Step 2: Diagnose by status

### `failed` — Build or deploy error
Check build logs first:
```bash
curl -s https://api.ml.ink/graphql \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer $INK_API_KEY" \
  -d '{"query": "{ serviceLogs(input: { serviceId: \"<ID>\", logType: BUILD, limit: 200 }) { entries { message } } }"}' \
  | jq '.data.serviceLogs.entries[] | .message' -r
```

Common build failures:
- **Missing dependencies**: package.json missing a dep, requirements.txt incomplete
- **Wrong runtime version**: needs specific Node/Python version
- **Build command fails**: check if custom buildCommand is needed
- **Dockerfile errors**: syntax issues, missing base image

### `crashed` — Container exited after starting
Check runtime logs:
```bash
curl -s https://api.ml.ink/graphql \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer $INK_API_KEY" \
  -d '{"query": "{ serviceLogs(input: { serviceId: \"<ID>\", logType: RUNTIME, limit: 200 }) { entries { timestamp level message } } }"}' \
  | jq '.data.serviceLogs.entries[] | "\(.timestamp) \(.message)"' -r
```

Common crash causes:
- **Missing env vars**: app expects env vars that aren't set
- **Wrong port**: app listens on a different port than configured. Must match the `port` in service config.
- **Binding to localhost**: app must bind to `0.0.0.0`, not `127.0.0.1` or `localhost`
- **Out of memory**: increase memory allocation (check metrics if the service ran briefly)

### `deploying` (stuck) — Health check failing
The container started but isn't responding on the configured port:
- Verify the app actually listens on the configured port
- Check it binds to `0.0.0.0`
- Check runtime logs for startup errors
- Ensure the app starts within the timeout (~60s)

### `queued` (stuck) — Build queue issue
Usually resolves on its own. If stuck for more than a few minutes, the build system may be at capacity.

## Step 3: Check resource usage

If the service ran briefly before crashing, check metrics:
```bash
curl -s https://api.ml.ink/graphql \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer $INK_API_KEY" \
  -d '{"query": "{ serviceMetrics(serviceId: \"<ID>\", timeRange: ONE_HOUR) { memoryUsageMB { dataPoints { timestamp value } } memoryLimitMB cpuLimitVCPUs } }"}' | jq
```

If memory usage hits the limit, recommend increasing memory.

## Step 4: Apply fix

Once the issue is identified, fix it:

**Update env vars:**
```bash
curl -s https://api.ml.ink/graphql \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer $INK_API_KEY" \
  -d '{
    "query": "mutation { updateService(input: { name: \"<NAME>\", envVars: [{ key: \"KEY\", value: \"VALUE\" }] }) { serviceId status } }"
  }' | jq
```

**Increase memory:**
```bash
curl -s https://api.ml.ink/graphql \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer $INK_API_KEY" \
  -d '{
    "query": "mutation { updateService(input: { name: \"<NAME>\", memory: \"1024Mi\" }) { serviceId status } }"
  }' | jq
```

**Fix port:**
```bash
curl -s https://api.ml.ink/graphql \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer $INK_API_KEY" \
  -d '{
    "query": "mutation { updateService(input: { name: \"<NAME>\", port: 8080 }) { serviceId status } }"
  }' | jq
```

After applying the fix, `updateService` triggers a redeploy. Monitor status until it reaches `active`.

## Step 5: Verify

Check the service is healthy:
```bash
curl -sI "https://<FQDN>" | head -5
```

If still failing, repeat from Step 1 with the new logs.
