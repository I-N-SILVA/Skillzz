---
name: import-automation
description: |
  Import a workflow JSON from make.com (Integromat) or n8n and translate it into a
  natural-language Claude routine, then optionally create it as a scheduled automation
  via CronCreate.
  Use when the user pastes or shares a make.com scenario JSON, an n8n workflow JSON,
  or asks to "convert", "import", or "migrate" a make/n8n workflow to Claude routines.
  Triggers: "import this n8n workflow", "convert my make.com scenario", "translate this
  automation JSON", "migrate this workflow to Claude", paste of raw JSON with make/n8n structure.
---

# Import Automation

## Workflow

### 1. Detect the platform

Check the JSON structure to identify the source:

**make.com** (Integromat) signatures:
- Top-level keys: `name`, `flow`, `metadata`, `designer`
- Modules in `flow[].module` (e.g., `"gmail:TriggerNewEmail"`, `"slack:CreateMessage"`)
- Filters in `flow[].filter`

**n8n** signatures:
- Top-level keys: `nodes`, `connections`, `active`, `settings`
- Each node has `type`, `name`, `parameters`, `position`
- Trigger nodes have type ending in `Trigger` (e.g., `n8n-nodes-base.gmailTrigger`)

If platform is ambiguous, ask the user.

### 2. Parse the workflow

Extract these elements:

| Element | make.com location | n8n location |
|---------|-------------------|--------------|
| Trigger | First module in `flow` | Node with `Trigger` in type |
| Steps/actions | Remaining `flow` items | All non-trigger nodes |
| Conditions/filters | `flow[].filter` | `IF` nodes or `conditions` params |
| Schedule | `scheduling` field | Cron trigger `parameters` |
| App connections | `module` field prefix | `type` field prefix |

### 3. Translate to natural language

Build a structured summary:

```
TRIGGER: [What starts this workflow — schedule, new email, webhook, etc.]

STEPS:
1. [Action in plain English]
2. [Next action]
3. [Delivery/output step]

CONDITIONS: [Any filters or branching logic]

CONNECTIONS NEEDED: [App1, App2, App3]

SUGGESTED SCHEDULE: [If cron-based, translate to human + cron expression]
```

Show this summary to the user and confirm it's accurate before proceeding.

### 4. Create the Claude routine prompt

Translate the summary into a self-contained Claude routine prompt.
Map make.com/n8n app names to available MCP tools:

See [references/connector-map.md](references/connector-map.md) for the full mapping table.

Key mappings:
- Gmail module → `mcp__claude_ai_Gmail__*` tools
- Google Calendar → `mcp__claude_ai_Google_Calendar__*`
- Slack → `mcp__claude_ai_Slack__*`
- Stripe → `mcp__claude_ai_Stripe__*`
- Telegram → `mcp__plugin_telegram_telegram__*`
- HTTP/Webhook → `WebFetch` or `WebSearch`
- Google Sheets/Drive → `mcp__claude_ai_Google_Drive__*`

For unsupported apps, note them as "requires custom MCP setup" and suggest a workaround.

### 5. Offer to create the routine

Ask: "Want me to create this as a scheduled Claude routine?"

If yes — invoke the `create-routine` skill with the translated prompt and schedule.

---

## Handling Edge Cases

**Complex branching (if/else trees):** Simplify to the primary path; note secondary paths as "optional branches" in the prompt.

**Multiple triggers:** Ask the user which trigger to use; Claude routines are single-trigger.

**Unsupported apps (e.g., HubSpot, Salesforce):** Flag clearly, suggest using `WebFetch` with their API as a workaround.

**Data transformation steps** (e.g., JSON parse, set variable): Inline the logic into the prompt step description.

See [references/connector-map.md](references/connector-map.md) for full app → MCP mappings.
