---
name: ink-status
description: Check the status of Ink (ml.ink) services, view logs, metrics, and billing. Use when the user asks about their deployments, wants to see logs, or check usage.
argument-hint: "[service-name]"
allowed-tools: Bash
---

# Ink Service Status

Check and manage services deployed on Ink (ml.ink).

## Prerequisites

Verify `$INK_API_KEY` is set. If not:
> Set your Ink API key: `export INK_API_KEY=dk_live_...`

## List all services

```bash
curl -s https://api.ml.ink/graphql \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer $INK_API_KEY" \
  -d '{"query": "{ listServices(first: 50) { nodes { id name status fqdn memory vcpus repo branch customDomain updatedAt } totalCount } }"}' | jq
```

Present as a table showing name, status, URL, resources.

## Service details

If the user asks about a specific service (`$ARGUMENTS`), get its full details:

```bash
curl -s https://api.ml.ink/graphql \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer $INK_API_KEY" \
  -d '{
    "query": "{ listServices(first: 50) { nodes { id name status fqdn errorMessage memory vcpus repo branch port gitProvider envVars { key } commitHash customDomain customDomainStatus createdAt updatedAt } } }"
  }' | jq '.data.listServices.nodes[] | select(.name == "'$ARGUMENTS'")'
```

## View logs

### Build logs
```bash
SERVICE_ID="<get from service details>"
curl -s https://api.ml.ink/graphql \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer $INK_API_KEY" \
  -d '{
    "query": "{ serviceLogs(input: { serviceId: \"'$SERVICE_ID'\", logType: BUILD, limit: 100 }) { entries { timestamp message } hasMore } }"
  }' | jq '.data.serviceLogs.entries[] | .message' -r
```

### Runtime logs
```bash
curl -s https://api.ml.ink/graphql \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer $INK_API_KEY" \
  -d '{
    "query": "{ serviceLogs(input: { serviceId: \"'$SERVICE_ID'\", logType: RUNTIME, limit: 100 }) { entries { timestamp level message } hasMore } }"
  }' | jq '.data.serviceLogs.entries[] | "\(.timestamp) [\(.level)] \(.message)"' -r
```

## View metrics

```bash
curl -s https://api.ml.ink/graphql \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer $INK_API_KEY" \
  -d '{
    "query": "{ serviceMetrics(serviceId: \"'$SERVICE_ID'\", timeRange: ONE_HOUR) { cpuUsage { dataPoints { timestamp value } } memoryUsageMB { dataPoints { timestamp value } } memoryLimitMB cpuLimitVCPUs } }"
  }' | jq
```

Summarize: current CPU %, current memory MB / limit, trend over the period.

## Billing & usage

```bash
curl -s https://api.ml.ink/graphql \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer $INK_API_KEY" \
  -d '{"query": "{ workspaceBilling { currentUsageCents estimatedBillCents subscription { tier status } spendingLimitCents suspended } usageBillBreakdown { memory { quantity unit totalCents } cpu { quantity unit totalCents } egress { quantity unit totalCents } currentBillCents periodStart periodEnd } }"}' | jq
```

Format costs as dollars (divide cents by 100).

## Resource limits

```bash
curl -s https://api.ml.ink/graphql \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer $INK_API_KEY" \
  -d '{"query": "{ resourceLimits { tier maxVcpus maxMemoryMi maxServices usage { services memoryMi vcpus } } }"}' | jq
```

Show usage vs limits as a summary.
