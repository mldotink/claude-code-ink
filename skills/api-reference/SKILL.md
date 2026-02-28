---
name: ink-api
description: Background knowledge about the Ink (ml.ink) GraphQL API. Auto-loaded when Claude needs to interact with Ink services, deployments, or the ml.ink platform.
user-invocable: false
---

# Ink (ml.ink) API Reference

Ink is a deployment platform at `ml.ink`. All platform operations go through a single GraphQL API.

## Endpoint

```
POST https://api.ml.ink/graphql
```

## Authentication

All data queries and mutations require an API key:

```
Authorization: Bearer <API_KEY>
```

API key format: `dk_live_` followed by 64 hex characters. Users generate them from https://ml.ink → Settings → API Keys.

**The key should be in the `INK_API_KEY` environment variable.** Always check `$INK_API_KEY` before making requests.

**Exception: schema introspection requires no authentication.**

## Making requests

```bash
curl -s https://api.ml.ink/graphql \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer $INK_API_KEY" \
  -d '{"query": "<GRAPHQL>"}' | jq
```

For mutations with variables:

```bash
curl -s https://api.ml.ink/graphql \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer $INK_API_KEY" \
  -d '{"query": "mutation($input: SomeInput!) { doThing(input: $input) { ... } }", "variables": {"input": {...}}}' | jq
```

## Schema introspection — do this first

**No auth needed.** Always introspect the live schema to discover exact field names, types, and arguments before building queries.

```bash
# All queries and mutations
curl -s https://api.ml.ink/graphql -H "Content-Type: application/json" \
  -d '{"query": "{ __schema { queryType { fields { name args { name type { name kind ofType { name kind ofType { name kind } } } } } } mutationType { fields { name args { name type { name kind ofType { name kind ofType { name kind } } } } } } } }"}' | jq

# Object type fields
curl -s https://api.ml.ink/graphql -H "Content-Type: application/json" \
  -d '{"query": "{ __type(name: \"TYPENAME\") { fields { name type { name kind ofType { name kind } } } } }"}' | jq

# Input type fields
curl -s https://api.ml.ink/graphql -H "Content-Type: application/json" \
  -d '{"query": "{ __type(name: \"TYPENAME\") { inputFields { name type { name kind ofType { name kind ofType { name kind } } } } } }"}' | jq

# Enum values
curl -s https://api.ml.ink/graphql -H "Content-Type: application/json" \
  -d '{"query": "{ __type(name: \"ENUMNAME\") { enumValues { name } } }"}' | jq
```

## Domain knowledge (not discoverable via introspection)

### Valid values for string fields

**Memory** (pick one exactly):
`128Mi`, `256Mi` (default), `512Mi`, `1024Mi`, `2048Mi`, `4096Mi`

**vCPUs** (pick one exactly):
`0.25` (default), `0.5`, `1`, `2`, `3`, `4`

**Build packs**:
- `railpack` (default) — universal auto-detection for Node.js, Python, Go, Rust, Ruby, PHP, Elixir, etc.
- `dockerfile` — custom Dockerfile
- `static` — static site served by nginx on port 8080
- `dockercompose` — Docker Compose

**Git providers** (the `host` field):
- `github` (default) — GitHub repos, requires Ink GitHub App installed
- `internal` — internal git at `git.ml.ink`

### Service status lifecycle

```
queued → building → deploying → active
                 ↘ failed
                            ↘ crashed
```

- `queued` — waiting to build
- `building` — actively building container image
- `deploying` — image built, waiting for k8s rollout + health check
- `active` — serving traffic
- `failed` — build or deploy failed (check error message + build logs)
- `crashed` — container exited after starting (check runtime logs)
- `suspended` — paused by billing or admin
- `superseded` — replaced by a newer deployment
- `cancelled` — deployment cancelled in-flight

### updateService is create-or-update

If the service name doesn't exist, it creates a new service. If it exists, it updates config and triggers an immediate redeploy. Only provided fields are updated (partial update). In-flight deployments are automatically cancelled.

### Port resolution

- **Static sites** (`buildpack=static`): always port `8080`, user port ignored
- **Railpack + publish_directory set**: port `8080`
- **Railpack otherwise**: user port or `3000`
- **Dockerfile**: user port, or auto-detected from last `EXPOSE` in Dockerfile
- Apps **must bind to `0.0.0.0`**, not `127.0.0.1` or `localhost`
- Health check: TCP socket probe on the container port

### Environment variables

- `PORT` is auto-injected with the resolved port value
- Providing `envVars` in an update **replaces the entire set** — always include all vars, not just new ones
- Stored as `[{"key": "...", "value": "..."}]`

### Repo format

- GitHub: `owner/repo` (not full URL, not `git@` format)
- Internal: `ink/repoName`

### Path validation (rootDirectory, dockerfilePath, publishDirectory)

- Must be relative paths (no leading `/`)
- No `..` allowed
- `publishDirectory` only valid with `buildpack=railpack`
- `dockerfilePath` only valid with `buildpack=dockerfile`

### Internal git

- Repos live at `git.ml.ink`
- Token format: `mlg_<base64>` (distinct from API keys)
- Auto-redeploy: pushing to the tracked branch triggers redeploy automatically (via post-receive hook, no webhook needed)
- Repo URL format: `git.ml.ink/users/{username}/{repoName}-{slug}`

### Resources (databases)

- Only type: `sqlite` (Turso-backed)
- Returns connection URL (`libsql://...`) and auth token
- Client libraries: `@libsql/client` (Node.js), `libsql-experimental` (Python)

### Billing tiers

- `free` — max 2 vCPU, 4GB RAM, 5 services
- `hobby` — max 10 vCPU, 20GB RAM, 25 services
- `pro` — max 40 vCPU, 80GB RAM, 200 services

Metering: `$0.000463/vCPU-min`, `$0.000231/GB-min`, `$0.05/GB egress`. Billing values in API are in cents — divide by 100 for dollars.

### Pagination

List queries use cursor-based pagination: `first` (page size), `after` (cursor from `pageInfo.endCursor`). Check `pageInfo.hasNextPage`.

### Workspace scoping

Many queries accept `workspaceSlug`. Omit for the user's default personal workspace. Personal workspace slug = user ID.

### Default project

Every workspace has a `default` project created on signup. Services without an explicit project go there.

### Image caching

If the commit SHA and build config haven't changed, the existing image is reused (no rebuild).
