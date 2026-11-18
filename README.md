# n8n silent failures – checklist + free watchdog workflow

Most n8n monitoring setups only react when an execution **fails**. The painful failures are the ones where nothing fails:

- a schedule trigger is "active" but stops firing,
- a run finishes green but processed 0 items,
- an execution hangs in "running" after a crash,
- an OAuth credential expires and the trigger quietly stops,
- the watchdog itself lives inside the n8n instance that just went down.

This repo contains a **practical checklist** for these cases and a **free n8n workflow** that catches three of them automatically.

## What's inside

| File | What it is |
|---|---|
| [`workflows/silent-failure-watchdog.json`](workflows/silent-failure-watchdog.json) | Import into n8n. Every 15 min it checks for **failed**, **stuck** and **silent schedule** workflows via the n8n public API and sends one readable message (what broke, next step, link). Optional external heartbeat ping. |
| [`CHECKLIST.md`](CHECKLIST.md) | 7 ways n8n workflows fail silently and how to catch each one. |

## Quick start

1. n8n → *Settings → n8n API* → create an API key.
2. Import `workflows/silent-failure-watchdog.json` (*Workflows → Import from file*).
3. Create a **Header Auth** credential (`X-N8N-API-KEY` = your key) and select it in both *Get* nodes.
4. In the **Config** node set `baseUrl`, `notifyWebhookUrl` (Slack/Discord/Teams incoming webhook) and, recommended, `heartbeatUrl` (healthchecks.io or Uptime Kuma push monitor).
5. Activate.

### Known limits
- Reads the latest 250 executions per run. Very busy instances should shorten the interval.
- Schedule rules with cron expressions are skipped (interval-based rules only).
- "Silent schedule" needs at least one past automatic run to compare against.
- It runs inside n8n – that's why the external heartbeat matters.

## Want this as tickets instead of chat pings?

I'm validating a hosted version that turns these findings into **ready-to-work tickets** (Jira, Linear, GitHub Issues or email) with cause, business impact, owner and next step – grouped, auto-closed on recovery, running **outside** your instance.
If that sounds useful, join the early-access list: **https://ticketwatch.ticketwatch-kontakt.workers.dev**

## License
MIT – use it, change it, ship it. Not affiliated with n8n GmbH.
