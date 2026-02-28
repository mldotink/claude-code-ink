---
name: ink-status
description: Check the status of Ink (ml.ink) services, view logs, metrics, and billing. Use when the user asks about their deployments, wants to see logs, or check usage.
argument-hint: "[service-name]"
allowed-tools: Bash
---

# Ink Service Status

Check services on Ink (ml.ink) via `POST https://api.ml.ink/graphql`.

Auth: `Authorization: Bearer $INK_API_KEY`. Check it's set first.

## Step 1: Introspect

Run schema introspection (no auth needed) to discover the exact queries, return types, and available fields for services, logs, metrics, and billing.

## Step 2: Execute

Based on what the user asked for:

**Services**: query the service list or details. Present as a table with name, status, URL, memory, vCPUs. Statuses: `queued`, `building`, `deploying`, `active`, `failed`, `crashed`, `suspended`.

**Logs**: query service logs. Two log types exist — introspect the log type enum. Build logs show build output; runtime logs show container stdout/stderr.

**Metrics**: query service metrics for a time range — introspect the time range enum for valid values. Summarize current CPU %, memory MB vs limit, trend direction.

**Billing**: query workspace billing and usage breakdown. All monetary values are in **cents** — divide by 100 for dollars. Tiers: `free` (5 services), `hobby` (25 services), `pro` (200 services).

**Resource limits**: query limits and current usage. Show as usage/max for services, memory, vCPUs.

## Presentation

- Service lists: table format
- Logs: plain text, chronological
- Metrics: summarize trends, don't dump raw data points unless asked
- Costs: always show as dollars, not cents
