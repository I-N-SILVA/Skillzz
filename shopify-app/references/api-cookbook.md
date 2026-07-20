# GraphQL Admin API Cookbook

Copy-paste recipes for the operations almost every app needs. All GraphQL, pinned to a quarterly
`api_version` — never REST. In the React Router template run these through
`admin.graphql(query, { variables })` after `authenticate.admin(request)`.

## Contents

1. [Pagination (cursor pattern)](#1-pagination-cursor-pattern)
2. [Products & Variants](#2-products--variants)
3. [Metafields & Metaobjects](#3-metafields--metaobjects)
4. [Orders & Fulfillment](#4-orders--fulfillment)
5. [Customers](#5-customers)
6. [Inventory](#6-inventory)
7. [Discounts](#7-discounts)
8. [Bulk Operations (large datasets)](#8-bulk-operations-large-datasets)
9. [Rate Limits & Error Handling](#9-rate-limits--error-handling)
10. [Useful Webhook Topics](#10-useful-webhook-topics)

---

## 1. Pagination (cursor pattern)

Never fetch more than 50 per page; loop on `hasNextPage`.

```graphql
query ($first: Int!, $after: String) {
  products(first: $first, after: $after, query: "status:active") {
    edges { cursor node { id title } }
    pageInfo { hasNextPage endCursor }
  }
}
```

```ts
let after: string | null = null;
const all = [];
do {
  const res = await admin.graphql(QUERY, { variables: { first: 50, after } });
  const { data } = await res.json();
  all.push(...data.products.edges.map((e) => e.node));
  after = data.products.pageInfo.hasNextPage ? data.products.pageInfo.endCursor : null;
} while (after);
```

The `query:` search syntax (`"status:active"`, `"created_at:>2026-01-01"`, `"sku:ABC*"`,
`"tag:sale"`) works on most list fields — filter server-side, not in your code.

---

## 2. Products & Variants

### Create/update product with options (productSet = idempotent upsert)
```graphql
mutation ($input: ProductSetInput!) {
  productSet(input: $input) {
    product { id variants(first: 10) { nodes { id sku } } }
    userErrors { field message }
  }
}
```
```json
{ "input": {
    "title": "Classic Tee",
    "productOptions": [{ "name": "Size", "values": [{ "name": "S" }, { "name": "M" }] }],
    "variants": [
      { "sku": "TEE-S", "price": "29.00", "optionValues": [{ "optionName": "Size", "name": "S" }] },
      { "sku": "TEE-M", "price": "29.00", "optionValues": [{ "optionName": "Size", "name": "M" }] }
    ]
} }
```

### Attach an image
```graphql
mutation ($productId: ID!, $media: [CreateMediaInput!]!) {
  productCreateMedia(productId: $productId, media: $media) {
    media { ... on MediaImage { id } }
    mediaUserErrors { field message }
  }
}
```
`media: [{ originalSource: "https://…/img.jpg", mediaContentType: IMAGE }]` — Shopify fetches and
CDN-hosts it.

---

## 3. Metafields & Metaobjects

Metafields = the glue between your app, the theme, and extensions (see SKILL.md Phase 7).

### Define once at install (so merchants see them in admin, pin-able, Theme-Editor-connectable)
```graphql
mutation {
  metafieldDefinitionCreate(definition: {
    name: "Rating", namespace: "$app:reviews", key: "rating",
    type: "number_decimal", ownerType: PRODUCT,
    access: { storefront: PUBLIC_READ }        # REQUIRED for Liquid/Storefront API to read it
  }) { createdDefinition { id } userErrors { field message } }
}
```

### Write values (up to 25 per call)
```graphql
mutation ($metafields: [MetafieldsSetInput!]!) {
  metafieldsSet(metafields: $metafields) {
    metafields { id }
    userErrors { field message }
  }
}
```
```json
{ "metafields": [{
    "ownerId": "gid://shopify/Product/123",
    "namespace": "$app:reviews", "key": "rating",
    "type": "number_decimal", "value": "4.8"
} ] }
```

In Liquid the `$app:` namespace appears as `app--{app-id}--reviews`:
`{{ product.metafields['app--123456--reviews'].rating.value }}`

### Metaobjects — app-defined data types (size charts, store locations, FAQ entries)
`metaobjectDefinitionCreate` → `metaobjectCreate` → reference from products via a
`metaobject_reference` metafield, or list them in Liquid via `shop.metaobjects.type_handle.values`.

---

## 4. Orders & Fulfillment

### Read orders with line items
```graphql
query ($first: Int!, $query: String) {
  orders(first: $first, query: $query, sortKey: CREATED_AT, reverse: true) {
    edges { node {
      id name createdAt displayFulfillmentStatus
      totalPriceSet { shopMoney { amount currencyCode } }
      customer { id email }
      lineItems(first: 50) { nodes { sku quantity title } }
      fulfillmentOrders(first: 5) { nodes { id status } }
    } }
    pageInfo { hasNextPage endCursor }
  }
}
```

### Fulfill (modern flow: fulfillment ORDERS, not legacy fulfillments)
```graphql
mutation ($fulfillment: FulfillmentInput!) {
  fulfillmentCreate(fulfillment: $fulfillment) {
    fulfillment { id status }
    userErrors { field message }
  }
}
```
```json
{ "fulfillment": {
    "lineItemsByFulfillmentOrder": [{ "fulfillmentOrderId": "gid://shopify/FulfillmentOrder/123" }],
    "trackingInfo": { "company": "UPS", "number": "1Z999...", "url": "https://..." },
    "notifyCustomer": true
} }
```

Requires `write_merchant_managed_fulfillment_orders` (or third-party equivalent) — not just `write_orders`.

### Refund calculation → execution
`refundCreate` after querying `order.suggestedRefund` for correct tax/shipping proration. Never
compute refund amounts yourself.

---

## 5. Customers

```graphql
mutation ($input: CustomerInput!) {
  customerCreate(input: $input) {
    customer { id email }
    userErrors { field message }
  }
}
```

- Search: `customers(query: "email:jane@example.com")`.
- Marketing consent is its own mutation: `customerEmailMarketingConsentUpdate` — setting it inside
  `customerCreate` without real consent violates policy.
- **Protected customer data:** public apps must request access (Partner Dashboard → API access) and
  declare purposes before order/customer PII flows; build and test with this ON from day one.

---

## 6. Inventory

Model: `InventoryItem` (per variant) × `Location` → `InventoryLevel` with quantity **states**
(`available`, `committed`, `on_hand`, `incoming`…).

### Absolute set (sync from an external truth — preferred for integrations)
```graphql
mutation ($input: InventorySetQuantitiesInput!) {
  inventorySetQuantities(input: $input) {
    inventoryAdjustmentGroup { reason }
    userErrors { field message }
  }
}
```
```json
{ "input": {
    "name": "available", "reason": "correction", "ignoreCompareQuantity": true,
    "quantities": [{
      "inventoryItemId": "gid://shopify/InventoryItem/123",
      "locationId": "gid://shopify/Location/456",
      "quantity": 42
    }]
} }
```

### Relative adjust (your app caused a delta)
`inventoryAdjustQuantities` with `delta`. Use absolute for reconciliation, relative for events —
never mix within one flow.

---

## 7. Discounts

| Kind | Mutation |
|---|---|
| Basic code discount (percent/amount) | `discountCodeBasicCreate` |
| Automatic discount (no code) | `discountAutomaticBasicCreate` |
| **Function-backed** custom logic | `discountAutomaticAppCreate` / `discountCodeAppCreate` (`functionId` required) |

Function-backed flow: deploy Function → create the discount pointing at `functionId` → write config
metafield the Function's input query reads (extensions-reference.md §4).

---

## 8. Bulk Operations (large datasets)

For anything over a few thousand records (full catalog export, all orders): one bulk query, poll,
download JSONL. No pagination loops, no rate-limit dance.

```graphql
mutation {
  bulkOperationRunQuery(query: """
    { products { edges { node { id title variants { edges { node { sku price } } } } } } }
  """) {
    bulkOperation { id status }
    userErrors { field message }
  }
}
```

Poll `currentBulkOperation { status url }` (or better: subscribe to the
`bulk_operations/finish` webhook) → download `url` → parse JSONL where child objects carry
`__parentId`. Bulk **imports**: `bulkOperationRunMutation` with a staged-upload JSONL of variables.
One bulk op of each kind runs at a time per shop — queue accordingly.

---

## 9. Rate Limits & Error Handling

- GraphQL is **cost-based**: ~1000 points, restoring 50–100/s (plan-dependent). Every response
  includes `extensions.cost` — log it.
- On throttle you get `errors[].extensions.code === "THROTTLED"` → wait
  `(requestedCost - available) / restoreRate` seconds, retry.
- Three error layers to handle EVERY call:
  1. HTTP/network failures → retry with backoff
  2. Top-level `errors[]` (query invalid, throttled, unauthorized)
  3. `userErrors[]` inside the mutation payload (business validation) → show to merchant

```ts
async function gql(admin, query, variables, retries = 3) {
  for (let i = 0; i <= retries; i++) {
    const res = await admin.graphql(query, { variables });
    const body = await res.json();
    const throttled = body.errors?.some((e) => e.extensions?.code === "THROTTLED");
    if (!throttled) return body;
    await new Promise((r) => setTimeout(r, 1000 * 2 ** i));
  }
  throw new Error("Throttled after retries");
}
```

---

## 10. Useful Webhook Topics

| Topic | Fire when | Common use |
|---|---|---|
| `orders/create` · `orders/paid` | New/paid order | Sync to ERP/channels, trigger fulfillment |
| `orders/fulfilled` · `orders/cancelled` | Status change | Notifications, channel updates |
| `products/create` · `products/update` · `products/delete` | Catalog change | Re-export listings, cache bust |
| `inventory_levels/update` | Stock change at a location | Multichannel sync |
| `customers/create` · `customers/update` | Customer change | CRM sync |
| `app/uninstalled` | Uninstall | Cleanup (mandatory) |
| `app_subscriptions/update` | Billing status change | Gate features on ACTIVE/CANCELLED |
| `bulk_operations/finish` | Bulk op done | Fetch JSONL result |
| `shop/update` | Shop settings change | Currency/plan-dependent behaviour |

Prefer **declarative subscriptions in `shopify.app.toml`** over runtime `webhookSubscriptionCreate`
— they version with `shopify app deploy` and can't drift per-shop.
