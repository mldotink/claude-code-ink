# Ink Plugin for Claude Code

Deploy and manage services on [Ink](https://ml.ink) directly from Claude Code.

## Install

```bash
/plugin marketplace add mldotink/claude-code-ink
/plugin install ink
```

Or test locally:

```bash
claude --plugin-dir /path/to/claude-code-ink
```

## What you get

### MCP Server

The plugin connects Claude Code to the Ink MCP server (`mcp.ml.ink`), giving you 20+ tools:

- **Services** — create, update, delete, list, inspect deployments
- **Projects** — browse your projects and services
- **Resources** — create and manage SQLite databases
- **Git** — create internal repos, manage git tokens
- **Domains** — add custom domains, manage DNS records
- **DNS** — delegate zones, manage records

### Skills

| Skill | Description |
|---|---|
| `/ink:deploy` | Deploy the current project to Ink |
| `/ink:setup` | Authenticate and connect to your Ink account |
| `/ink:troubleshoot` | Debug a failing deployment |
| `/ink:domains` | Add and manage custom domains |
| `/ink:database` | Create and manage SQLite databases |
| `/ink:migrate` | Migrate from Vercel, Railway, Render, Heroku, or Fly.io |

## Quick start

```
> /ink:setup
> /ink:deploy
```

That's it. Claude will analyze your project, detect the right build configuration, create a service on Ink, and deploy it.

## How it works

The plugin provides two layers:

1. **MCP tools** — structured, typed API calls to the Ink platform (create services, manage DNS, etc.)
2. **Skills** — higher-level workflow guidance that teaches Claude how to orchestrate the tools for common tasks (deploy a project, migrate from another platform, troubleshoot failures)

Skills use the MCP tools under the hood. You can also use the MCP tools directly by asking Claude (e.g., "list my services on Ink").

## Authentication

The Ink MCP server uses OAuth. On first use, Claude Code will open a browser window for you to authorize. After that, your session is remembered.

Alternatively, set an API key:

1. Go to https://ml.ink → Settings → API Keys
2. Generate a key (format: `mlg_...`)
3. Set it: `export INK_API_KEY=mlg_...`

## Supported project types

Ink auto-detects your project type and builds accordingly:

- Node.js (Next.js, Express, Fastify, etc.)
- Python (Flask, FastAPI, Django, etc.)
- Go
- Rust
- Ruby
- Static sites (HTML/CSS/JS)
- Docker & Docker Compose
- MCP servers (any language)

## License

MIT
