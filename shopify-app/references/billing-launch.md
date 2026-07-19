# Billing & App Store Launch Playbook

How to charge for a Shopify app and get it listed, approved, and growing. Applies to **public apps**;
custom apps skip billing (invoice the client directly) and skip review entirely.

## Contents

1. [Pricing Models](#1-pricing-models)
2. [Managed Pricing vs Billing API](#2-managed-pricing-vs-billing-api)
3. [Billing API Recipes](#3-billing-api-recipes)
4. [Gating Features on Subscription Status](#4-gating-features-on-subscription-status)
5. [App Review — What Actually Gets Apps Rejected](#5-app-review--what-actually-gets-apps-rejected)
6. [Listing Optimization](#6-listing-optimization)
7. [Launch Sequence](#7-launch-sequence)

---

## 1. Pricing Models

| Model | Fits | Notes |
|---|---|---|
| Free | Lead-gen for a service, ecosystem play | Skips billing entirely; still full review |
| Flat monthly subscription | Most apps | Easiest to build and to buy; offer 2–3 tiers |
| Tiered by usage caps (orders/mo, SKUs) | Sync/ops apps | Tie the metric to merchant value, not your costs |
| Usage/metered charges | Per-SMS, per-label, per-render | Requires capped amount; merchant approves the cap |
| One-time charge | Setup fees, lifetime deals | `appPurchaseOneTimeCreate`; rare as primary model |
| Free plan + paid tiers (freemium) | Widget apps competing on installs | Best review-count engine |

Rules of thumb: anchor to merchant revenue impact, not effort · always include a **free trial**
(7–14 days) · annual option at ~2 months off · Shopify takes 0% on the first $1M/yr
(registration required in Partner Dashboard), 15% after.

---

## 2. Managed Pricing vs Billing API

| | **Managed pricing** (recommended default) | **Billing API** |
|---|---|---|
| Setup | Define plans in Partner Dashboard; Shopify hosts plan-picker page | You code `appSubscriptionCreate` flows |
| Code needed | Almost none — check active plan, gate features | Mutations, confirmation redirects, callbacks |
| Flexibility | Fixed plans, trials, discounts | Dynamic/negotiated pricing, usage charges, per-merchant deals |
| Use when | Standard tiered subscriptions | Usage-based, custom enterprise deals, in-app upsell flows |

Start with managed pricing; move to the Billing API only when a need appears that managed can't do.

---

## 3. Billing API Recipes

### Create a subscription (merchant must approve via `confirmationUrl`)
```graphql
mutation {
  appSubscriptionCreate(
    name: "Pro Plan"
    returnUrl: "https://your-app.example.com/app/billing/confirm"
    trialDays: 14
    test: true            # ← true on dev stores; NEVER ship true to production
    lineItems: [{
      plan: { appRecurringPricingDetails: {
        price: { amount: 19.00, currencyCode: USD }
        interval: EVERY_30_DAYS
      } }
    }]
  ) {
    appSubscription { id status }
    confirmationUrl
    userErrors { field message }
  }
}
```
Redirect the merchant to `confirmationUrl` (App Bridge `open`), then verify status on return —
never assume approval.

### Usage-based add-on (requires a capped amount)
```graphql
lineItems: [{
  plan: { appUsagePricingDetails: {
    terms: "$0.05 per SMS sent"
    cappedAmount: { amount: 50.00, currencyCode: USD }
  } }
}]
```
Then bill per event: `appUsageRecordCreate(subscriptionLineItemId, price, description)` — rejected
automatically if it would exceed the cap.

### Check current subscription (on every app load)
```graphql
query {
  currentAppInstallation {
    activeSubscriptions { id name status trialDays currentPeriodEnd test }
  }
}
```

---

## 4. Gating Features on Subscription Status

- Gate **server-side** in loaders/actions — UI-only gating is bypassable and fails review economics.
- Subscribe to `app_subscriptions/update` webhook: on `CANCELLED`/`EXPIRED`/`FROZEN` flip the shop
  to the free tier in your DB; on `ACTIVE` restore.
- Uninstall cancels the subscription automatically; reinstall = new subscription (handle returning
  shops gracefully — keep their data through the `shop/redact` window, then delete).
- The template pattern: a `requireBilling` helper in `shopify.server.ts` that redirects to your
  pricing route when no active subscription matches the route's required plan.

---

## 5. App Review — What Actually Gets Apps Rejected

Ranked by how often they bite:

1. **Broken install/onboarding on a fresh dev store** — reviewers install cold. Test the zero-state.
2. **Missing/failing GDPR webhooks** — all three compliance topics must respond 200 with HMAC verified.
3. **Non-embedded feel** — leaves the iframe, no App Bridge nav, non-Polaris UI, broken back button.
4. **Scope over-reach** — requesting `write_customers` "just in case". Every scope must map to a
   visible feature.
5. **Billing bypass** — features usable without the charge, or charge created without approval flow.
6. **Performance** — app home > 3s load, Lighthouse hit on storefront (theme extension JS must be
   deferred, < 10KB ideal), console errors.
7. **Listing/app mismatch** — screenshots or copy promising features that aren't in the build.
8. **Protected customer data not configured** — apps touching orders/customers must declare data
   purposes in Partner Dashboard and pass the automated check.

Process reality: first response typically days-to-2-weeks; expect 1–3 rejection rounds; answer
precisely and re-submit fast — reviews resume, they don't restart.

---

## 6. Listing Optimization

- **App name**: function-first, ≤ 30 chars; keyword-stuffing gets rejected ("Stockly — Inventory
  Sync" beats "Inventory Sync Stock Alert Manager Pro").
- **Tagline + first paragraph**: merchant outcome, not features ("Never oversell on Etsy again").
- **Key benefit blocks (3)**: one merchant problem each, with a screenshot showing it solved.
- **Screenshots**: real UI, annotated, 1600×900; first one must make sense at thumbnail size.
- **Demo video** (< 60s) measurably lifts installs; screen-record the golden path.
- **Pricing clarity**: name limits explicitly per tier — vague tiers cause uninstalls + bad reviews.
- **Reviews flywheel**: in-app ask after a success moment (never at install), respond to every
  negative review publicly + fix.
- Categories & search: pick the category merchants browse, mention integration names ("Etsy",
  "Klaviyo") in the description body — App Store search indexes them.

---

## 7. Launch Sequence

- [ ] `shopify app deploy` final version; `application_url` on production host (not tunnel)
- [ ] Prisma on Postgres/MySQL; backups on; error tracking (Sentry) wired
- [ ] Billing: plans live, `test: false`, 0%-fee registration done in Partner Dashboard
- [ ] All webhooks verified in Partner Dashboard delivery metrics (0 failures)
- [ ] Fresh-store walkthrough: install → onboard → core value in < 5 minutes, no docs needed
- [ ] Listing assets uploaded; support email + privacy policy + docs URL live
- [ ] Submit for review; monitor Partner Dashboard for reviewer feedback daily
- [ ] Post-approval: install on 3–5 friendly merchant stores before any promotion
- [ ] Instrument: track installs → activation (first value event) → paid conversion → churn
- [ ] Iterate weekly; `shopify app deploy` ships config+extensions atomically, server deploys are yours
