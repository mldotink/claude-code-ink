---
name: ink-deploy
description: Deploy the current project to Ink (ml.ink). Use when the user wants to deploy, ship, publish, or host their code on Ink.
argument-hint: "[service-name]"
allowed-tools: Bash, Read, Glob, Grep
---

# Deploy to Ink

Deploy the current project to Ink (ml.ink) via `POST https://api.ml.ink/graphql`.

Auth: `Authorization: Bearer $INK_API_KEY` (format `dk_live_...`). Check it's set first; if not, tell user to get one from https://ml.ink → Settings → API Keys.

## Step 1: Introspect the API

Run a schema introspection query (no auth needed) to discover the exact mutation name, input type, and return fields for creating/updating a service. Also introspect the input type to see all available fields.

## Step 2: Analyze the project

Read project files to detect the stack and determine configuration:

| File | Stack | Recommended config |
|---|---|---|
| `package.json` | Node.js | `railpack`, 512Mi, 0.5 vCPU |
| `requirements.txt` / `pyproject.toml` | Python | `railpack`, 512Mi, 0.5 vCPU |
| `go.mod` | Go | `railpack`, 256Mi, 0.25 vCPU |
| `Cargo.toml` | Rust | `railpack`, 512Mi, 0.5 vCPU |
| `Dockerfile` | Docker | `dockerfile`, 512Mi, 0.5 vCPU |
| `docker-compose.yml` | Compose | `dockercompose`, 512Mi, 0.5 vCPU |
| `index.html` (no server) | Static | `static`, 128Mi, 0.25 vCPU |

Detect the **port** the app listens on (check code, Dockerfile EXPOSE, config). If not found:
- Static sites: always `8080` (nginx, automatic)
- Railpack with `publishDirectory`: always `8080`
- Otherwise: defaults to `3000`

Check `.env.example`, README, or config for required **environment variables**.

**Build packs**: `railpack` (auto-detect, works for almost everything), `dockerfile`, `static`, `dockercompose`.

**Memory**: `128Mi`, `256Mi`, `512Mi`, `1024Mi`, `2048Mi`, `4096Mi`.

**vCPUs**: `0.25`, `0.5`, `1`, `2`, `3`, `4`.

## Step 3: Check for existing services

Query the service list to see if already deployed. `updateService` is create-or-update — if the name exists it redeploys; if not it creates.

## Step 4: Set up git

**GitHub** (repo already on GitHub):
- Use `host: "github"` and repo as `owner/name` (NOT the full URL)
- User needs the Ink GitHub App installed on their repo

**Internal git** (no GitHub):
- Use `host: "internal"`
- The service mutation creates the internal repo automatically
- After creation, query the service to get the git remote URL, then:
  ```bash
  git remote add ink <REMOTE_URL>
  git push ink main
  ```
- Future pushes auto-trigger redeployment (post-receive hook)

## Step 5: Deploy

Call the service create/update mutation with detected config.

- **Service name**: use `$ARGUMENTS` if provided, else derive from directory name (lowercase, alphanumeric + hyphens)
- **Repo format**: `owner/name` for GitHub, `ink/name` for internal — never full URLs
- **Paths**: `rootDirectory`, `dockerfilePath`, `publishDirectory` must be relative (no leading `/`, no `..`)
- **`publishDirectory`** only valid with `railpack`; **`dockerfilePath`** only valid with `dockerfile`
- **envVars replaces the entire set** — always send all vars, not just new ones
- `PORT` env var is auto-injected, don't set it manually

Ask the user for secret env var values — never guess them.

## Step 6: Monitor

Poll service status. Expected: `queued` → `building` → `deploying` → `active`.

If `failed`: query build logs.
If `crashed`: query runtime logs. Common causes:
- Missing env vars
- Wrong port (must match what the app listens on)
- Binding to `localhost` instead of `0.0.0.0`
- Out of memory (increase memory)

## Step 7: Report

When `active`, tell the user:
- Live URL from the service's `fqdn` field
- Pushing to the branch auto-redeploys
- Logs available via GraphQL (build and runtime log types)
