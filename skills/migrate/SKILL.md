---
name: ink-migrate
description: Migrate a project from another platform (Vercel, Railway, Render, Heroku, Fly.io) to Ink (ml.ink). Use when the user wants to move their deployment to Ink.
argument-hint: "[source-platform]"
---

# Migrate to Ink

Help the user migrate their project from another deployment platform to Ink (ml.ink).

## Steps

### 1. Identify source platform

Determine where the project is currently deployed. Check for:
- `vercel.json` → Vercel
- `railway.json` / `railway.toml` → Railway
- `render.yaml` → Render
- `Procfile` → Heroku
- `fly.toml` → Fly.io
- `app.yaml` → Google App Engine
- Ask the user if unclear

### 2. Extract current configuration

Read the platform-specific config file to understand:
- **Build command** and **start command**
- **Environment variables** (names only — ask user for secret values)
- **Port** the app listens on
- **Memory / CPU** allocation
- **Custom domains** configured
- **Databases** or other attached resources

### 3. Map to Ink configuration

| Source concept | Ink equivalent |
|---|---|
| Build command | Auto-detected by railpack (or Dockerfile) |
| Start command | Auto-detected by railpack (or Dockerfile) |
| Environment variables | `envVars` in service config |
| Custom domains | `mcp__ink__add_custom_domain` |
| Postgres/MySQL | Not available — use Ink SQLite databases or external DB |
| Redis | Not available — use external provider |
| Cron jobs | Not available yet — use external scheduler |

### 4. Create Ink service

Follow the deploy workflow:
1. Set up git repo (internal or GitHub)
2. Create the service with `mcp__ink__create_service`
3. Add environment variables
4. Monitor deployment

### 5. Migrate DNS

If the user has custom domains:
1. Add domains to the Ink service with `mcp__ink__add_custom_domain`
2. Update DNS records (CNAME to `<service>.ml.ink`)
3. Or delegate the zone to Ink for full DNS management

### 6. Cleanup checklist

After successful migration, remind the user to:
- [ ] Verify the app works on the new Ink URL
- [ ] Update DNS for custom domains
- [ ] Wait for TLS certificate provisioning
- [ ] Test all endpoints / pages
- [ ] Update any CI/CD pipelines to deploy to Ink
- [ ] Decommission the old platform deployment
- [ ] Update any webhook URLs or callback URLs
