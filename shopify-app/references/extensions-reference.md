# Shopify App Extensions — Deep Reference

Code templates for every extension surface. Generate each with `shopify app generate extension`
inside the app project; ship all of them with `shopify app deploy`.

## Contents

1. [Theme App Extensions](#1-theme-app-extensions)
2. [App Proxy (storefront → your server)](#2-app-proxy-storefront--your-server)
3. [Checkout UI Extensions](#3-checkout-ui-extensions)
4. [Shopify Functions](#4-shopify-functions)
5. [Customer Account UI Extensions](#5-customer-account-ui-extensions)
6. [Admin UI Extensions](#6-admin-ui-extensions)
7. [Web Pixels](#7-web-pixels)
8. [Flow Triggers & Actions](#8-flow-triggers--actions)
9. [Choosing the Right Surface](#9-choosing-the-right-surface)

---

## 1. Theme App Extensions

The ONLY sanctioned way for an app to render in a merchant's theme. Never edit theme files from an
app — that model is dead and fails review.

### Structure
```
extensions/my-widget/
├── shopify.extension.toml       # type = "theme"
├── blocks/
│   ├── star-rating.liquid       # app block  — merchant places it per-section
│   └── app-embed.liquid         # app embed  — floating/global (chat bubble, analytics)
├── snippets/
├── assets/                      # css/js/images — reference with {{ 'file.css' | asset_url }}
└── locales/
```

### App block template
```liquid
{%- comment -%} blocks/star-rating.liquid {%- endcomment -%}
{{ 'star-rating.css' | asset_url | stylesheet_tag }}

<div class="star-rating" data-product-id="{{ product.id }}"
     style="--star-color: {{ block.settings.star_color }}">
  {%- render 'stars', rating: product.metafields.app--123456.rating.value -%}
</div>

<script src="{{ 'star-rating.js' | asset_url }}" defer></script>

{% schema %}
{
  "name": "Star Rating",
  "target": "section",
  "settings": [
    { "type": "color", "id": "star_color", "label": "Star colour", "default": "#f5a623" }
  ]
}
{% endschema %}
```

- `"target": "section"` → app block (placeable in any JSON-template section that accepts app blocks)
- `"target": "body"` (in an embed block, `"target": "head"` also allowed) → app embed, toggled in
  Theme Editor → App embeds
- Block settings behave exactly like theme section settings — merchants edit them visually.

### Rules
- All assets via `asset_url` from the extension's own `assets/` — CDN-hosted by Shopify.
- Read app data via **metafields your app wrote** (`$app:` reserved namespace becomes
  `app--{your-app-id}` in Liquid) — zero-latency, no API call.
- Live/dynamic data → fetch from the **app proxy** (next section), never a hardcoded external URL.
- Give merchants a one-click setup deep link:
  `https://admin.shopify.com/store/{store}/themes/current/editor?template=product&addAppBlockId={uuid}/{handle}&target=mainSection`

---

## 2. App Proxy (storefront → your server)

Lets storefront JS call your app server on the shop's own domain: shopper hits
`https://shop.com/apps/my-proxy/reviews` → Shopify forwards to your server with a signed query.

### Configure in `shopify.app.toml`
```toml
[app_proxy]
url = "https://your-app.example.com/proxy"
subpath = "my-proxy"
prefix = "apps"
```

### Handle in the app (signature is verified for you)
```tsx
// app/routes/proxy.reviews.tsx
import { authenticate } from "../shopify.server";

export const loader = async ({ request }) => {
  const { session, liquid } = await authenticate.public.appProxy(request);
  const url = new URL(request.url);
  const productId = url.searchParams.get("product_id");
  const reviews = await db.review.findMany({ where: { shop: session.shop, productId } });
  return Response.json({ reviews });
};
```

Return `liquid("...")` instead of JSON to render Liquid processed by the shop (rare but powerful).

---

## 3. Checkout UI Extensions

React components rendered at **targets** across checkout, Thank-you, and Order-status pages.
Sandboxed: no DOM access, only `@shopify/ui-extensions-react/checkout` components.

### `shopify.extension.toml`
```toml
api_version = "2026-04"

[[extensions]]
type = "ui_extension"
name = "free-shipping-bar"
handle = "free-shipping-bar"

  [[extensions.targeting]]
  target = "purchase.checkout.block.render"     # merchant places it anywhere in checkout
  module = "./src/Checkout.tsx"
```

Common targets: `purchase.checkout.block.render` (placeable), `purchase.checkout.delivery-address.render-before`,
`purchase.checkout.shipping-option-list.render-after`, `purchase.thank-you.block.render`,
`customer-account.order-status.block.render`.

### Component
```tsx
import {
  reactExtension, Banner, useTotalAmount, useSettings,
} from "@shopify/ui-extensions-react/checkout";

export default reactExtension("purchase.checkout.block.render", () => <FreeShippingBar />);

function FreeShippingBar() {
  const { amount, currencyCode } = useTotalAmount();
  const { threshold = 75 } = useSettings();     // merchant-configurable in checkout editor
  const remaining = Number(threshold) - Number(amount);
  if (remaining <= 0) return <Banner status="success" title="You've unlocked free shipping!" />;
  return <Banner title={`Add ${remaining.toFixed(2)} ${currencyCode} more for free shipping`} />;
}
```

### Rules
- Network calls only to your server (declare in toml `[extensions.capabilities] network_access = true`)
  or via `useApi().query` for Storefront API data.
- Merchant-editable settings go in `[extensions.settings]` in the toml — mirror of theme block settings.
- Checkout-step targets on the info/shipping/payment pages work on all plans for apps; advanced
  branding & some targets are Plus-only — check the target's docs before promising a feature.
- Test with `shopify app dev` → checkout preview link; unit-test with `@shopify/ui-extensions-tester`.

---

## 4. Shopify Functions

Server-side logic compiled to Wasm, executed BY Shopify at checkout speed (<5ms budget).
Written in **Rust (recommended)** or **JavaScript**. No network access, no state — pure
input → output transforms.

| Function API | Replaces | Example |
|---|---|---|
| Discount | Scripts line-item discounts | Tiered "spend $100 save 15%" |
| Delivery customization | Scripts shipping | Hide express shipping for PO boxes |
| Payment customization | Scripts payment | Hide COD above $200 |
| Cart & checkout validation | — | Block checkout if quantity > stock cap |
| Cart transform | — | Bundles: merge/expand line items |
| Fulfillment constraints | — | Force items to ship from same location |

### Anatomy (discount, JS flavour)
```
extensions/volume-discount/
├── shopify.extension.toml         # type = "function", api = "discount"
├── src/
│   ├── run.graphql                # INPUT QUERY — what Shopify feeds your function
│   └── run.js                     # the transform
```

```graphql
# src/run.graphql — you choose the input; metafields carry merchant config
query RunInput {
  cart {
    lines { id quantity merchandise { __typename ... on ProductVariant { id } } }
  }
  discountNode { metafield(namespace: "$app:volume", key: "config") { value } }
}
```

```js
// src/run.js
export function run(input) {
  const config = JSON.parse(input.discountNode.metafield?.value ?? "{}");
  const discounts = input.cart.lines
    .filter((line) => line.quantity >= (config.minQty ?? 3))
    .map((line) => ({
      targets: [{ cartLine: { id: line.id } }],
      value: { percentage: { value: config.percent ?? 10 } },
    }));
  return { discounts, discountApplicationStrategy: "ALL" };
}
```

### Wiring it live (the step everyone forgets)
A deployed Function is inert until your app registers it:
```graphql
mutation {
  discountAutomaticAppCreate(automaticAppDiscount: {
    title: "Volume discount",
    functionId: "YOUR_FUNCTION_ID",
    startsAt: "2026-01-01T00:00:00Z"
  }) { userErrors { field message } }
}
```
Then your admin UI writes the config metafield (`$app:volume/config`) that the input query reads.
Test locally: `shopify app function run` with a JSON input fixture.

---

## 5. Customer Account UI Extensions

Same model as checkout UI extensions (React, targets, sandboxed) but rendered in the new customer
account pages. Targets like `customer-account.order-status.block.render`,
`customer-account.profile.block.render`. Use for: returns/exchanges, subscription management,
loyalty balances, warranty registration. Auth is automatic — the extension runs in the logged-in
customer's context; call your server for app data (network_access) keyed by
`useApi().authenticatedAccount.customer`.

---

## 6. Admin UI Extensions

React extensions INSIDE Shopify admin pages — no iframe app navigation needed for quick actions.

- **Admin action** (`admin.product-details.action.render`): menu-item → modal on a resource page.
  E.g. "Generate SEO description" on the product page.
- **Admin block** (`admin.order-details.block.render`): persistent card on the resource page.
  E.g. show your app's fraud score on every order.

```tsx
import { reactExtension, AdminAction, Button, Text } from "@shopify/ui-extensions-react/admin";

export default reactExtension("admin.product-details.action.render", () => <App />);

function App() {
  const { close, data } = useApi();      // data.selected → current product id
  return (
    <AdminAction
      primaryAction={<Button onPress={generate}>Generate</Button>}
      secondaryAction={<Button onPress={close}>Cancel</Button>}>
      <Text>Generate an AI description for this product?</Text>
    </AdminAction>
  );
}
```

Direct Admin GraphQL access is available inside admin extensions via `query` — same scopes as the app.

---

## 7. Web Pixels

Sandboxed analytics workers subscribing to standardized customer events — the only reliable way to
track checkout since direct script injection died with checkout.liquid.

```js
// extensions/my-pixel/src/index.js
import { register } from "@shopify/web-pixels-extension";

register(({ analytics, settings }) => {
  analytics.subscribe("checkout_completed", (event) => {
    fetch(`https://your-server.com/collect?acct=${settings.accountID}`, {
      method: "POST", keepalive: true,
      body: JSON.stringify({
        orderId: event.data.checkout.order?.id,
        total: event.data.checkout.totalPrice?.amount,
      }),
    });
  });
});
```

Activate per-shop via the `webPixelCreate` mutation (with the merchant's settings). Events:
`page_viewed`, `product_viewed`, `product_added_to_cart`, `checkout_started`, `checkout_completed`, etc.

---

## 8. Flow Triggers & Actions

Make your app a lego brick in merchants' own automations (Shopify Flow).

- **Trigger**: your server fires `flowTriggerReceive` mutation when something happens in YOUR app
  ("Review received") → merchants build any workflow off it.
- **Action**: toml-defined form + HTTPS endpoint on your server; Flow calls it with the merchant's
  configured payload ("Send coupon via MyApp").

Cheap to build, huge review-listing value — most apps skip them; don't.

---

## 9. Choosing the Right Surface

| Requirement | Wrong answer | Right answer |
|---|---|---|
| Show widget in theme | Inject `<script>` into theme.liquid | Theme app extension block |
| Track checkout conversion | Script tag on checkout | Web pixel |
| Custom discount stacking | Scripts (sunset 6/2026) | Discount Function |
| Hide a payment method | Checkout UI extension | Payment customization Function (logic ≠ UI) |
| Upsell in checkout | Edit checkout.liquid | Checkout UI extension (`purchase.checkout.block.render`) |
| Post-purchase offer | Thank-you page hack | `purchase.post.purchase` extension surface |
| Show data on admin order page | Standalone app page | Admin block |
| Storefront needs live app data | Direct calls to your API w/ keys in JS | App proxy |

**Logic vs pixels rule of thumb:** if it changes *what the buyer pays or can choose* → Function;
if it changes *what the buyer sees* → UI extension; if it *records what happened* → web pixel.
