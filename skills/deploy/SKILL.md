---
name: ink-deploy
description: Deploy the current project to Ink (ml.ink). Use when the user wants to deploy, ship, publish, or host their code on Ink.
argument-hint: "[service-name]"
allowed-tools: Bash, Read, Glob, Grep
---

# Deploy to Ink

Deploy the current project to Ink (ml.ink) using the GraphQL API.

## Prerequisites

Check that `$INK_API_KEY` is set. If not, tell the user:
> Set your Ink API key: `export INK_API_KEY=dk_live_...`
> Get one from https://ml.ink → Settings → API Keys

Verify auth works:

```bash
curl -s https://api.ml.ink/graphql \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer $INK_API_KEY" \
  -d '{"query": "{ me { email displayName } }"}' | jq
```

## Step 1: Analyze the project

Read project files to determine:

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
- **buildPack**: `railpack` (works for most — auto-detects), `dockerfile`, `static`, or `dockercompose`
- **port**: What port the app listens on
- **envVars**: Required env vars from `.env.example`, README, or config files
- **memory**: `256Mi` for static/simple, `512Mi` for Node/Python, `1024Mi` for larger apps
- **vcpus**: `0.25` for simple, `0.5` for most, `1` for compute-heavy

## Step 2: Check for existing service

```bash
curl -s https://api.ml.ink/graphql \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer $INK_API_KEY" \
  -d '{"query": "{ listServices(first: 50) { nodes { name status fqdn repo } } }"}' | jq
```

If a service with this name already exists, `updateService` will redeploy it.

## Step 3: Set up the git repo

The project needs a git remote that Ink can pull from.

**Option A — GitHub** (if repo is already on GitHub):
- Use `gitProvider: "github"` and the GitHub HTTPS URL
- User needs the Ink GitHub App installed on their repo

**Option B — Ink internal git** (no GitHub needed):
- The user pushes to `git.ml.ink`
- Create the initial deploy with `updateService` using `host: "ink"` — this creates the internal repo automatically
- Then add the remote and push:

```bash
git remote add ink "$(curl -s https://api.ml.ink/graphql \
  -H 'Content-Type: application/json' \
  -H "Authorization: Bearer $INK_API_KEY" \
  -d '{"query": "{ serviceDetails(id: \"<SERVICE_ID>\") { internalUrl } }"}' | jq -r '.data.serviceDetails.internalUrl')"
git push ink main
```

## Step 4: Deploy

Create or update the service:

```bash
curl -s https://api.ml.ink/graphql \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer $INK_API_KEY" \
  -d '{
    "query": "mutation($input: UpdateServiceInput!) { updateService(input: $input) { serviceId name status } }",
    "variables": {
      "input": {
        "name": "<SERVICE_NAME>",
        "repo": "<GIT_URL>",
        "host": "<ink|github>",
        "branch": "main",
        "port": <PORT>,
        "memory": "<MEMORY>",
        "vcpus": "<VCPUS>",
        "buildPack": "<BUILD_PACK>",
        "envVars": [
          { "key": "NODE_ENV", "value": "production" }
        ]
      }
    }
  }' | jq
```

Use `$ARGUMENTS` as the service name if provided, otherwise derive from the directory name (lowercase, alphanumeric + hyphens only).

Ask the user for values of any secret environment variables — never guess them.

## Step 5: Monitor

Poll the service status until deployment completes:

```bash
curl -s https://api.ml.ink/graphql \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer $INK_API_KEY" \
  -d '{"query": "{ listServices(first: 1) { nodes { name status fqdn errorMessage } } }"}' | jq
```

Expected progression: `queued` → `building` → `deploying` → `active`

If `failed` or `crashed`, check build logs:

```bash
curl -s https://api.ml.ink/graphql \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer $INK_API_KEY" \
  -d '{
    "query": "{ serviceLogs(input: { serviceId: \"<SERVICE_ID>\", logType: BUILD, limit: 50 }) { entries { timestamp message } } }"
  }' | jq '.data.serviceLogs.entries[] | .message'
```

## Step 6: Report

When deployment reaches `active`, tell the user:
- **URL**: `https://<fqdn>` (from the service's `fqdn` field)
- **Custom domains**: they can add one from the Ink dashboard or via the `addDnsRecord` / custom domain mutations
- **Auto-redeploy**: pushing to the branch triggers automatic redeployment
- **Logs**: available via GraphQL (`serviceLogs` query with `BUILD` or `RUNTIME` log type)
