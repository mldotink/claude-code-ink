---
name: ink-status
description: Check the status of Ink (ml.ink) services, view logs, metrics, and billing. Use when the user asks about their deployments, wants to see logs, or check usage.
argument-hint: "[service-name]"
allowed-tools: Bash
---

# Ink Service Status

Check and manage services deployed on Ink (ml.ink) via the GraphQL API at `https://api.ml.ink/graphql`.

## Prerequisites

Check `$INK_API_KEY` is set. Auth: `Authorization: Bearer $INK_API_KEY`.

## Step 1: Discover available queries

Introspect the schema (no auth needed) to find the exact queries and their return types:

```bash
curl -s https://api.ml.ink/graphql \
  -H "Content-Type: application/json" \
  -d '{"query": "{ __schema { queryType { fields { name args { name type { name kind ofType { name } } } } } } }"}' | jq
```

Then inspect the return types to learn what fields are available on services, logs, metrics, billing, etc.

## Step 2: Execute the relevant query

Based on what the user asked for:

- **List services**: query the service list, present as a table (name, status, URL, resources)
- **Service details**: query a specific service by name or ID, show full config
- **Logs**: query service logs. Two log types exist: build logs and runtime logs. Inspect the enum to confirm values.
- **Metrics**: query service metrics for a time range. Inspect the time range enum for valid values. Summarize CPU %, memory usage vs limit, trends.
- **Billing**: query workspace billing and usage breakdown. Format cents as dollars.
- **Resource limits**: query limits and current usage, show usage vs limits.

## Presentation

- Format service lists as tables
- Format logs as plain text, most recent last
- Summarize metrics (don't dump raw data points unless asked)
- Format costs as dollars (divide cents by 100)
