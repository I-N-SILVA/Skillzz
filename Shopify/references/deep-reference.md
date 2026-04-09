# Shopify Theme — Deep Reference

Read sections as needed. Use the table of contents to jump directly.

## Contents

1. [Metafields & Custom Data](#1-metafields--custom-data)
2. [App Blocks & Theme Extension Points](#2-app-blocks--theme-extension-points)
3. [Dynamic Sources & Metaobjects](#3-dynamic-sources--metaobjects)
4. [Predictive Search & AJAX API](#4-predictive-search--ajax-api)
5. [Theme Store Submission Checklist](#5-theme-store-submission-checklist)
6. [Internationalisation (i18n)](#6-internationalisation-i18n)
7. [Section Groups (Header / Footer)](#7-section-groups-header--footer)
8. [Cart Behaviour Patterns](#8-cart-behaviour-patterns)

---

## 1. Metafields & Custom Data

### Access in Liquid
```liquid
{{ product.metafields.custom.care_instructions.value }}
```

### With fallback
```liquid
{%- assign care = product.metafields.custom.care_instructions.value -%}
{%- unless care == blank -%}
  <p class="care-instructions">{{ care }}</p>
{%- endunless -%}
```

### Metafield types → Liquid output
| Type | Output |
|------|--------|
| `single_line_text_field` | String |
| `multi_line_text_field` | String — use `\| newline_to_br` |
| `rich_text_field` | HTML — output raw (already sanitised) |
| `number_integer` | Integer |
| `boolean` | `true` / `false` |
| `color` | Hex string |
| `url` | String |
| `file_reference` → image | Image object — use `image_url`, `image_tag` |
| `list.*` | Array — iterate with `for` |

### Connect in schema (merchants wire via Theme Editor ⚡ icon)
```json
{ "type": "text", "id": "badge_label", "label": "Badge label",
  "info": "Or connect to a metafield via the Theme Editor" }
```

---

## 2. App Blocks & Theme Extension Points

### Enable app blocks in a section
```json
"blocks": [
  { "type": "@app" }
]
```

### Enable app blocks on a template
```json
{
  "sections": {
    "main":            { "type": "main-product" },
    "app-extensions":  { "type": "@app" }
  },
  "order": ["main", "app-extensions"]
}
```

Always add `"@app"` to product, collection, and article templates — it's a
Theme Store requirement and lets merchants install app widgets without touching code.

---

## 3. Dynamic Sources & Metaobjects

```liquid
{%- assign team = shop.metaobjects.team_member.values -%}
{%- for member in team -%}
  <div class="team-card">
    <h3>{{ member.name.value }}</h3>
    <p>{{ member.role.value }}</p>
    {{- member.photo.value | image_url: width: 400 | image_tag: loading: 'lazy' -}}
  </div>
{%- endfor -%}
```

### Metaobject reference in schema
```json
{ "type": "metaobject", "id": "team_member", "label": "Team member",
  "metaobject_type": "team_member" }
```

---

## 4. Predictive Search & AJAX API

### Search endpoint
```
GET /search/suggest.json
  ?q={query}
  &resources[type]=product,article
  &resources[limit]=6
  &resources[options][unavailable_products]=last
```

### Minimal implementation
```js
async function fetchSuggestions(q) {
  const url = `/search/suggest.json?q=${encodeURIComponent(q)}`
    + `&resources[type]=product&resources[limit]=6`;
  const res = await fetch(url, { headers: { 'Content-Type': 'application/json' } });
  return res.json();
}
```

### Cart AJAX
```js
// Add to cart
await fetch('/cart/add.js', {
  method: 'POST',
  headers: { 'Content-Type': 'application/json' },
  body: JSON.stringify({ id: variantId, quantity: 1 })
});

// Update quantity
await fetch('/cart/change.js', {
  method: 'POST',
  headers: { 'Content-Type': 'application/json' },
  body: JSON.stringify({ id: variantId, quantity: newQty })
});

// Fetch cart state
const cart = await (await fetch('/cart.js')).json();
```

---

## 5. Theme Store Submission Checklist

### Functionality
- [ ] All page types have JSON templates (product, collection, blog, article, page, cart, 404, password, gift_card, index)
- [ ] Every section has a `presets` entry
- [ ] All content editable via Theme Editor — zero hardcoded merchant content
- [ ] `"@app"` blocks enabled on product, collection, article templates
- [ ] Cart works with both AJAX drawer and redirect fallback
- [ ] Variant selectors work (size, colour, material…)
- [ ] Theme functions with any combination of active sections

### Code quality
- [ ] `shopify theme check` → zero errors, zero warnings
- [ ] No `{% include %}` — only `{% render %}`
- [ ] No deprecated Liquid tags or filters
- [ ] No hardcoded store-specific data

### Performance
- [ ] Mobile Lighthouse Performance ≥ 60 (target 80+)
- [ ] LCP image: `loading="eager"`, `fetchpriority="high"`
- [ ] All other images: `loading="lazy"` + `srcset`
- [ ] No render-blocking resources

### Accessibility (WCAG 2.1 AA)
- [ ] Colour contrast ≥ 4.5:1 body, ≥ 3:1 large text / UI
- [ ] Fully keyboard-navigable
- [ ] Skip-to-content link as first focusable element
- [ ] `aria-*` on all interactive components (dropdowns, modals, tabs)
- [ ] Focus styles visible — never `outline: none` without a replacement

### Merchant experience
- [ ] Tested at 320px, 375px, 768px, 1280px, 1440px
- [ ] All colours driven by `color_scheme` / settings (no hardcoded hex)
- [ ] All fonts driven by `font_picker` settings
- [ ] `config/settings_data.json` ships with polished demo content

---

## 6. Internationalisation (i18n)

### Translation key in schema
```json
{ "type": "text", "id": "heading", "label": "t:sections.hero.settings.heading.label" }
```

### `locales/en.default.json`
```json
{
  "sections": {
    "hero": {
      "name": "Hero banner",
      "settings": {
        "heading": { "label": "Heading" }
      }
    }
  }
}
```

### In Liquid
```liquid
{{ 'general.cart.empty' | t }}
{{ 'products.product.add_to_cart' | t }}
```

### Pluralisation
```json
"cart_count": { "one": "{{ count }} item", "other": "{{ count }} items" }
```
```liquid
{{ 'cart.cart_count' | t: count: cart.item_count }}
```

---

## 7. Section Groups (Header / Footer)

```liquid
{%- sections 'header-group' -%}
```

### `sections/groups/header-group.json`
```json
{
  "type": "header",
  "name": "Header",
  "sections": {
    "announcement-bar": { "type": "announcement-bar" },
    "header":           { "type": "header" }
  },
  "order": ["announcement-bar", "header"]
}
```

Mark sections that only belong in header/footer:
```json
"enabled_on":  { "groups": ["header"] }
"disabled_on": { "groups": ["header", "footer"] }
```

---

## 8. Cart Behaviour Patterns

### Drawer cart (recommended pattern)
```js
// After successful add-to-cart
document.dispatchEvent(new CustomEvent('cart:open'));

// Drawer listens:
document.addEventListener('cart:open', () => {
  drawer.setAttribute('aria-hidden', 'false');
  trapFocus(drawer);
});

// Close on Escape
document.addEventListener('keydown', (e) => {
  if (e.key === 'Escape') closeDrawer();
});
```

Focus management:
- Trap focus inside open drawer (`Tab` cycles within)
- Close on `Escape` and overlay click
- Restore focus to the "Add to cart" button that triggered it

### Cart error handling
```js
const res = await fetch('/cart/add.js', { method: 'POST', body: ... });
if (!res.ok) {
  const { description } = await res.json();
  showErrorMessage(description); // e.g. "Only 2 left in stock"
}
```

### Optimistic UI
1. Immediately increment cart bubble in header DOM
2. Fire add-to-cart fetch in background
3. On error: revert the count, show error toast
