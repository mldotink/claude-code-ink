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

Every request requires an API key in the Authorization header:

```
Authorization: Bearer <API_KEY>
```

API keys have the format `dk_live_<64 hex chars>`. Users generate them from the Ink dashboard (Settings > API Keys) or via the `createAPIKey` mutation if already authenticated.

**The API key should be stored in the `INK_API_KEY` environment variable.** Always check for `$INK_API_KEY` before making requests. If not set, tell the user to set it.

## Making requests

Use `curl` to call the GraphQL API. Always use POST with `Content-Type: application/json`:

```bash
curl -s https://api.ml.ink/graphql \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer $INK_API_KEY" \
  -d '{"query": "{ me { id email displayName } }"}'
```

Pipe through `jq` for readable output when showing results to the user.

## Schema introspection

**Introspection is open — no authentication required.** Always introspect first if you're unsure about field names, arguments, or types. The live schema is the source of truth.

List all queries and mutations:

```bash
curl -s https://api.ml.ink/graphql \
  -H "Content-Type: application/json" \
  -d '{"query": "{ __schema { queryType { fields { name args { name type { name kind ofType { name kind ofType { name kind } } } } } } mutationType { fields { name args { name type { name kind ofType { name kind ofType { name kind } } } } } } } }"}' | jq
```

Inspect a specific type (e.g. Service, UpdateServiceInput, LogsInput):

```bash
curl -s https://api.ml.ink/graphql \
  -H "Content-Type: application/json" \
  -d '{"query": "{ __type(name: \"Service\") { fields { name type { name kind ofType { name kind } } } } }"}' | jq
```

Inspect an input type:

```bash
curl -s https://api.ml.ink/graphql \
  -H "Content-Type: application/json" \
  -d '{"query": "{ __type(name: \"UpdateServiceInput\") { inputFields { name type { name kind ofType { name kind ofType { name kind } } } } } }"}' | jq
```

Inspect enums:

```bash
curl -s https://api.ml.ink/graphql \
  -H "Content-Type: application/json" \
  -d '{"query": "{ __type(name: \"LogType\") { enumValues { name } } }"}' | jq
```

**When in doubt, introspect.** It costs nothing and gives you the exact schema.

## Key queries

### Identity
```graphql
{ me { id email displayName githubUsername } }
```

### List projects
```graphql
{ listProjects(first: 20) {
    nodes { id name ref services { id name status } createdAt }
    pageInfo { hasNextPage endCursor }
    totalCount
} }
```

### List services
```graphql
{ listServices(first: 20) {
    nodes {
      id name repo branch status port memory vcpus
      fqdn customDomain customDomainStatus
      gitProvider envVars { key value } commitHash
      createdAt updatedAt
    }
    pageInfo { hasNextPage endCursor }
    totalCount
} }
```

### Service details
```graphql
{ serviceDetails(id: "<SERVICE_ID>") {
    id name repo branch status errorMessage
    fqdn internalUrl port memory vcpus
    gitProvider envVars { key value } commitHash
    customDomain customDomainStatus
    createdAt updatedAt
} }
```

### Service logs
```graphql
{ serviceLogs(input: { serviceId: "<SERVICE_ID>", logType: BUILD, limit: 100 }) {
    entries { timestamp level message }
    hasMore
} }
```

Log types: `BUILD`, `RUNTIME`.

### Service metrics
```graphql
{ serviceMetrics(serviceId: "<SERVICE_ID>", timeRange: ONE_HOUR) {
    cpuUsage { dataPoints { timestamp value } }
    memoryUsageMB { dataPoints { timestamp value } }
    memoryLimitMB
    cpuLimitVCPUs
} }
```

Time ranges: `ONE_HOUR`, `SIX_HOURS`, `SEVEN_DAYS`, `THIRTY_DAYS`.

### Resource limits
```graphql
{ resourceLimits {
    tier maxVcpus maxMemoryMi maxServices
    usage { services memoryMi vcpus }
} }
```

### Billing
```graphql
{ workspaceBilling {
    currentUsageCents includedUsageCents estimatedBillCents
    subscription { tier status currentPeriodEnd }
    spendingLimitCents suspended
} }
```

### Usage breakdown
```graphql
{ usageBillBreakdown {
    memory { quantity unit totalCents }
    cpu { quantity unit totalCents }
    egress { quantity unit totalCents }
    currentBillCents periodStart periodEnd
} }
```

## Key mutations

### Update a service
```graphql
mutation {
  updateService(input: {
    name: "<SERVICE_NAME>"
    repo: "<GIT_URL>"
    branch: "main"
    port: 3000
    memory: "512Mi"
    vcpus: "0.5"
    buildPack: "railpack"
    envVars: [{ key: "NODE_ENV", value: "production" }]
  }) { serviceId name status }
}
```

`updateService` creates a new service if the name doesn't exist, or updates + redeploys if it does.

**Valid memory values**: `256Mi`, `512Mi`, `1024Mi`, `2048Mi`, `4096Mi`
**Valid vcpu values**: `0.25`, `0.5`, `1`, `2`, `4`
**Valid buildPack values**: `railpack` (auto-detect), `dockerfile`, `static`, `dockercompose`

### Delete a service
```graphql
mutation { deleteService(name: "<SERVICE_NAME>") { serviceId name message } }
```

### Create API key
```graphql
mutation { createAPIKey(name: "my-key") { apiKey { id name prefix } secret } }
```

Save the `secret` immediately — it cannot be retrieved later.

### DNS operations
```graphql
mutation { createHostedZone(zone: "example.com") { zoneId zone status dnsRecords { host type value } } }
mutation { verifyHostedZone(zone: "example.com") { zone status message } }
mutation { addDnsRecord(zone: "example.com", name: "www", type: "CNAME", content: "myapp.ml.ink", ttl: 3600) { id name type content } }
mutation { deleteDnsRecord(zone: "example.com", recordId: "<RECORD_ID>") }
```

## Pagination

List queries use cursor-based pagination:
- `first: Int` — number of items per page
- `after: String` — cursor from `pageInfo.endCursor`
- Check `pageInfo.hasNextPage` to know if there are more results

## Workspace scoping

Many queries accept an optional `workspaceSlug: String` parameter. Omit it to use the user's default personal workspace.

## Service statuses

Services go through these states:
- `queued` → waiting to build
- `building` → build in progress
- `deploying` → container starting, waiting for health check
- `active` → running and healthy
- `failed` → build or deploy failed (check `errorMessage`)
- `crashed` → container exited after starting (check runtime logs)
- `suspended` → paused by billing or admin action

## Git providers

- `ink` — internal git hosting at `git.ml.ink`
- `github` — GitHub repositories (requires GitHub app installation)
