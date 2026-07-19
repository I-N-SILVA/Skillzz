# Skillzz

Reusable Claude Code skills. Each folder is one skill: a `SKILL.md` entry point (with YAML
frontmatter for name + trigger description) plus optional `references/` files loaded on demand.

## Skills

| Skill | Folder | What it does |
|---|---|---|
| `shopify-theme` | `Shopify/` | Shopify **websites**: Liquid themes, OS 2.0 sections/blocks, Theme Editor schemas, brand intake → production-ready theme code. References: deep reference (metafields, metaobjects, i18n, cart patterns, Theme Store submission) + store launch checklist (payments/shipping/taxes, legal, analytics, recommended app stack, QA, go-live). |
| `shopify-app` | `shopify-app/` | Shopify **plugins/apps**: CLI + React Router scaffold, Polaris/App Bridge admin UI, GraphQL Admin API, webhooks, all extension surfaces (theme app extensions, checkout UI, Functions, admin, pixels), step-by-step wiring between components, deploy & App Store review. References: extensions cookbook · GraphQL Admin API cookbook · billing & App Store launch playbook · multichannel/migration playbook (Etsy, Amazon, eBay, WooCommerce, Hydrogen decision). |
| `create-routine` | `create-routine/` | Scheduled Claude automations: cron routines, self-contained prompts, delivery to Slack/Telegram/email. |
| `import-automation` | `import-automation/` | Import automations across connectors (see folder). |

## The Shopify suite — how the two skills fit together

```
merchant request
      │
      ├─ "make my store look …" ──────────────► shopify-theme  (Liquid, sections, design)
      ├─ "add a feature / plugin / integration" ► shopify-app   (app server, API, extensions)
      └─ spans both (app widget styled in theme) ► use both: app provides the block,
                                                   theme skill styles & places it
```

Decision guidance lives in `shopify-app/SKILL.md` → Phase 0.

## Conventions for new skills

- One folder per skill, `SKILL.md` ≤ ~450 lines; push depth into `references/*.md`.
- Frontmatter `description` lists explicit trigger phrases — it's what makes the skill fire.
- Prefer checklists, decision tables, and copy-paste-ready code over prose.
- State hard rules ("Core Mandates") and known gotchas as tables.
