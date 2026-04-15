# Connector Mapping: make.com / n8n → Claude MCP Tools

## Supported Connectors (Native MCP)

| make.com module prefix | n8n node type | Claude MCP tool prefix | Notes |
|------------------------|---------------|------------------------|-------|
| `gmail:*` | `n8n-nodes-base.gmail*` | `mcp__claude_ai_Gmail__*` | Read, search, draft |
| `google-calendar:*` | `n8n-nodes-base.googleCalendar*` | `mcp__claude_ai_Google_Calendar__*` | List, create, update events |
| `slack:*` | `n8n-nodes-base.slack*` | `mcp__claude_ai_Slack__*` | Messages, channels |
| `stripe:*` | `n8n-nodes-base.stripe*` | `mcp__claude_ai_Stripe__*` | Payments, customers |
| `telegram-bot-api:*` | `n8n-nodes-base.telegram*` | `mcp__plugin_telegram_telegram__*` | Send/reply messages |
| `google-drive:*` | `n8n-nodes-base.googleDrive*` | `mcp__claude_ai_Google_Drive__*` | Files, folders |
| `zapier:*` | — | `mcp__claude_ai_zapier__*` | Zaps, hooks |
| `atlassian-*` | `n8n-nodes-base.jira*` | `mcp__claude_ai_alassian__*` | Jira, Confluence |

---

## Partial Support (via WebFetch/WebSearch)

These have no native MCP but can be handled via HTTP:

| App | Workaround |
|-----|-----------|
| Notion | `WebFetch` to Notion API `https://api.notion.com/v1/...` |
| Airtable | `WebFetch` to Airtable API `https://api.airtable.com/v0/...` |
| GitHub | `WebFetch` to GitHub API `https://api.github.com/...` |
| Twitter/X | `WebFetch` to X API v2 (requires bearer token) |
| Linear | `WebFetch` to Linear GraphQL API |
| HubSpot | `WebFetch` to HubSpot API |
| Salesforce | `WebFetch` to Salesforce REST API |
| Discord | `WebFetch` to Discord webhook URL |
| Shopify | `WebFetch` to Shopify Admin API |
| WordPress | `WebFetch` to WordPress REST API |
| RSS/Atom feeds | `WebFetch` directly to feed URL |
| Any webhook | `WebFetch` POST to endpoint URL |

---

## Unsupported (Custom MCP Required)

These require the user to set up a dedicated MCP server:

| App | Recommendation |
|-----|----------------|
| Salesforce (complex flows) | Use official Salesforce MCP |
| SAP | Custom MCP required |
| Oracle | Custom MCP required |
| Custom internal APIs | User must build/install MCP |

When translating a workflow with unsupported connectors:
1. Flag clearly in the translation output
2. Suggest the WebFetch workaround where possible
3. Note what credentials/tokens the user needs to provide in the routine prompt

---

## Action Type Mappings

| Automation concept | Claude routine equivalent |
|--------------------|--------------------------|
| Trigger: Schedule | CronCreate with cron expression |
| Trigger: Webhook | RemoteTrigger |
| Trigger: New email | Routine polls Gmail with `newer_than:Xh` |
| Filter / condition | Inline logic in prompt ("if X, then Y, otherwise skip") |
| Iterator / loop | Prompt instructs Claude to "for each item..." |
| Set variable | Not needed — Claude handles state in reasoning |
| JSON parse | Claude reads and parses JSON natively |
| HTTP request | `WebFetch` tool |
| Error handler | "On error, notify [destination] with details" |
| Delay/wait | Not supported — split into separate routines |

---

## Schedule Translation

| make.com / n8n schedule | Cron expression |
|------------------------|-----------------|
| Every day at 8 AM | `0 8 * * *` |
| Every hour | `0 * * * *` |
| Every 15 minutes | `*/15 * * * *` |
| Every Monday | `0 9 * * 1` |
| Every weekday | `0 9 * * 1-5` |
| First of month | `0 8 1 * *` |
| Every 6 hours | `0 */6 * * *` |

Remember: Claude cron times are UTC. Convert from user's local timezone.
