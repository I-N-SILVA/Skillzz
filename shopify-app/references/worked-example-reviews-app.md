# Worked Example — "Stardust Reviews" End to End

One complete build that exercises every pattern in the skill: embedded admin UI, own database,
metafields, a theme app extension, an app proxy, webhooks, and deploy. Adapt the shape; the
feature (product reviews) is just a realistic carrier.

**What it does:** shoppers submit reviews from the product page → merchant moderates in the
embedded admin → approved rating shows as stars in the theme, editable by the merchant in the
Theme Editor.

**Component map (this IS SKILL.md Phase 7 made concrete):**

```
Storefront (theme app extension block)
  ├─ reads avg rating from product metafield        ← zero-latency display
  ├─ fetches review list via app proxy (JSON)       ← live data from app DB
  └─ POSTs new reviews via app proxy                → app server validates + stores
App server (React Router + Prisma)
  ├─ admin routes: moderate queue (Polaris)
  ├─ on approve: recompute avg → metafieldsSet      → theme reads it next render
  └─ webhooks: products/delete (cascade), app/uninstalled + GDPR (cleanup)
```

---

## 1. Scaffold & data model

```bash
shopify app init            # React Router template → name: stardust-reviews
cd stardust-reviews
shopify app generate extension   # → Theme app extension → name: reviews-widget
```

```prisma
// prisma/schema.prisma (add below the Session model)
model Review {
  id         String   @id @default(cuid())
  shop       String
  productId  String                       // gid://shopify/Product/…
  rating     Int                          // 1–5
  author     String
  body       String
  status     String   @default("pending") // pending | approved | rejected
  createdAt  DateTime @default(now())

  @@index([shop, productId, status])
}
```

`shopify.app.toml` additions:

```toml
[access_scopes]
scopes = "write_products"        # metafields on products need product write

[app_proxy]
url = "https://your-app.example.com"
subpath = "stardust"
prefix = "apps"                  # storefront path: /apps/stardust/…
```

---

## 2. Metafield definition (run once per shop, at install)

In `afterAuth` (or an idempotent setup loader), define the metafield so it's visible in admin and
readable by Liquid:

```ts
await admin.graphql(`#graphql
  mutation {
    metafieldDefinitionCreate(definition: {
      name: "Average rating", namespace: "$app:reviews", key: "avg",
      type: "number_decimal", ownerType: PRODUCT,
      access: { storefront: PUBLIC_READ }
    }) { createdDefinition { id } userErrors { code message } }
  }`);
// Ignore the TAKEN error code — that just means it already exists (idempotent).
```

---

## 3. Admin moderation UI (Polaris)

```tsx
// app/routes/app._index.tsx
import { authenticate } from "../shopify.server";
import db from "../db.server";

export const loader = async ({ request }) => {
  const { session } = await authenticate.admin(request);
  const pending = await db.review.findMany({
    where: { shop: session.shop, status: "pending" },
    orderBy: { createdAt: "desc" },
  });
  return { pending };
};

export const action = async ({ request }) => {
  const { admin, session } = await authenticate.admin(request);
  const form = await request.formData();
  const id = form.get("id") as string;
  const verdict = form.get("verdict") as string;        // "approved" | "rejected"

  const review = await db.review.update({ where: { id }, data: { status: verdict } });

  if (verdict === "approved") {
    // Recompute and push the average into the product metafield
    const agg = await db.review.aggregate({
      where: { shop: session.shop, productId: review.productId, status: "approved" },
      _avg: { rating: true },
    });
    const res = await admin.graphql(`#graphql
      mutation ($metafields: [MetafieldsSetInput!]!) {
        metafieldsSet(metafields: $metafields) {
          userErrors { field message }
        }
      }`,
      { variables: { metafields: [{
          ownerId: review.productId,
          namespace: "$app:reviews", key: "avg",
          type: "number_decimal",
          value: String(agg._avg.rating ?? 0),
      }] } });
    const { data } = await res.json();
    if (data.metafieldsSet.userErrors.length) return { errors: data.metafieldsSet.userErrors };
  }
  return { ok: true };
};

// UI: Polaris <Page><Card><ResourceList> of pending reviews with Approve/Reject
// buttons submitting <Form method="post">; shopify.toast.show("Review approved") on success.
```

---

## 4. App proxy routes (storefront ↔ server)

```tsx
// app/routes/proxy.reviews.tsx  → shopper-facing at /apps/stardust/reviews
import { authenticate } from "../shopify.server";
import db from "../db.server";

export const loader = async ({ request }) => {
  const { session } = await authenticate.public.appProxy(request);   // verifies signature
  const productId = new URL(request.url).searchParams.get("product_id") ?? "";
  const reviews = await db.review.findMany({
    where: { shop: session.shop, productId: `gid://shopify/Product/${productId}`, status: "approved" },
    select: { rating: true, author: true, body: true, createdAt: true },
    orderBy: { createdAt: "desc" }, take: 20,
  });
  return Response.json({ reviews });
};

export const action = async ({ request }) => {
  const { session } = await authenticate.public.appProxy(request);
  const form = await request.formData();
  const rating = Number(form.get("rating"));
  const author = String(form.get("author") ?? "").slice(0, 80).trim();
  const body = String(form.get("body") ?? "").slice(0, 2000).trim();
  const productId = String(form.get("product_id") ?? "");
  if (!(rating >= 1 && rating <= 5) || !author || !body || !productId) {
    return Response.json({ error: "Invalid submission" }, { status: 422 });
  }
  await db.review.create({ data: {
    shop: session.shop, productId: `gid://shopify/Product/${productId}`,
    rating, author, body,                       // status defaults to "pending"
  } });
  return Response.json({ ok: true });
};
```

Validate everything — the proxy proves the request came through the shop's domain, not that the
payload is honest.

---

## 5. Theme app extension block

```liquid
{%- comment -%} extensions/reviews-widget/blocks/reviews.liquid {%- endcomment -%}
{{ 'reviews.css' | asset_url | stylesheet_tag }}

{%- assign avg = product.metafields['$app:reviews'].avg.value | default: 0 -%}
<div class="stardust" data-product-id="{{ product.id }}"
     style="--star-color: {{ block.settings.star_color }}">
  <div class="stardust__summary" aria-label="Rated {{ avg }} out of 5">
    {%- for i in (1..5) -%}
      <span class="stardust__star {% if i <= avg %}is-filled{% endif %}">★</span>
    {%- endfor -%}
    <span class="stardust__avg">{{ avg }}</span>
  </div>
  <div class="stardust__list" hidden></div>
  {%- if block.settings.allow_submissions -%}
    <button class="stardust__open" type="button">{{ block.settings.cta_label | escape }}</button>
    <form class="stardust__form" hidden>{%- comment -%} rating, author, body inputs {%- endcomment -%}</form>
  {%- endif -%}
</div>
<script src="{{ 'reviews.js' | asset_url }}" defer></script>

{% schema %}
{
  "name": "Product Reviews",
  "target": "section",
  "settings": [
    { "type": "color", "id": "star_color", "label": "Star colour", "default": "#f5a623" },
    { "type": "checkbox", "id": "allow_submissions", "label": "Allow new reviews", "default": true },
    { "type": "text", "id": "cta_label", "label": "Button label", "default": "Write a review" }
  ]
}
{% endschema %}
```

```js
// extensions/reviews-widget/assets/reviews.js — fetch + submit through the proxy
document.querySelectorAll(".stardust").forEach(async (el) => {
  const id = el.dataset.productId;
  const res = await fetch(`/apps/stardust/reviews?product_id=${id}`);
  const { reviews } = await res.json();
  // render list; wire form submit → POST /apps/stardust/reviews (FormData incl. product_id)
});
```

Notes: the metafield renders server-side (no flicker, SEO-visible); the review list hydrates after
load; the merchant controls colour/CTA per-placement in the Theme Editor. Onboarding deep link:
`…/themes/current/editor?template=product&addAppBlockId={uuid}/reviews&target=mainSection`.

---

## 6. Webhooks

`shopify.app.toml`:

```toml
  [[webhooks.subscriptions]]
  topics = ["products/delete"]
  uri = "/webhooks/products/delete"
```

```tsx
// app/routes/webhooks.products.delete.tsx — cascade-delete reviews for removed products
export const action = async ({ request }) => {
  const { shop, payload } = await authenticate.webhook(request);
  await db.review.deleteMany({
    where: { shop, productId: `gid://shopify/Product/${payload.id}` },
  });
  return new Response();
};
```

Template already routes `app/uninstalled` (delete sessions — extend it to delete the shop's
reviews after the grace period) and the three GDPR topics: `customers/data_request` → export
reviews matching the customer email; `customers/redact` → anonymize author fields; `shop/redact`
→ `deleteMany({ where: { shop } })`.

---

## 7. Ship it

```bash
shopify app dev          # dev store: place block via deep link, submit + approve a review
shopify app deploy       # versions app config + extension together
```

Production checklist for THIS app: Prisma → Postgres · rate-limit the proxy POST (per-IP) to stop
review spam · consider `orders/paid` webhook to mark "verified buyer" · billing + listing per
references/billing-launch.md.

**Extension ideas that reuse the same skeleton:** size-guide app (metaobjects instead of DB rows) ·
FAQ/Q&A app (same moderation loop) · wishlist (proxy + customer id) · back-in-stock alerts
(proxy POST + `inventory_levels/update` webhook → email).
