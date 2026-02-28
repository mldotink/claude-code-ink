# Ink Plugin for Claude Code

Deploy and manage services on [Ink](https://ml.ink) directly from Claude Code. No MCP server required — Claude talks to the Ink GraphQL API using `curl`.

## Install

```bash
/plugin marketplace add mldotink/claude-code-ink
/plugin install ink
```

Or test locally:

```bash
claude --plugin-dir /path/to/claude-code-ink
```

## Setup

1. Get an API key from https://ml.ink → Settings → API Keys
2. Set it in your environment:

```bash
export INK_API_KEY=dk_live_...
```

## Skills

| Skill | Description |
|---|---|
| `/ink:deploy` | Deploy the current project to Ink |
| `/ink:status` | Check services, logs, metrics, billing |
| `/ink:troubleshoot` | Debug a failing deployment |

Claude also has background knowledge of the full Ink GraphQL API (`ink-api` skill), so you can ask it anything — "list my services", "show me build logs for X", "what's my current bill" — and it will make the right API calls.

## How it works

The plugin teaches Claude the Ink GraphQL API:

- **Endpoint**: `POST https://api.ml.ink/graphql`
- **Auth**: `Authorization: Bearer $INK_API_KEY`
- **Schema discovery**: Claude can introspect the schema at any time to discover or verify operations

No binary, no runtime, no MCP server. Claude uses `curl` + `jq` to call the API directly. The skills provide workflow guidance for common tasks (deploying, debugging), while the API reference skill gives Claude the full schema knowledge to handle any request.

## Quick start

```
> /ink:deploy
```

Claude analyzes your project, detects the stack, creates a service on Ink, and deploys it.

```
> show me the runtime logs for my-api
```

Claude calls the GraphQL API and shows you the logs.

## Supported stacks

Ink auto-detects your project via [Railpack](https://railpack.io):

- Node.js (Next.js, Express, Fastify, etc.)
- Python (Flask, FastAPI, Django, etc.)
- Go, Rust, Ruby, PHP, Elixir
- Static sites
- Docker & Docker Compose

## License

MIT
