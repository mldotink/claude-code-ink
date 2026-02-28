---
name: ink-troubleshoot
description: Debug a failing Ink (ml.ink) deployment. Use when a deploy fails, a service crashes, or something isn't working.
argument-hint: "[service-name]"
allowed-tools: Bash, Read, Grep
---

# Troubleshoot Ink Deployment

Debug failing services on Ink (ml.ink) via the GraphQL API at `https://api.ml.ink/graphql`.

## Prerequisites

Check `$INK_API_KEY` is set. Auth: `Authorization: Bearer $INK_API_KEY`.

## Step 1: Discover the API

Introspect the schema (no auth needed) to find queries for services, logs, and metrics. Inspect return types to learn available fields.

## Step 2: Get service status

Query the service list or details. Find the failing service (use `$ARGUMENTS` if provided). Note its status and any error message.

## Step 3: Diagnose by status

### `failed` — Build or deploy error
Query build logs. Common causes:
- Missing dependencies (package.json, requirements.txt incomplete)
- Wrong runtime version
- Build command fails — may need custom build command
- Dockerfile syntax errors

### `crashed` — Container exited after starting
Query runtime logs. Common causes:
- Missing environment variables
- Wrong port — app listens on a different port than configured
- Binding to `localhost` — app must bind to `0.0.0.0`
- Out of memory — check metrics, increase memory

### `deploying` (stuck) — Health check failing
Container started but not responding on the configured port:
- Wrong port configuration
- App not binding to `0.0.0.0`
- App takes too long to start (~60s timeout)

### `queued` (stuck) — Build queue
Usually resolves on its own. If stuck for several minutes, build system may be at capacity.

## Step 4: Check resource usage

If the service ran briefly before crashing, query metrics. If memory usage hit the limit, recommend increasing.

## Step 5: Apply fix

Use the service update mutation to fix the issue (env vars, port, memory, etc.). The update triggers a redeploy automatically.

## Step 6: Verify

Poll status until `active`. Curl the service URL to confirm it responds.

If still failing, repeat from Step 2 with updated logs.
