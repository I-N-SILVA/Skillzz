---
name: create-routine
description: |
  Create Claude Code scheduled routines and automations using CronCreate and RemoteTrigger.
  Use when the user wants to automate a recurring task, schedule a Claude agent to run on
  a cron schedule, set up a daily/weekly workflow, or build a background automation that
  runs without manual intervention.
  Triggers: "create a routine", "schedule this", "automate this task", "run this every X",
  "set up a daily/weekly automation", "remind me every...", "create a cron job for Claude".
---

# Create Routine

## Workflow

### 1. Gather requirements (ask only what's missing)

- **What** should happen? (the task/action)
- **When** — cron schedule, or event-triggered?
- **Tools needed** — Gmail, Slack, Telegram, Google Calendar, web search, files?
- **Output destination** — Telegram, email, Slack, file, Google Sheets?

### 2. Choose the mechanism

| Need | Use |
|------|-----|
| Fixed schedule (daily, hourly, every Monday) | `CronCreate` |
| Event-triggered (webhook, on-demand, RemoteTrigger) | `RemoteTrigger` |
| Recurring loop in current session only | `loop` skill |

Load schemas before use: `ToolSearch select:CronCreate` or `ToolSearch select:RemoteTrigger`

### 3. Craft the routine prompt

The routine agent runs **standalone — it has zero memory of this conversation.**
Write the prompt as if briefing a fresh agent.

**Template:**
```
You are an automated agent for [task description].
Today's date is [use dynamic: new Date().toISOString() or equivalent].
You have access to: [list MCP tools/connectors needed].

TASK:
1. [Step 1 — be specific, no ambiguity]
2. [Step 2]
3. [Always end with a delivery step — send/save/post results]

OUTPUT: Send results to [destination — Telegram chat_id, email address, Slack channel].
If nothing to report, send a brief confirmation message anyway.
On error: notify [destination] with the error details.
```

**Self-contained prompt checklist:**
- No references to "this conversation" or "what we discussed"
- Explicit output destination with all needed IDs/addresses
- Handles empty results and errors explicitly
- Tool/MCP names match what's installed
- No hardcoded dates — use dynamic date resolution

### 4. Create the routine

Call `CronCreate` with:
- `name`: kebab-case, descriptive (e.g., `daily-email-digest`, `monday-standup-prep`)
- `prompt`: self-contained prompt from step 3
- `schedule`: cron expression (see reference below)
- `description`: one sentence of what it does

### 5. Confirm to user

After creation show:
- Routine name + ID
- Schedule in plain English ("runs every weekday at 8:00 AM UTC")
- What it does in one sentence
- Management: use `/schedule` skill or `CronList`/`CronDelete` to manage

---

## Cron Schedule Reference

| Schedule | Expression |
|----------|------------|
| Every day at 8 AM | `0 8 * * *` |
| Every Monday at 9 AM | `0 9 * * 1` |
| Weekdays at 9 AM | `0 9 * * 1-5` |
| Every hour | `0 * * * *` |
| Every 30 minutes | `*/30 * * * *` |
| First of month at 8 AM | `0 8 1 * *` |
| Every Sunday at 10 PM | `0 22 * * 0` |
| Twice a day (9 AM & 5 PM) | `0 9,17 * * *` |

> All times are UTC. Convert timezone if user specifies one (e.g., BRT = UTC-3, EST = UTC-5).

---

## Prompt Templates by Category

See [references/prompt-patterns.md](references/prompt-patterns.md) for copy-paste templates:
- Email digest (Gmail → Telegram)
- Calendar prep (Google Calendar → Telegram)
- Payment alerts (Stripe → Slack/Telegram)
- Content scheduler (AI-generated → social/email)
- GitHub / PR reminders
- Research & daily briefing
