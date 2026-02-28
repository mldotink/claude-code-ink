---
name: ink-setup
description: Set up Ink (ml.ink) authentication. Use when the user wants to connect to Ink or configure their API key.
argument-hint: ""
---

# Ink Setup

Help the user authenticate with their Ink (ml.ink) account.

## Steps

1. **Check if already connected**: Try calling the `mcp__ink__whoami` tool. If it succeeds, tell the user they're already authenticated and show their account info.

2. **If not authenticated**: The Ink MCP server uses OAuth. Claude Code should automatically trigger the OAuth flow when you first call an Ink tool. If it doesn't:
   - Direct the user to https://ml.ink to sign up or log in
   - They can generate an API key from their dashboard under Settings > API Keys
   - The API key format is `mlg_<base64>`
   - They should set it as an environment variable: `export INK_API_KEY=mlg_...`

3. **Verify connection**: Call `mcp__ink__whoami` to confirm authentication works.

4. **Show workspace info**: Call `mcp__ink__list_workspaces` to show available workspaces.

After setup, the user can use `/ink:deploy` to deploy their first service.
