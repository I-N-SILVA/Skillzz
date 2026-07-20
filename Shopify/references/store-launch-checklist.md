# Store Launch Checklist & Recommended App Stack

Everything AFTER the theme is built: store configuration, content, apps, QA, and go-live. Work top
to bottom — items are ordered by dependency.

## Contents

1. [Store Settings](#1-store-settings)
2. [Payments, Shipping, Taxes](#2-payments-shipping-taxes)
3. [Legal & Trust Pages](#3-legal--trust-pages)
4. [Domain, Email, Analytics](#4-domain-email-analytics)
5. [Content & Merchandising](#5-content--merchandising)
6. [Recommended App Stack](#6-recommended-app-stack)
7. [Pre-Launch QA](#7-pre-launch-qa)
8. [Go-Live & First Week](#8-go-live--first-week)

---

## 1. Store Settings

- [ ] Store name, contact email, sender email (Settings → General / Notifications)
- [ ] Timezone, currency, unit system, order ID prefix
- [ ] Customer accounts: choose **new customer accounts** (passwordless) unless an app requires classic
- [ ] Markets: set primary market; add regions + currencies only if actually shipping there
- [ ] Notification emails: re-brand at least Order confirmation + Shipping confirmation
  (Settings → Notifications — Liquid templates, match theme colours/logo)
- [ ] Checkout: contact method, address options, tipping off/on, marketing consent checkboxes

## 2. Payments, Shipping, Taxes

- [ ] Shopify Payments activated (ID verification early — payouts hold until done); enable
  Shop Pay, Apple Pay, Google Pay
- [ ] PayPal (or regional equivalents) as secondary
- [ ] Test order via Shopify Payments **test mode**, then a REAL $1 transaction refunded
- [ ] Shipping zones + rates: weight/price-based or flat; free-shipping threshold consistent with
  theme messaging (and any Function logic)
- [ ] Package dimensions/weights on products (accurate carrier rates + customs)
- [ ] Local pickup/delivery if relevant
- [ ] Taxes: enable collection per registration (US: only where nexus; EU: OSS; UK: VAT);
  prices-include-tax setting matches the market convention
- [ ] Duties/customs for international (HS codes on products)

## 3. Legal & Trust Pages

Generate drafts (Settings → Policies has templates), then have a human review:
- [ ] Privacy policy · Terms of service · Refund/return policy · Shipping policy
- [ ] Cookie banner if EU traffic (Shopify's Customer Privacy settings + banner)
- [ ] Link all policies in footer menu; return policy linked from PDP/FAQ
- [ ] Contact page with a real reply-to (form or email) — required for many payment providers

## 4. Domain, Email, Analytics

- [ ] Custom domain connected (Settings → Domains), `www` vs apex chosen, SSL green
- [ ] Old-platform redirects imported BEFORE DNS flip (see shopify-app multichannel-playbook §5)
- [ ] Sender domain authenticated (SPF/DKIM records) so order emails don't hit spam
- [ ] Google Analytics 4 via the official **Google & YouTube** channel app
- [ ] Meta pixel via official **Facebook & Instagram** app (never hand-pasted pixel code in theme.liquid)
- [ ] Google Search Console verified + sitemap `/sitemap.xml` submitted
- [ ] Shopify Inbox or support channel decided

## 5. Content & Merchandising

- [ ] All products: title, description, ≥ 3 images with alt text, price, compare-at where honest,
      SKU, barcode, weight, inventory tracked, status Active
- [ ] Collections built (manual for curation, automated by tag/type for scale) with images + SEO text
- [ ] Navigation: main menu ≤ 7 top-level items; footer menu with policies/about/contact
- [ ] Homepage sections populated with REAL content (no lorem ipsum, no placeholder images)
- [ ] About page tells an actual story; FAQ answers shipping/returns/sizing
- [ ] Blog: minimum 1–3 posts if the section exists — an empty blog looks abandoned
- [ ] 404 page customized with a path back to collections
- [ ] SEO fields (title/meta description) on homepage, top collections, top products
- [ ] Image weights sane: hero < 500KB, products < 300KB (Shopify serves WebP/AVIF automatically,
      but source files still matter)

## 6. Recommended App Stack

Install the minimum. Every app adds JS weight, monthly cost, and a support surface.

| Job | Solid picks | Note |
|---|---|---|
| Email marketing / flows | Klaviyo · Shopify Email | Shopify Email is free-tier friendly |
| Reviews | Judge.me · Loox · Yotpo | Judge.me best free tier |
| Sell on marketplaces | **Marketplace Connect** (first-party) | Amazon/eBay/Etsy/Walmart |
| Subscriptions | Recharge · Shopify Subscriptions (free) | Native app fine for simple cases |
| Bundles/upsell | Shopify Bundles (free) · Rebuy | Prefer Function-based bundle apps |
| Page builder for landing pages | Only if theme sections can't do it | Try theme sections first — builders add weight |
| Back-in-stock alerts | Klaviyo native or dedicated app | |
| Loyalty | Smile.io · Rivo | Wait until repeat-purchase volume justifies it |
| Support/chat | Shopify Inbox (free) · Gorgias | |
| Bulk data ops | Matrixify | Imports/exports/redirects at scale |

**Rules:** prefer first-party free apps → theme-app-extension-based apps (no theme code edits) →
everything else. After EVERY install re-run Lighthouse; uninstall anything costing > 5 points that
isn't earning revenue. On uninstall, check for leftover code in theme.liquid (older apps litter).

## 7. Pre-Launch QA

- [ ] `shopify theme check` — zero errors
- [ ] Full purchase on mobile AND desktop: browse → PDP → variant switch → cart → discount code →
      checkout → order email received → refund flow
- [ ] Every nav link, footer link, and button clicked (no dead links, no "#")
- [ ] Forms tested: contact, newsletter (check the double-opt-in email)
- [ ] Lighthouse mobile: LCP < 2.5s on home/PLP/PDP; images lazy-loaded below fold
- [ ] Accessibility pass: focus states visible, alt text present, contrast ≥ 4.5:1, one `h1` per page
- [ ] Theme Editor sanity: every section's settings actually work; no orphan settings
- [ ] Browser sweep: Safari iOS (the one that breaks), Chrome Android, desktop trio
- [ ] Password page OFF only at the moment of launch

## 8. Go-Live & First Week

- [ ] Remove password page; verify robots.txt isn't blocking; confirm indexing in Search Console
- [ ] Announce (email list, socials); enable abandoned-checkout email (Marketing → Automations)
- [ ] Watch first orders end-to-end: payment captured, notification sent, fulfillment flows
- [ ] Daily for 7 days: 404 report (patch redirects), speed, app error logs, checkout analytics
      (Analytics → Sessions by landing page / conversion funnel)
- [ ] Collect the first 3 customer feedback notes — they will find what QA missed
