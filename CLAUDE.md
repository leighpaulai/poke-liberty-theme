# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What this is

A Shopify **Horizon** theme (v4.1.4) for Poke Liberty, a Pokémon TCG store. Pure Liquid/CSS/JS — no bundler, no package.json build step, no test framework. The only tooling is the Shopify CLI.

## Commands

```bash
shopify theme check                                    # lint Liquid + schema (run before every push)
shopify theme dev --store <store-domain>                # local dev server with live reload
shopify theme push --theme=<numeric-id>                 # push to an EXISTING theme (published or unpublished) by ID
shopify theme pull --theme=<numeric-id>                 # pull an existing theme's current state down
shopify theme list                                       # list themes with their numeric IDs and roles (live/unpublished)
```

**Push gotcha:** `--unpublished --theme=<value>` does **not** target an existing theme by ID — Shopify CLI treats it as "create a new unpublished theme with this **name**." To update an existing unpublished theme, push with `--theme=<id>` and omit `--unpublished`. Using the wrong form silently creates a duplicate theme instead of erroring.

**Store domain gotcha:** this store's vanity domain (`poke-liberty.myshopify.com`) is not its permanent `*.myshopify.com` domain. `shopify store auth` needs the permanent one — run it once and the CLI will report the correct domain in the error if you get it wrong.

**Store-scoped Admin API access** (e.g. reading/creating collections, products) goes through the CLI, not a separate API client:
```bash
shopify store auth --store <permanent-domain>.myshopify.com --scopes <scopes>
shopify store execute --store <permanent-domain>.myshopify.com --query '...'   # add --allow-mutations for writes
```

No linter/formatter config (no ESLint/Stylelint/Prettier), no `.gitignore`, no test suite — `shopify theme check` is the only automated check.

## Deployment workflow used in this repo

Live theme changes are never made directly. Work happens on a git branch (currently `brand-refresh-ink-gold`), gets pushed to a dedicated **unpublished** theme in the store ("Poke Liberty — Ink & Gold"), and is reviewed via that theme's preview URL before anyone publishes it. Keep following this pattern unless told otherwise: branch → edit → `shopify theme check` → `shopify theme push --theme=<unpublished-theme-id>` → share the preview link.

## Architecture

### Section groups, sections, blocks — the composition model
`layout/theme.liquid` renders two **section groups** (`sections/header-group.json`, `sections/footer-group.json`) plus `content_for_layout`, which resolves to whichever **template** JSON (`templates/*.json`) matches the current page. Every template/group JSON is a tree of section instances → block instances, each pointing at a `sections/*.liquid` or `blocks/*.liquid` file by `"type"` and carrying that instance's `"settings"`.

Blocks and sections prefixed with `_` (e.g. `blocks/_product-card.liquid`, `blocks/_card.liquid`) are **internal/static** building blocks — referenced from other schemas or marked `"static": true` in template JSON, not directly addable by a merchant in the theme editor. Unprefixed blocks (`blocks/button.liquid`, `blocks/text.liquid`, `blocks/collection-card.liquid`, …) are the public, merchant-addable ones. Snippets (`snippets/*.liquid`) hold reusable render logic and are `{% render %}`'d from sections/blocks; they don't have their own schema.

### Design tokens cascade from one palette
Nearly every color setting in `config/settings_data.json` and in section/template JSON is bound to `settings.color_palette.background/.foreground/.color1/.color2` via Liquid strings baked directly into the JSON (e.g. `"background_color": "{{ settings.color_palette.background }}"`). Changing the four hex values in `color_palette` re-themes buttons, inputs, popovers, drawers, badges, variant pickers, cart, and footer at once. A few settings hold literal hex instead of a palette reference (e.g. badge colors) — those need updating by hand when re-theming and won't follow the palette automatically. `snippets/color-palette.liquid` is what turns the palette settings into the actual `--color-*` CSS custom properties consumed everywhere else; `snippets/theme-styles-variables.liquid` builds the rest of the token system (spacing, type scale, animation, z-index layers).

### `{% style %}` vs `{% stylesheet %}`
Liquid files use both tags and they behave differently: `{% style %}` is evaluated per-render and can reference `settings`/`section`/`block` values dynamically (used for anything settings-driven, e.g. `snippets/color-palette.liquid`, `snippets/fonts.liquid`). `{% stylesheet %}` is deduped/cached static CSS with no per-instance Liquid interpolation — used for a component's fixed layout rules (e.g. `snippets/product-card-styles.liquid`). Follow the existing split when adding styles: settings-driven values go in `{% style %}`, static component CSS goes in `{% stylesheet %}`.

### Global cross-cutting styles must be rendered from the layout
A snippet's `{% stylesheet %}`/`{% style %}` block is only collected into the page when that snippet actually renders somewhere. For CSS that needs to apply to a surface appearing across many different sections/templates (e.g. product cards showing up in collection grids, homepage featured lists, and predictive search), the snippet is rendered once, unconditionally, from `layout/theme.liquid` — see `snippets/card-hover-effect-styles.liquid` and `snippets/holographic-card-effects.liquid`, both rendered there rather than from the sections that happen to use product cards.

### Custom elements + declarative events
Interactive UI is built as custom elements (`<header-component>`, `<product-card>`, `<dropdown-localization-component>`, …) backed by ES modules in `assets/*.js`, loaded natively via `<script type="module">` (no bundler). Markup wires events declaratively with `ref="name"` and `on:click="/methodName"` attributes rather than manual `querySelector`/`addEventListener` in most cases. `assets/jsconfig.json` defines a `@theme/*` path alias resolving to `assets/*` (e.g. `import { hydrate } from '@theme/section-hydration'`).

### Product taxonomy
Products are categorized by `product_type` (e.g. `"Sealed Product"`, `"Singles"`, `"Grading-Ready Pulls"`), which backs automated (rule-based) collections — this is the pattern to extend when adding new product categories (e.g. future TCG supplies) rather than manual/hand-curated collections. The homepage surfaces these via `blocks/category-tile.liquid` (a self-contained, image-free tile — icon panel + heading + description + collection link) rather than Shopify's built-in `collection-card`/`_collection-card-image`, since those depend on real collection photography that doesn't exist yet.

### JSON template block IDs must be unique per section, including nested ones
Block IDs (the JSON object keys under a section's/block's `"blocks"`) must be unique across the **entire** section, not just unique among siblings — that includes static sub-blocks nested inside other blocks. Reusing the same ID in two different blocks (e.g. two `collection-card` blocks that each nest a static `_collection-card-image` block under the identical key) causes Shopify to conflate which block's content belongs to which instance, and content silently fails to render for the colliding ones — no error, just a blank result. Give every block instance in a template/section JSON file, at every nesting level, its own unique key.

### Locales
`locales/*.json` are storefront-facing translation strings; `locales/*.schema.json` are the parallel translations for section/block schema labels (`t:` references in `{% schema %}` blocks resolve against these, not the non-`.schema` files — a schema `"label"`/`"name"` field can also just be a literal string if there's no matching `.schema.json` entry). `en.default.json` / `en.default.schema.json` are the source-of-truth English files.

### Test / dummy data
The store's product catalog is currently mostly placeholder: ~24 fake products (fictional names, no real product photography — flat on-brand SVG/PNG icon placeholders instead) were created via the Admin API to exercise collection/grid layouts before real inventory exists. Every one of them is tagged **`test-data`**, split evenly across `product_type` "Sealed Product" / "Singles" / "Grading-Ready Pulls". To bulk-remove them once real inventory arrives: filter the Shopify admin product list by `tag:test-data` (or query `products(query: "tag:test-data")` via the Admin API) and delete in bulk. The 3 real collections (`sealed-product`, `singles`, `grading-ready-pulls`) and the `blocks/category-tile.liquid` homepage tiles are not test data — they stay.
