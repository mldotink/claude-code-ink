---
name: ink-troubleshoot
description: Debug a failing Ink (ml.ink) deployment. Use when a deploy fails, a service crashes, or something isn't working.
argument-hint: "[service-name]"
allowed-tools: Bash, Read, Grep
---

# Troubleshoot Ink Deployment

Debug failing services on Ink (ml.ink) via `POST https://api.ml.ink/graphql`.

Auth: `Authorization: Bearer $INK_API_KEY`. Check it's set first.

## Step 1: Introspect + get service status

Introspect the schema (no auth needed) to discover queries for services, logs, and metrics. Then query the service list with auth. Find the failing service (use `$ARGUMENTS` if provided). Note its status and error message.

## Step 2: Diagnose by status

### `failed` — Build or deploy error
Query **build logs**. Common causes:
- Missing dependencies (package.json/requirements.txt incomplete)
- Wrong runtime version (needs specific Node/Python version)
- `rootDirectory` doesn't exist in the repo (non-retryable)
- Dockerfile not found at `dockerfilePath` (non-retryable)
- Railpack plan generation failed (unsupported project structure)
- Missing base image in Dockerfile

### `crashed` — Container exited after starting
Query **runtime logs**. Common causes:
- Missing environment variables the app expects
- Wrong port — app listens on a port different from what's configured. Health check is TCP socket on the configured port.
- App binding to `127.0.0.1` or `localhost` — must bind to `0.0.0.0`
- Out of memory — check metrics to confirm, then increase (`256Mi` → `512Mi` → `1024Mi` etc.)
- `PORT` env var is auto-injected; if app reads it, the configured port must match

### `deploying` (stuck >2min) — Health check failing
Container started but TCP probe on configured port is failing:
- App not listening on the right port
- App not binding to `0.0.0.0`
- App takes too long to start (health check: initial delay 1s, period 2s, failure threshold 3)
- Static site (`buildpack=static`): port is always `8080`, any user port config is ignored

### `queued` (stuck >5min) — Build queue
Usually transient. If stuck, build system may be at capacity.

### `superseded` — Replaced
A newer deployment was triggered. This is normal if the user pushed again or called updateService while a deploy was in progress.

## Step 3: Check resource usage

If the service ran briefly before crashing, query metrics. If memory usage was at the limit, the container was OOM-killed — recommend increasing memory.

Memory tiers: `128Mi`, `256Mi`, `512Mi`, `1024Mi`, `2048Mi`, `4096Mi`.

## Step 4: Apply fix

Use the service update mutation to fix the issue. Remember:
- `envVars` **replaces the entire set** — include all existing vars plus the fix
- Updating any field triggers an immediate redeploy (in-flight deploys are auto-cancelled)
- `publishDirectory` only valid with `railpack`; `dockerfilePath` only valid with `dockerfile`
- Paths must be relative (no leading `/`, no `..`)

## Step 5: Verify

Poll status until `active`. Curl the `fqdn` URL to confirm it responds.

If still failing, repeat from Step 2 with fresh logs (old deploy logs may be from the previous attempt).
