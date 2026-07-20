# Hydrogen Quickstart — Custom React Storefront

Only reach for this after the honesty check in multichannel-playbook.md §6 passes. Hydrogen is
Shopify's React Router v7-based framework for headless storefronts; **Oxygen** is the free
edge hosting included with every paid Shopify plan.

## Contents

1. [Scaffold & Run](#1-scaffold--run)
2. [Project Anatomy](#2-project-anatomy)
3. [Storefront API Queries](#3-storefront-api-queries)
4. [Cart](#4-cart)
5. [Customer Accounts](#5-customer-accounts)
6. [SEO, Caching, Analytics](#6-seo-caching-analytics)
7. [Deploy to Oxygen](#7-deploy-to-oxygen)
8. [Working Alongside Apps & Checkout](#8-working-alongside-apps--checkout)
9. [Maintenance Contract](#9-maintenance-contract)

---

## 1. Scaffold & Run

```bash
npm create @shopify/hydrogen@latest    # pick: TypeScript, tailwind if wanted, full-featured demo
cd my-storefront
npx shopify hydrogen link              # link to the store's Hydrogen channel (creates storefront)
npx shopify hydrogen env pull          # pulls PUBLIC_STOREFRONT_API_TOKEN etc. into .env
npm run dev                            # http://localhost:3000 with HMR + GraphiQL at /graphiql
```

Requires the **Hydrogen sales channel** installed on the store (free, App Store). The demo-store
template ships working PLP/PDP/cart/account — strip down rather than build up.

## 2. Project Anatomy

```
app/
├── root.tsx                 # layout, header/footer queries, <Analytics.Provider>
├── entry.server.tsx         # SSR + Content-Security-Policy
├── routes/
│   ├── _index.tsx           # home
│   ├── products.$handle.tsx # PDP
│   ├── collections.$handle.tsx
│   ├── cart.tsx             # cart route (CartForm actions)
│   └── account*.tsx         # customer account routes
├── lib/fragments.ts         # shared GraphQL fragments
└── components/
server.ts                    # Oxygen worker entry: context, session, cart handler
.env                         # tokens from `env pull` — never commit
```

Rendering model = React Router loaders/actions with **streaming SSR**: critical data awaited in
the loader, below-the-fold data deferred (`defer`) and streamed with `<Suspense>/<Await>`.

## 3. Storefront API Queries

```tsx
// app/routes/products.$handle.tsx
export async function loader({ params, context }) {
  const { product } = await context.storefront.query(PRODUCT_QUERY, {
    variables: { handle: params.handle },
    cache: context.storefront.CacheLong(),
  });
  if (!product) throw new Response(null, { status: 404 });
  return { product };
}

const PRODUCT_QUERY = `#graphql
  query Product($handle: String!) {
    product(handle: $handle) {
      id title descriptionHtml
      featuredImage { url altText width height }
      selectedOrFirstAvailableVariant {
        id availableForSale price { amount currencyCode }
      }
    }
  }
`;
```

- `context.storefront` is pre-authenticated (public token, safe client-side rates).
- Cache strategies per query: `CacheLong()` (1h, product/collection data), `CacheShort()` (1s–1m,
  inventory-ish), `CacheNone()` (cart, personalization).
- Use `@shopify/hydrogen` components where they exist: `<Image>`, `<Money>`, `<ShopPayButton>`,
  `getSelectedProductOptions` — they handle srcset/locale/currency correctly.

## 4. Cart

The skeleton wires a cart handler in `server.ts`; interact via `CartForm`:

```tsx
import { CartForm } from "@shopify/hydrogen";

<CartForm route="/cart" action={CartForm.ACTIONS.LinesAdd}
          inputs={{ lines: [{ merchandiseId: variantId, quantity: 1 }] }}>
  <button>Add to cart</button>
</CartForm>
```

The `/cart` route's action switches on `CartForm.getFormInput(formData)` → `context.cart.addLines /
updateLines / removeLines / updateDiscountCodes`. Checkout = redirect to `cart.checkoutUrl` —
**checkout itself always stays on Shopify** (with your checkout UI extensions + Functions intact).

## 5. Customer Accounts

Use the **Customer Account API** (OAuth, passwordless) — the skeleton's `account*.tsx` routes wire
login/callback/logout via `context.customerAccount`. Requires setting the callback URIs in the
Hydrogen channel settings. Don't build password forms; classic customer accounts are the legacy path.

## 6. SEO, Caching, Analytics

- Route `meta` exports for title/description; `[sitemap.xml]` and `[robots.txt]` routes come with
  the skeleton — keep them.
- Full-page/edge caching comes from Oxygen + your loader cache strategies; avoid `CacheNone()` on
  anything indexable.
- Analytics: wrap the app in `<Analytics.Provider>` (skeleton does) → publishes the same
  standardized events web pixels consume, so pixel-based tracking apps still work.
- Redirects: Oxygen honours the store's admin URL redirects; add route-level redirects for the rest.

## 7. Deploy to Oxygen

```bash
npx shopify hydrogen deploy        # manual deploy from CLI
```

Preferred: connect the GitHub repo in admin → Hydrogen channel → every push = preview deployment,
main branch = production, env vars managed per-environment in admin. Custom domain: point it at the
Hydrogen storefront in admin → Domains. Bring-your-own hosting (Vercel/Cloudflare) is possible
(`@shopify/remix-oxygen` swap) but you give up the free edge hosting + integrated envs.

## 8. Working Alongside Apps & Checkout

| Piece | On Hydrogen |
|---|---|
| Checkout UI extensions & Functions | Work unchanged (checkout is still Shopify) |
| Theme app extensions | **Do NOT work** — no theme. Need API-first apps or rebuild the widget against your app's proxy/API |
| App proxies | Work (they're storefront-domain routes) — call `/apps/...` from Hydrogen routes |
| Metafields | Readable via Storefront API if definition has `storefront: PUBLIC_READ` |
| Shopify admin content (pages/blogs/menus) | Query via Storefront API (`menu`, `blog`, `page`) |

## 9. Maintenance Contract

Adopting Hydrogen commits the team to: quarterly Storefront API version bumps (review breaking
changes each release), `@shopify/hydrogen` upgrades (`npx shopify hydrogen upgrade` shows a guided
changelog), and owning perf/accessibility that Liquid themes give for free. Budget ~1 dev-day per
quarter minimum. If that's not viable, the answer was Liquid (shopify-theme skill).
