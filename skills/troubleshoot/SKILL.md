---
name: ink-troubleshoot
description: Debug a failing Ink (ml.ink) deployment. Use when a deployment fails, crashes, or behaves unexpectedly.
argument-hint: "[service-name]"
---

# Troubleshoot Ink Deployment

Help the user debug a failing or misbehaving service on Ink (ml.ink).

## Steps

### 1. Identify the service

If `$ARGUMENTS` is provided, use that as the service name. Otherwise, call `mcp__ink__list_services` and ask the user which service to troubleshoot.

### 2. Get service details

Call `mcp__ink__get_service(name: "<service-name>")` and check:
- **status**: What state is the service in?
  - `queued` — Waiting to build. If stuck, there may be a build queue issue.
  - `building` — Build in progress. Check build logs.
  - `deploying` — Container starting. Check runtime logs.
  - `active` — Running but maybe returning errors.
  - `failed` — Build or deploy failed. Check logs.
  - `crashed` — Container crashed after starting. Check runtime logs.
  - `suspended` — Service was suspended (billing or admin action).

### 3. Check logs

Based on the status, guide the user:

**Build failures** (`failed` during build):
- Check build logs for errors
- Common issues:
  - Missing dependencies in package.json / requirements.txt
  - Wrong Node.js / Python version
  - Build command failing
  - Dockerfile syntax errors

**Runtime crashes** (`crashed`):
- Check runtime logs
- Common issues:
  - Missing environment variables
  - Wrong port configuration (app must listen on the port configured in the service)
  - Insufficient memory (increase memory allocation)
  - Missing runtime dependencies

**Deployment stuck** (`deploying` for too long):
- The container is starting but health check is failing
- Check that the app listens on the correct port
- Check that the app starts within the timeout period
- Ensure the app binds to `0.0.0.0`, not `127.0.0.1` or `localhost`

### 4. Check metrics

Call the relevant tools to check resource usage:
- Is the service running out of memory? Suggest increasing memory.
- Is CPU maxed out? Suggest increasing CPU allocation.

### 5. Suggest fixes

Based on the diagnosis:
1. If env vars are missing → help set them with `mcp__ink__update_service`
2. If port is wrong → update the service port configuration
3. If memory is insufficient → increase memory allocation
4. If build fails → help fix the code/config and re-push
5. If the app binds to localhost → fix to bind to `0.0.0.0`

### 6. Verify the fix

After applying fixes, monitor the redeployment:
1. Check status progression: `building` → `deploying` → `active`
2. Confirm the service URL is responding
