---
name: ink-deploy
description: Deploy a project to Ink (ml.ink). Use when the user wants to deploy, ship, or publish their code.
argument-hint: "[service-name]"
---

# Deploy to Ink

Deploy the current project to Ink (ml.ink).

## Steps

### 1. Analyze the project

Determine what kind of project this is by reading the project files:
- `package.json` → Node.js (check for Next.js, Express, MCP SDK, etc.)
- `requirements.txt` / `pyproject.toml` → Python
- `go.mod` → Go
- `Cargo.toml` → Rust
- `Dockerfile` → Docker
- `docker-compose.yml` → Docker Compose
- Static HTML/CSS → Static site

Identify:
- **Build pack**: `railpack` (auto-detect, works for most), `dockerfile`, `static`, or `dockercompose`
- **Port**: What port the app listens on (check code, Dockerfile, or config)
- **Environment variables**: Any required env vars (check `.env.example`, README, config files)
- **Is it an MCP server?**: Check if it uses `@modelcontextprotocol/sdk`, `mcp` Python package, or similar

### 2. Check existing services

Call `mcp__ink__list_projects` to see if this project already has a service deployed. If updating an existing service, use `mcp__ink__update_service` instead.

### 3. Create an internal git repo (if needed)

If the project isn't on GitHub, create an internal Ink git repo:
```
mcp__ink__create_repo(name: "<service-name>")
```

Then get a git token:
```
mcp__ink__get_git_token(repo: "<service-name>")
```

Add the remote and push:
```bash
git remote add ink https://git.ml.ink/<user>/<service-name>.git
git push ink main
```

### 4. Deploy the service

Call `mcp__ink__create_service` with:
- **name**: Service name (lowercase, alphanumeric, hyphens). Use `$ARGUMENTS` if provided, otherwise derive from project directory name.
- **gitHost**: `ink` (internal) or `github`
- **gitUrl**: The repository URL
- **branch**: `main` (or whatever branch to deploy)
- **buildPack**: Auto-detected build pack
- **port**: Detected port number
- **memory**: Start with `256Mi` for simple apps, `512Mi` for Node.js/Python apps, `1024Mi` for larger apps
- **cpu**: Start with `0.25` for simple apps, `0.5` for most apps
- **envVars**: Any required environment variables (ask the user for values of secrets)

### 5. Monitor deployment

After creating the service:
1. Call `mcp__ink__get_service` to check the deployment status
2. Wait a few seconds and check again — look for status progressing through: `queued` → `building` → `deploying` → `active`
3. If status is `failed` or `crashed`, check build logs with `mcp__ink__get_service` and help troubleshoot

### 6. Report result

When deployment succeeds, tell the user:
- Service URL: `https://<service-name>-<hash>.ml.ink`
- How to add a custom domain: `/ink:domains`
- How to check logs: use the `mcp__ink__get_service` tool
- Future deploys: just `git push` to trigger automatic redeployment
