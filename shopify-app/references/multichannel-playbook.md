# Multichannel & Migration Playbook

For merchants who **also sell somewhere else** (Etsy, Amazon, eBay, Walmart, WooCommerce, Wix,
Square/Weebly, Lightspeed…) or want to **move to Shopify**. Read this before designing any data
flow — multichannel changes where the source of truth lives.

## Contents

1. [First Question: Migrate, Sync, or Both?](#1-first-question-migrate-sync-or-both)
2. [Migration to Shopify (platform → platform)](#2-migration-to-shopify)
3. [Ongoing Sync: Selling on Multiple Channels](#3-ongoing-sync-selling-on-multiple-channels)
4. [Source-of-Truth Architecture](#4-source-of-truth-architecture)
5. [SEO & Cutover Checklist](#5-seo--cutover-checklist)
6. [Headless / Hydrogen Decision](#6-headless--hydrogen-decision)

---

## 1. First Question: Migrate, Sync, or Both?

| Situation | Strategy |
|---|---|
| "I'm leaving WooCommerce/Wix for Shopify" | **Migrate** (§2) then retire the old store |
| "I sell on Etsy/Amazon and want my own site" | **Shopify as new home + sync** (§2 import, §3 sync) |
| "Shopify is my store, I want to ALSO list on Amazon/eBay/Etsy/Walmart" | **Marketplace Connect** (§3) |
| "I have physical stores / Square POS" | Migrate data; consider Shopify POS to unify |
| "Multiple regional sites" | One Shopify store + Markets (multi-currency/language), not multiple stores |

Ask before proposing anything:
1. Which platforms, and which one takes the most orders today?
2. Same SKUs everywhere, or channel-specific catalogs?
3. Where is inventory counted today (spreadsheet, ERP, one platform)?
4. Keep selling on the old channel after Shopify launches? (Etsy/Amazon: usually YES — keep the
   traffic; WooCommerce/Wix: usually NO — redirect.)

---

## 2. Migration to Shopify

### Tool ladder (try in this order)

| Tool | From | Notes |
|---|---|---|
| **Shopify "Store Migration" app** (free, first-party) | WooCommerce, Wix, Squarespace, Square, Etsy, Amazon, eBay, Clover, Lightspeed | Products + customers; simplest path |
| **Marketplace Connect import** | Amazon, eBay, Etsy, Walmart | When they'll KEEP selling there — imports and then stays synced |
| **CSV import** (Products/Customers admin importers) | Anything that exports CSV | Reshape columns to Shopify's product CSV schema; variants = repeated handle rows |
| **Matrixify** (paid app) | Large/complex catalogs, orders, metafields, redirects | The pro tool: 100k+ SKUs, order history, scheduled jobs |
| **Custom script via Admin GraphQL** | Weird legacy systems | `productSet` / `customerCreate` mutations in bulk; use Bulk Operations API for scale |

### What migrates cleanly vs. what needs care

| Data | Difficulty | Notes |
|---|---|---|
| Products, variants, images | Easy | Watch variant limits and option structure |
| Customers | Easy | Passwords NEVER migrate — plan a "reset your password" email campaign |
| Order history | Medium | Importable (Matrixify/API) as archived orders — do it for LTV/reporting |
| Discount codes | Medium | Recreate; logic-based promos become Functions |
| Reviews | Medium | Export → import into your chosen reviews app's format |
| Gift cards | Hard | Shopify plan-dependent; issue replacements if needed |
| Subscriptions | Hard | Requires a subscriptions app + customer re-consent to billing |
| Blog/pages content | Medium | CSV/Matrixify or copy manually; keep URLs for redirects |

**Order of operations:** products → collections → customers → orders → theme build (shopify-theme
skill) → apps/plugins (shopify-app SKILL.md) → redirects → DNS cutover.

---

## 3. Ongoing Sync: Selling on Multiple Channels

### Default answer: Shopify Marketplace Connect (free, first-party)

Connects **Amazon, eBay, Etsy, Walmart** to one Shopify admin:
- Push Shopify products out as channel listings (per-channel price/title overrides supported)
- Orders from all channels flow INTO Shopify as orders → one fulfillment queue
- Inventory decrements everywhere on every sale — the #1 overselling fix

Setup steps to give a merchant:
1. Shopify admin → Apps → install "Marketplace Connect"
2. Link each marketplace account (existing seller accounts keep their reviews/history)
3. Map existing marketplace listings to Shopify products (match by SKU — clean SKUs first!)
4. Choose sync policy: Shopify as source of truth for price/inventory (recommended)
5. Test: place one order per channel, confirm inventory drops everywhere

### When Marketplace Connect isn't enough

| Need | Use |
|---|---|
| WooCommerce/Wix running in parallel during transition | QuickSync, Nembol, or hold inventory in Shopify and feed the old store manually |
| Channel not covered (Faire, TikTok Shop, regional marketplaces) | Native channel apps from the App Store (TikTok, Faire have official ones) |
| ERP/warehouse is the real inventory master | Integration app or custom app: webhooks out, `inventorySetQuantities` in (see SKILL.md Phase 7) |
| Handmade/craft with supplies tracking | Craftybase-style tools alongside |

### Building sync into a custom app (when you're the developer)

- Outbound: subscribe `orders/create`, `inventory_levels/update` → queue → push to channel APIs.
- Inbound: channel webhooks/polling → `inventorySetQuantities`, `orderCreate` (for imported orders).
- Idempotency keys on every write; nightly reconciliation job comparing absolute quantities.
- Never sync computed stock between three systems pairwise — hub-and-spoke only (§4).

---

## 4. Source-of-Truth Architecture

**Rule: exactly ONE system owns each data type. Everything else is a read replica.**

```
            ┌────────────────────────────┐
            │   SHOPIFY (hub / truth)    │
            │ products · prices · stock  │
            └──┬──────┬──────┬──────┬────┘
               ▼      ▼      ▼      ▼
            Amazon   eBay   Etsy  Walmart      ← spokes: listings + incoming orders
```

Recommended ownership for a typical merchant:
- **Products, prices, inventory** → Shopify (or the ERP if one exists — then Shopify is a spoke too)
- **Orders** → created on each channel, but **consolidated into Shopify** for fulfillment
- **Customers** → Shopify (marketplaces won't give you buyer emails for marketing anyway)

Anti-patterns to refuse:
- Editing stock manually in two places "just this once"
- Pairwise sync (Etsy↔Shopify AND Etsy↔Amazon) — creates loops and race conditions
- Using marketplace stock as truth while running Shopify ads (overselling machine)

---

## 5. SEO & Cutover Checklist (leaving an old platform)

- [ ] Crawl old site → export all URLs (products, collections, pages, blog)
- [ ] Build 301 map → Shopify admin: **Online Store → Navigation → URL redirects** (bulk CSV or Matrixify)
- [ ] Match slugs where possible (`/product/foo` → `/products/foo`)
- [ ] Carry over meta titles/descriptions into Shopify SEO fields
- [ ] Keep old platform live but unindexed until Shopify verified, then flip DNS
- [ ] Re-verify Search Console + submit new sitemap (`/sitemap.xml`, automatic)
- [ ] Announce to customers: password-reset campaign (accounts) + "new site" email
- [ ] Watch 404 report for the first month; patch redirects

---

## 6. Headless / Hydrogen Decision

Sometimes "my other store" means "my main website is elsewhere (Next.js/Wordpress) and Shopify only
powers commerce." Options, cheapest first:

| Option | What it is | Choose when |
|---|---|---|
| **Liquid theme** (default) | Standard Shopify storefront | Almost always — fastest to ship, apps just work |
| **Buy Button / Storefront API widgets** | Embed Shopify cart in an existing site | Content site exists (WordPress etc.), light commerce |
| **Hydrogen + Oxygen** | React (React Router v7) storefront, Shopify edge hosting (free with paid plans) | Custom UX Liquid can't do, React team on staff, Shopify is long-term home |
| **Storefront API + own stack** (Next.js etc.) | Fully custom | Existing frontend investment; you own hosting/perf |

Hydrogen honesty check (recommend Liquid unless ALL THREE are true):
1. A specific, named UX problem Liquid cannot solve (configurator, real-time personalization,
   multi-brand single-backend).
2. Engineering capacity to absorb quarterly Storefront API version bumps forever.
3. Theme-app-store apps they need are API-first (theme-embed apps won't work headless).

If they go headless, checkout still runs on Shopify — Functions and checkout UI extensions from
extensions-reference.md apply unchanged.
