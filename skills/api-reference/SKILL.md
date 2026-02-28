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

The API key should be stored in the `INK_API_KEY` environment variable. Always check for `$INK_API_KEY` before making requests. If not set, tell the user to set it:
> Get one from https://ml.ink → Settings → API Keys

**Exception: schema introspection requires no authentication.** See below.

## Making requests

Use `curl` to call the GraphQL API:

```bash
curl -s https://api.ml.ink/graphql \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer $INK_API_KEY" \
  -d '{"query": "<GRAPHQL_QUERY>"}' | jq
```

For mutations with variables:

```bash
curl -s https://api.ml.ink/graphql \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer $INK_API_KEY" \
  -d '{"query": "mutation($input: SomeInput!) { someMutation(input: $input) { ... } }", "variables": {"input": {...}}}' | jq
```

## Schema introspection — ALWAYS DO THIS FIRST

**Introspection is open — no auth needed.** Before making any data request, introspect the live schema to discover the exact queries, mutations, types, and fields available. Never guess or assume field names.

### Discover all queries and mutations

```bash
curl -s https://api.ml.ink/graphql \
  -H "Content-Type: application/json" \
  -d '{"query": "{ __schema { queryType { fields { name args { name type { name kind ofType { name kind ofType { name kind } } } } } } mutationType { fields { name args { name type { name kind ofType { name kind ofType { name kind } } } } } } } }"}' | jq
```

### Inspect a type (object or input)

```bash
# Object type (e.g. Service, Project)
curl -s https://api.ml.ink/graphql \
  -H "Content-Type: application/json" \
  -d '{"query": "{ __type(name: \"TYPENAME\") { fields { name type { name kind ofType { name kind } } } } }"}' | jq

# Input type (e.g. UpdateServiceInput)
curl -s https://api.ml.ink/graphql \
  -H "Content-Type: application/json" \
  -d '{"query": "{ __type(name: \"TYPENAME\") { inputFields { name type { name kind ofType { name kind ofType { name kind } } } } } }"}' | jq
```

### Inspect enums

```bash
curl -s https://api.ml.ink/graphql \
  -H "Content-Type: application/json" \
  -d '{"query": "{ __type(name: \"ENUMNAME\") { enumValues { name } } }"}' | jq
```

**When in doubt, introspect.** It costs nothing and gives you the exact live schema.

## Domain knowledge

These are things you can't learn from introspection:

- **`updateService` is create-or-update**: if the service name doesn't exist, it creates; if it does, it updates and redeploys.
- **Git providers**: `ink` (internal git at `git.ml.ink`) or `github` (requires GitHub App installation).
- **Build packs**: `railpack` (auto-detect — works for most stacks), `dockerfile`, `static`, `dockercompose`.
- **Service status lifecycle**: `queued` → `building` → `deploying` → `active`. Failure states: `failed`, `crashed`, `suspended`.
- **Apps must bind to `0.0.0.0`**, not `127.0.0.1` or `localhost`.
- **Pagination**: list queries use cursor-based pagination (`first`, `after`, `pageInfo`).
- **Workspace scoping**: many queries accept `workspaceSlug` param. Omit for default workspace.
