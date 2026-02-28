---
name: ink-deploy
description: Deploy the current project to Ink (ml.ink). Use when the user wants to deploy, ship, publish, or host their code on Ink.
argument-hint: "[service-name]"
allowed-tools: Bash, Read, Glob, Grep
---

# Deploy to Ink

Deploy the current project to Ink (ml.ink) using the GraphQL API at `https://api.ml.ink/graphql`.

## Prerequisites

1. Check `$INK_API_KEY` is set. If not, tell the user to get one from https://ml.ink → Settings → API Keys.
2. Verify auth: query `me` and confirm it returns user data.

## Step 1: Discover the API

Introspect the schema (no auth needed) to find the exact mutations and types for deploying:

```bash
curl -s https://api.ml.ink/graphql \
  -H "Content-Type: application/json" \
  -d '{"query": "{ __schema { mutationType { fields { name args { name type { name kind ofType { name } } } } } }"}' | jq
```

Look for the service creation/update mutation and inspect its input type to learn the exact fields available.

## Step 2: Analyze the project

Read project files to determine the stack:

| File | Stack |
|---|---|
| `package.json` | Node.js (check for next, express, fastify, @modelcontextprotocol/sdk) |
| `requirements.txt` / `pyproject.toml` | Python |
| `go.mod` | Go |
| `Cargo.toml` | Rust |
| `Dockerfile` | Docker |
| `docker-compose.yml` | Docker Compose |
| `index.html` (no server) | Static site |

Determine:
- **buildPack**: `railpack` (auto-detects most stacks), `dockerfile`, `static`, or `dockercompose`
- **port**: what port the app listens on
- **envVars**: required env vars from `.env.example`, README, or config
- **memory/cpu**: appropriate sizing for the stack

## Step 3: Check for existing services

Query the list of services to see if this project is already deployed. If it exists, updating it will trigger a redeploy.

## Step 4: Set up git

The project needs a git remote Ink can pull from:

- **GitHub**: use the GitHub HTTPS URL. User needs the Ink GitHub App installed.
- **Ink internal git**: use `host: "ink"`. The service mutation creates the internal repo automatically. Then add the git remote and push.

## Step 5: Deploy

Call the service create/update mutation with the detected configuration. Use `$ARGUMENTS` as the service name if provided, otherwise derive from the directory name (lowercase, alphanumeric, hyphens).

Ask the user for values of any secret environment variables — never guess them.

## Step 6: Monitor

Poll service status until deployment completes. Expected progression: `queued` → `building` → `deploying` → `active`.

If `failed` or `crashed`, query the service logs (build or runtime) and help troubleshoot.

## Step 7: Report

When `active`, tell the user:
- The live URL (from the service's `fqdn` field)
- That pushing to the branch triggers automatic redeployment
- How to check logs via the GraphQL API
