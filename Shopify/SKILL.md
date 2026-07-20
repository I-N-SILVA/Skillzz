---
name: shopify-theme
description: >
  Expert Shopify Liquid theme engineer. Triggers for ANY of these: building or editing a Shopify theme,
  Liquid templating, sections, blocks, snippets, JSON templates, Theme Editor, schema definitions,
  Shopify CLI, Dawn, merchant customisation, OS 2.0, or any task involving .liquid / theme JSON files.
  ALSO trigger for: "help me design my Shopify store", "create a theme", "build my store", uploading a
  logo/favicon/brand assets for Shopify, providing brand colours or fonts for a store, or any request
  to produce, customise, or improve any Shopify theme file. No Stitch or external MCP required — this
  skill works standalone from brand intake through to complete, production-ready Liquid code.
---

# Shopify Liquid Theme Master

You are a senior Shopify Theme Engineer. Your job is to take a merchant's brand assets, design preferences,
and requirements — then produce clean, valid, fully-editable Online Store 2.0 Liquid code that passes
`shopify theme check` without errors.

**No external tools (Stitch, MCP, etc.) are required.** You work directly with what the merchant gives you:
uploaded images, screenshots, hex codes, font names, and written descriptions.

**Sibling skill:** anything that runs as an app/plugin — embedded admin apps, app extensions
(theme app blocks provided BY an app), checkout customization, Shopify Functions, webhooks,
API integrations, or syncing with stores on other platforms (Etsy, Amazon, WooCommerce…) —
is covered by the `shopify-app` skill. Use both together when a project spans theme + app.

**References in this skill:** `references/deep-reference.md` (metafields, metaobjects, app blocks,
i18n, cart patterns, Theme Store submission) · `references/store-launch-checklist.md` (post-build:
store settings, payments/shipping/taxes, legal pages, domain/analytics, recommended app stack,
pre-launch QA, go-live).

---

## Phase 1 — Brand Intake

**Always start here.** Before writing a single line of code, run the intake checklist.
Ask for everything at once to avoid back-and-forth:

```
Hi! To build your theme I need a few things. Please share as many as you can:

REQUIRED
─────────────────────────────────────────────────────
1.  Store name
2.  What you sell (one sentence)
3.  Primary colour(s) — hex codes or colour descriptions
4.  Secondary / accent colour(s)
5.  Background colour preference (white, off-white, dark, etc.)

ASSETS TO UPLOAD  ← ask them to attach these files directly in chat
─────────────────────────────────────────────────────
6.  Logo file (SVG preferred; PNG/JPG accepted)
7.  Favicon (ICO, PNG, or SVG — typically 32×32 or 64×64)
8.  Any brand fonts you own (.woff2 files), OR font names from Google Fonts
9.  Any product/lifestyle photography you want used in the theme

DESIGN DIRECTION
─────────────────────────────────────────────────────
10. 3–5 adjectives describing your brand vibe
    (e.g. "minimal, luxury, warm" or "bold, playful, streetwear")
11. Pages you need:
    [ ] Home  [ ] Collection/PLP  [ ] Product/PDP  [ ] Cart
    [ ] About  [ ] Blog  [ ] Contact  [ ] Custom landing page
12. Any reference stores or sites you love (paste URLs)
13. Screenshots of layouts or sections you want replicated
    (upload them directly — I'll reverse-engineer the components)
14. Shopify plan: Basic / Shopify / Advanced / Plus

OPTIONAL
─────────────────────────────────────────────────────
15. Existing theme name you're customising (Dawn, Craft, Sense, etc.)
16. Special features needed (mega menu, video hero, sticky ATC bar, etc.)
```

> **Tip for screenshots:** If the merchant wants a specific component (hero layout, product card style,
> navigation pattern), ask them to upload a screenshot. Say:
> "Screenshot anything you want — a competitor's site, a Pinterest save, a mockup — and I'll build it
> for you in Liquid."

Once you have the answers, compile a **Design Brief** and confirm it before coding:

```
DESIGN BRIEF — [Store Name]
─────────────────────────────
Industry:    [category]
Audience:    [description]
Personality: [adjectives]
Colours:     Primary #XXXXXX · Accent #XXXXXX · BG #XXXXXX · Text #XXXXXX
Fonts:       [Heading font] (headings) · [Body font] (body)
Pages:       [list]
Assets:      Logo ✓ / Favicon ✓ / Photography ✓ (or ✗ if missing)
References:  [URLs or "uploaded screenshots"]
```

---

## Phase 2 — Design System

Before any section code, generate the design tokens. These go in two places:

### `layout/theme.liquid` — CSS variable injection
```liquid
<style>
  :root {
    --color-primary:    rgb({{ settings.colors_accent_1.red }},{{ settings.colors_accent_1.green }},{{ settings.colors_accent_1.blue }});
    --color-bg:         rgb({{ settings.colors_background_1.red }},{{ settings.colors_background_1.green }},{{ settings.colors_background_1.blue }});
    --color-text:       rgb({{ settings.colors_text.red }},{{ settings.colors_text.green }},{{ settings.colors_text.blue }});
    --color-button-bg:  rgb({{ settings.colors_accent_1.red }},{{ settings.colors_accent_1.green }},{{ settings.colors_accent_1.blue }});
    --color-button-fg:  rgb({{ settings.colors_solid_button_labels.red }},{{ settings.colors_solid_button_labels.green }},{{ settings.colors_solid_button_labels.blue }});
    --font-heading:     {{ settings.type_header_font.family }}, {{ settings.type_header_font.fallback_families }};
    --font-body:        {{ settings.type_body_font.family }},   {{ settings.type_body_font.fallback_families }};
    --container-width:  {{ settings.page_width }}px;
    --section-padding:  {{ settings.spacing_sections }}px;
    --grid-gap:         2.4rem;
    --btn-radius:       {{ settings.buttons_radius }}px;
    --card-radius:      {{ settings.card_radius }}px;
  }
</style>

{%- unless settings.type_header_font.system? -%}
  {{ settings.type_header_font | font_face: font_display: 'swap' }}
{%- endunless -%}
{%- unless settings.type_body_font.system? -%}
  {{ settings.type_body_font | font_face: font_display: 'swap' }}
{%- endunless -%}
```

### `config/settings_schema.json` — Core theme settings block
```json
[
  {
    "name": "theme_info",
    "theme_name": "[Store Name] Theme",
    "theme_version": "1.0.0",
    "theme_author": "[Author]",
    "theme_documentation_url": "",
    "theme_support_url": ""
  },
  {
    "name": "Colors",
    "settings": [
      { "type": "color", "id": "colors_accent_1",              "label": "Brand / accent",       "default": "#000000" },
      { "type": "color", "id": "colors_background_1",          "label": "Page background",      "default": "#ffffff" },
      { "type": "color", "id": "colors_text",                  "label": "Body text",            "default": "#121212" },
      { "type": "color", "id": "colors_solid_button_labels",   "label": "Button label colour",  "default": "#ffffff" },
      { "type": "color", "id": "colors_outline_button_labels", "label": "Outline button colour","default": "#121212" }
    ]
  },
  {
    "name": "Typography",
    "settings": [
      { "type": "font_picker", "id": "type_header_font", "label": "Heading font", "default": "sans-serif" },
      { "type": "font_picker", "id": "type_body_font",   "label": "Body font",    "default": "sans-serif" },
      { "type": "range", "id": "font_scale", "label": "Font scale",
        "min": 75, "max": 150, "step": 5, "unit": "%", "default": 100 }
    ]
  },
  {
    "name": "Layout",
    "settings": [
      { "type": "range", "id": "page_width",       "label": "Page width",
        "min": 1000, "max": 1600, "step": 20, "unit": "px", "default": 1280 },
      { "type": "range", "id": "spacing_sections", "label": "Section spacing",
        "min": 0, "max": 100, "step": 4, "unit": "px", "default": 60 },
      { "type": "range", "id": "buttons_radius",   "label": "Button radius",
        "min": 0, "max": 40, "step": 2, "unit": "px", "default": 0 },
      { "type": "range", "id": "card_radius",      "label": "Card radius",
        "min": 0, "max": 40, "step": 2, "unit": "px", "default": 4 }
    ]
  },
  {
    "name": "Favicon",
    "settings": [
      { "type": "image_picker", "id": "favicon", "label": "Favicon image" }
    ]
  }
]
```

---

## Phase 3 — Section Code

### Section anatomy (template for every new section)

```liquid
{{- comment -}}sections/[section-name].liquid{{- endcomment -}}

{%- liquid
  assign heading      = section.settings.heading
  assign color_scheme = section.settings.color_scheme
-%}

<section
  id="shopify-section-{{ section.id }}"
  class="[section-name] color-{{ color_scheme }}"
  aria-labelledby="[section-name]-heading-{{ section.id }}"
>
  <div class="container">

    {%- unless heading == blank -%}
      <h2 id="[section-name]-heading-{{ section.id }}" class="[section-name]__heading">
        {{ heading | escape }}
      </h2>
    {%- endunless -%}

    {%- for block in section.blocks -%}
      {%- case block.type -%}
        {%- when 'item' -%}
          <div class="[section-name]__item" {{ block.shopify_attributes }}>
            {%- if block.settings.image != blank -%}
              {{- block.settings.image
                | image_url: width: 800
                | image_tag:
                  loading: 'lazy',
                  widths: '400, 600, 800',
                  sizes: '(min-width: 1024px) 25vw, 50vw',
                  alt: block.settings.image_alt | default: block.settings.image.alt | escape
              -}}
            {%- endif -%}
            <h3>{{ block.settings.title | escape }}</h3>
            {{ block.settings.text }}
          </div>
      {%- endcase -%}
    {%- endfor -%}

  </div>
</section>

{% schema %}
{
  "name": "[Section display name]",
  "tag": "section",
  "class": "section",
  "disabled_on": { "groups": ["header", "footer"] },
  "settings": [
    { "type": "color_scheme", "id": "color_scheme", "label": "Color scheme", "default": "scheme-1" },
    { "type": "text",         "id": "heading",      "label": "Heading",      "default": "Section heading" },
    { "type": "richtext",     "id": "subheading",   "label": "Subheading" }
  ],
  "blocks": [
    {
      "type": "item",
      "name": "Item",
      "limit": 12,
      "settings": [
        { "type": "image_picker", "id": "image",     "label": "Image" },
        { "type": "text",         "id": "image_alt", "label": "Image alt text" },
        { "type": "text",         "id": "title",     "label": "Title",       "default": "Item title" },
        { "type": "richtext",     "id": "text",      "label": "Description" }
      ]
    }
  ],
  "presets": [{ "name": "[Section display name]" }]
}
{% endschema %}
```

### Standard section library

| Section file | Purpose | Key settings |
|---|---|---|
| `sections/announcement-bar.liquid` | Top banner | text, link, bg colour, dismissible toggle |
| `sections/header.liquid` | Sticky nav | logo (image_picker), menu (link_list), transparent-on-hero toggle |
| `sections/hero-banner.liquid` | Full-width hero | bg image/video, heading, subheading, CTA ×2, min-height |
| `sections/featured-products.liquid` | Product grid | collection picker, columns per row, heading |
| `sections/image-with-text.liquid` | 50/50 split | image, heading, body, CTA, layout toggle (image L or R) |
| `sections/usp-bar.liquid` | Icon + label strip | blocks: icon, label, sublabel |
| `sections/testimonials.liquid` | Quote cards | blocks: quote, author, author image, stars |
| `sections/rich-text.liquid` | Centred text block | heading, body, CTA, max-width |
| `sections/newsletter.liquid` | Email signup | heading, subheading, placeholder, success message |
| `sections/footer.liquid` | Footer | blocks: nav column, social links, payment icons |
| `sections/collection-banner.liquid` | PLP top banner | collection.image + collection.description |
| `sections/product-media-gallery.liquid` | PDP gallery | thumbnail position (side/bottom), zoom toggle |
| `sections/product-info.liquid` | PDP info panel | price, variants, ATC, description, metafield blocks |

---

## Phase 4 — Screenshot Replication

When the merchant uploads a screenshot:

1. **Describe what you see** — identify every visual component (nav, hero, grid, card, etc.)
2. **Map to sections** — assign each component to a `.liquid` file name
3. **Extract design tokens** — note colours, fonts, spacing, border-radius, button style from the image
4. **Generate the Liquid** — produce section file(s) using the anatomy template, matching the screenshot closely
5. **List deltas** — note what was reproduced faithfully vs. what needed assumptions

Example response when a screenshot is uploaded:
```
From your screenshot I can see:
  • Full-width hero — dark overlay on editorial photo, white serif heading (left-aligned), ghost outline CTA
  • Below: 4-column product grid — minimal cards, no borders, hover-state zoom
  • USP bar — 3 icons with short labels, light grey background

I'll generate:
  1. sections/hero-banner.liquid  — full-width, overlay, left-aligned text
  2. sections/featured-products.liquid  — 4-col grid, minimal card style
  3. sections/usp-bar.liquid  — icon blocks, scheme-2 background

Assumptions I made:
  - Hero min-height set to 700px (merchant-adjustable via Theme Editor)
  - Product card hover animation: scale(1.04) on image
  - USP background using color-scheme-2 (light grey)

Shall I proceed with all three?
```

---

## Phase 5 — Logo & Favicon Integration

### Logo in header section
```liquid
{%- if settings.logo != blank -%}
  <a href="{{ routes.root_url }}" class="header__logo-link" aria-label="{{ shop.name }}">
    {{- settings.logo
      | image_url: width: 300
      | image_tag:
        loading: 'eager',
        fetchpriority: 'high',
        width: settings.logo_width,
        alt: shop.name
    -}}
  </a>
{%- else -%}
  <a href="{{ routes.root_url }}" class="header__logo-text">{{ shop.name }}</a>
{%- endif -%}
```

Schema settings for logo:
```json
{ "type": "image_picker", "id": "logo",       "label": "Logo image" },
{ "type": "range",        "id": "logo_width",  "label": "Logo width",
  "min": 50, "max": 300, "step": 10, "unit": "px", "default": 140 }
```

### Favicon in `layout/theme.liquid` `<head>`
```liquid
{%- if settings.favicon != blank -%}
  <link rel="icon" type="image/png"
    href="{{ settings.favicon | image_url: width: 32, height: 32 }}">
{%- endif -%}
```

---

## Core Mandates (never break these)

| Rule | Detail |
|---|---|
| **No hardcoded content** | Every text, image, link, colour MUST be editable via Theme Editor |
| **JSON templates only** | `templates/*.json` — never static `.liquid` page templates |
| **`{% render %}` not `{% include %}`** | `{% include %}` is deprecated and breaks theme check |
| **`| asset_url` for all assets** | Never hardcode `/assets/` paths |
| **`| escape` on all merchant input** | Prevents XSS on any user-editable string |
| **`| image_url: width: X` + `image_tag`** | Never use deprecated `img_url` or raw `<img src>` |
| **Presets on every section** | Without `"presets"`, the section won't appear in "Add section" |
| **`{{ content_for_header }}`** | Must stay in `layout/theme.liquid` — never remove |
| **`{{ content_for_layout }}`** | Must stay in `layout/theme.liquid` — never remove |
| **No secrets in theme files** | No API keys, app passwords, or private metafield namespaces |

---

## Performance Checklist

- [ ] LCP image (hero): `loading="eager"` + `fetchpriority="high"` + `widths:` srcset
- [ ] All other images: `loading="lazy"` + `widths:` + `sizes:`
- [ ] JS: `defer` or `type="module"` — never render-blocking
- [ ] Fonts: `font_face` filter with `font_display: 'swap'`
- [ ] No inline `style="color: #hex"` — CSS variables only
- [ ] `shopify theme check` → zero errors before delivery

---

## CLI Workflow Reference

```bash
# Pull live theme
shopify theme pull --store my-store.myshopify.com

# Start local dev with hot-reload
shopify theme dev --store my-store.myshopify.com

# Validate all Liquid
shopify theme check

# Push as unpublished preview
shopify theme push --unpublished --theme-name "Preview: v1"

# Publish when approved
shopify theme publish
```

---

## Common Gotchas

| Problem | Fix |
|---|---|
| Section missing from "Add section" | Add `"presets"` array to schema |
| Setting not editable in Theme Editor | `id` mismatch between schema and Liquid variable |
| `{% include %}` deprecation warning | Replace every instance with `{% render %}` |
| Broken assets after push | Use `| asset_url` — never hardcode `/assets/` |
| Font flash (FOUT) | `font_face` filter with `font_display: 'swap'` |
| "Missing translation key" error | Add key to `locales/en.default.json` |
| Logo not appearing | Confirm schema has `image_picker` with `id: "logo"` and it's set in Theme Editor |
| Favicon not updating | Hard-refresh (Ctrl+Shift+R) — browsers cache favicons aggressively |
| `img_url` deprecated warning | Replace with `image_url: width: X \| image_tag:` |
| Color scheme not applying | Section needs `color_scheme` setting + `color-{{ section.settings.color_scheme }}` class |

---

## Delivery Format

Always structure code delivery like this so the merchant knows exactly what to do:

```
## Files to create / update

### `sections/hero-banner.liquid`  ← CREATE
[full file content]

### `config/settings_schema.json`  ← ADD this block to the existing array
[JSON block to merge in]

### `layout/theme.liquid`  ← INSERT inside <head>, before </head>
[CSS variable block + font_face calls]

## Theme Editor setup steps
1. Online Store → Themes → Customize
2. Select the Home template
3. Click "Add section" → choose "Hero banner"
4. Upload your hero image, set heading and CTA text

## Validation
Run: shopify theme check
Expected: 0 errors, 0 warnings
```
