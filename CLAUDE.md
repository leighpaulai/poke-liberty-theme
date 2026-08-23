# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What this is

A Shopify **Horizon** theme (v4.1.4) for Liberty TCG (formerly "Poke Liberty" — renamed Aug 2026, same legal entity Liberty TCG LLC), a Pokémon TCG store. Pure Liquid/CSS/JS — no bundler, no package.json build step, no test framework. The only tooling is the Shopify CLI.

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

**Current live theme is "Liberty TCG — Ink & Gold" (#189088694568)** — published Aug 22, 2026, ending this repo's long unpublished-theme phase. `git log` on `brand-refresh-ink-gold` still tells that whole story (brand refresh → homepage restructure → rebrand → publish).

Going forward, keep the same discipline that got it here safely: don't push to the live theme without explicit sign-off in the moment, even though the CLI will now happily do it (`shopify theme push --theme=189088694568 --allow-live` — the `--allow-live` flag is required specifically because it's live, and is itself a deliberate confirmation gate worth respecting each time, not just the first time). For any new work, prefer creating/reusing an unpublished theme, iterating there, and only pushing to live once it's reviewed. There's already a leftover unpublished theme in the store, "Copy of Poke Liberty — Ink & Gold" (#189104881960) — Shopify auto-created it around the time of publishing (likely a backup of what got replaced); nobody's confirmed whether it's safe to delete or worth keeping, so it's just sitting there for now.

Workflow shape, updated for the live-theme era: branch → edit → `shopify theme check` → push to whichever theme you're actually targeting (unpublished for review, live only with explicit go-ahead) → confirm on the actual URL, not just the diff.

## Rebrand status: Poke Liberty → Liberty TCG (Aug 2026) — COMPLETE

Same legal entity (Liberty TCG LLC) and same visual system (navy/gold/parchment palette, Fraunces/Space Grotesk, holo-card + lightning-bolt motif) — this was a name change, not a redesign. All items done and live as of the Aug 22 publish:

- **Text sweep** (`d228c4e`): homepage heading + this doc's own prose, "Poke Liberty" → "Liberty TCG". No `hello@pokeliberty.com` references anywhere in theme files.
- **Header logo** (`f5a56fe`): `LibertyTCG_Logo_Full_Transparent.png`, uploaded via Admin API staged-upload, `settings.logo` repointed. The homepage hero background image intentionally stayed `PokeLiberty_Hero_Banner.png` (a card graphic, not brand text — no conflict, per direct request).
- **Public domain**: `libertytcgshop.com` is the store's primary domain (verified via `shop.primaryDomain`); `poke-liberty.myshopify.com` 301-redirects to it. Note: the storefront currently has **password protection enabled** (Preferences) — separate, pre-existing setting, unrelated to the rebrand or theme publish; nobody's touched it.
- **Favicon**: `LibertyTCG_Favicon_32.png`, uploaded via Admin API, `settings.favicon` set. Confirmed rendering in `<head>` on the live site.
- **Social sharing image**: `LibertyTCG_Social_Share_1200x628.png`, set via Admin → Online Store → Preferences (no Admin API field exists for this — confirmed by schema introspection, so it has to go through that screen). Confirmed live — `og:image` resolves to it.
- **Shop's registered name**: `shop.name` is now "Liberty TCG" (also set via Admin UI — no `shopUpdate`-style mutation exists in this API for it either). Confirmed via Admin API query.
- **Theme display name**: renamed via `shopify theme rename` from "Poke Liberty — Ink & Gold" to "Liberty TCG — Ink & Gold", before it became the live theme.

**One unconfirmed leftover**: homepage meta description. Attempted via `metafieldsSet` on `global.description_tag` (the documented mechanism for this) but it's never been observed actually rendering on the live page — `page_description` still empty as of last check. Might just be more of the same storefront-indexing lag seen everywhere else this session (see below), might be the wrong mechanism entirely. Set it directly in Admin → Online Store → Preferences → homepage meta description if it's still missing by the time anyone checks again — don't trust the metafield write alone.

**Deliberately left as-is, not an oversight:** `poke-liberty.myshopify.com` (the store's actual Shopify infra subdomain — a separate, bigger action from a branding text change, not requested).

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

### Collections created via the Admin API aren't published to any sales channel by default
`collectionCreate` (Admin GraphQL) creates the collection but does **not** publish it anywhere — unlike creating one through the Admin UI, which auto-publishes to Online Store. An unpublished collection still resolves fine via direct Admin API lookups (`collectionByHandle`, etc.), still accepts a `"type": "collection"` block-schema reference by **plain handle string** (that part was never the problem), but is invisible in the Theme Editor's collection picker and any storefront/Liquid resolution of it comes back blank — a tile/link pointing at it silently falls back to its "no collection selected" state instead of erroring. Fix with `publishablePublish(id: <collection gid>, input: [{ publicationId: <channel gid> }, ...])` for each sales channel (list them via `publications(first: 10) { nodes { id name } }`). Any collection created via the Admin API in this repo's workflow needs this as a follow-up step.

### Collection filters (facets) — theme is zero-code, but enabling a new filter isn't via API
`templates/collection.json`'s `main-collection` section already has a `filters` block (`enable_filtering: true`) that dynamically renders whatever filters are turned on for that collection — no theme code needed to add a new filterable property. Confirmed live: the rendered page uses the `facets`/`facets__*` classes (from `blocks/filters.liquid` + `snippets/list-filter.liquid`).

**Product `series` (Series/Set) metafield**: `custom.series`, type `single_line_text_field`, restricted to a fixed `choices` validation list (currently ~40 Pokémon TCG set names — extend by editing this definition's validation in Admin → Settings → Custom data → Products → "Series / Set", or via `metafieldDefinitionUpdate` with a new `choices` validation value covering the old list plus additions). Created via `metafieldDefinitionCreate`; confirmed this part works fine over the Admin API.

**Turning a metafield into an actual storefront filter does not, however, go through the Admin GraphQL API** — searched the full schema for anything filter-related (`FilterOption`, `MetafieldCapability*`, etc.) and found only `adminFilterable` (admin product-list filtering, not storefront) and `smartCollectionCondition` (automated-collection rules, not filters). Enabling a filter is a **Search & Discovery app** setting (Admin → Apps → Search and discovery → Filters) with no exposed GraphQL mutation — Paul enabled it there.

**A second, separate blocker found after that**: a metafield definition's `access.storefront` defaults to `NONE` when created via `metafieldDefinitionCreate` without explicitly setting it — meaning the metafield is invisible to the storefront/theme layer entirely, independent of whatever Search & Discovery's filter toggle says. Confirmed via `metafieldDefinition(id: ...) { access { storefront } }` — came back `"NONE"`. This alone would keep a filter from ever appearing no matter how correctly the Search & Discovery side is configured. Fixed with `metafieldDefinitionUpdate(definition: { namespace: "custom", key: "series", ownerType: PRODUCT, access: { storefront: PUBLIC_READ } })`. **Any future custom metafield meant to be customer-facing (filters, or referenced directly in theme Liquid) needs `access: { storefront: PUBLIC_READ }` set explicitly at creation** — it is not the default.

**Third factor, the one that actually mattered most**: `collection.products` (unpaginated) only reflects a bounded window of the collection's current membership — as real Singles inventory grew, the original 5 test-data products I'd tagged fell outside whatever window the storefront was rendering, so Search & Discovery had zero visible products carrying a `custom.series` value to build a filter option from, independent of the storefront-access fix. Confirmed via `shopify theme console --url /collections/singles` + `collection.products | map: 'title'`, which showed a completely different product set than the Admin API's full membership list. Resolved by setting `custom.series` on 8 of the *currently in-view* real products instead (Zamazenta V, Simisear V, Leafeon V ×2, Morpeko ex ×2, Ambipom — spanning Celebrations/Crown Zenith/Pitch Black/Phantasmal Flames) — filter appeared immediately after, no further lag. **Lesson: when a storefront-facing feature depends on specific products carrying a value, verify those exact products are within the collection's currently-rendered window** (`shopify theme console` + `collection.products | map: 'title'`) rather than assuming Admin-API collection membership is what the storefront is currently serving.

Fully verified end-to-end via `shopify theme console`: `collection.filters | map: "label"` → `["Availability", "Price", "Series / Set"]`; the filter's own value list → `["Celebrations", "Crown Zenith", "Phantasmal Flames", "Pitch Black"]`. Choices list is now ~52 values (original 40 + Chaos Rising, Perfect Order, Ascended Heroes, Phantasmal Flames, Mega Evolution, Black Bolt, White Flare, Destined Rivals, Journey Together, Prismatic Evolutions, Scarlet & Violet Energies, Pitch Black — added once real inventory revealed sets the starter list didn't cover; "Pitch Black" doesn't have an officially finalized set code yet as of this writing but is already this store's real inventory convention). `CLB` = Celebrations — confirmed directly by Paul (wasn't in his own set-code reference table, which explicitly excludes older/special sets, but he confirmed the mapping is correct).

**CSV import column header** for the bulk product import: Shopify's standard, store-independent format for a metafield column is `Product Metafield: {namespace}.{key} [{type}]` — for this one, `Product Metafield: custom.series [single_line_text_field]`. This is a documented Shopify convention, not something that varies per store, but the surest way to see the literal header Shopify will produce for this exact setup is Admin → Products → Export a product that already has `custom.series` set (e.g. one of the 5 tagged test products) and read the header row it generates.

### Custom elements + declarative events
Interactive UI is built as custom elements (`<header-component>`, `<product-card>`, `<dropdown-localization-component>`, …) backed by ES modules in `assets/*.js`, loaded natively via `<script type="module">` (no bundler). Markup wires events declaratively with `ref="name"` and `on:click="/methodName"` attributes rather than manual `querySelector`/`addEventListener` in most cases. `assets/jsconfig.json` defines a `@theme/*` path alias resolving to `assets/*` (e.g. `import { hydrate } from '@theme/section-hydration'`).

### Product card: stock status + always-visible Add to Cart
`snippets/product-card.liquid` is the single shared card component (collection grids, search, homepage, "featured product" — everywhere a product grid renders), so both of these apply everywhere automatically, no per-template edits needed:

- **Stock status line** (`snippets/product-card-stock-status.liquid`, rendered unconditionally from `product-card.liquid`): "In Stock" / "Only [X] left" / "Sold out", read from `product.selected_or_first_available_variant.inventory_quantity`. **To change the "low stock" threshold** (currently 9): Theme Editor → Theme settings → Product cards → "Low stock threshold" — it's `settings.low_stock_threshold` in `config/settings_schema.json`, a real merchant-facing setting, not hardcoded — no code push needed to adjust it. Untracked variants (inventory not managed by Shopify) or overselling-enabled variants at 0 quantity fall through to "In Stock" rather than showing a misleading count.
- **Add to Cart** used to be Shopify Horizon's stock quick-add: a hover-only, mobile-hidden floating pill overlaid on the product image (`blocks/_product-card-gallery.liquid` + `snippets/quick-add.liquid`/`quick-add-styles.liquid`). It's now relocated into normal card flow as an always-visible, full-width `.button`-styled button below the image/title/price (`.product-card__buy-row` in `product-card.liquid`, scoped CSS overrides so other quick-add usages elsewhere — e.g. product hotspots — keep their original floating-pill behavior). `mobile_quick_add` is now on (`config/settings_data.json`) so it also renders on touch devices. Disabled with "Sold out" text (not just hidden) when the variant is unavailable — see the `add_to_cart_text` override in `quick-add.liquid`. Multi-option products (e.g. condition/language) still use the theme's existing "Choose options" → variant-picker-modal flow unchanged.

### Product taxonomy
Products are categorized by `product_type` (e.g. `"Sealed Product"`, `"Singles"`, `"Grading-Ready Pulls"`), which backs automated (rule-based) collections — this is the pattern to extend when adding new product categories (e.g. future TCG supplies) rather than manual/hand-curated collections. The homepage surfaces these via `blocks/category-tile.liquid` (heading + description + a panel linking to the collection) rather than Shopify's built-in `collection-card`/`_collection-card-image`, which depends on collection-level photography this store doesn't use.

### Category tile photo rotation
`blocks/category-tile.liquid`'s panel crossfades through up to 6 of the linked collection's product images (`block.settings.rotation_speed`, a Theme Editor range setting, controls seconds/photo) — falls back to a static on-brand icon when a collection has no product photos yet (e.g. an empty/new category). Two non-obvious things learned building this, both worth knowing before touching this file or writing similar Liquid elsewhere in the theme:

- **`collection.products` doesn't reliably support generic array filters** (`slice`, `map`, etc.) outside a `paginate` block in this environment — silently returns nothing at runtime with zero warning from `shopify theme check` or a rendered Liquid error, just an empty result. The proven pattern used everywhere else in this codebase (grep `collection.products` across `sections/`/`snippets/` for examples) is always a plain `for product in collection.products limit: N` loop. Use that, not filter chains, whenever pulling a bounded slice of a collection's products.
- **A code comment inside a `{% liquid %}` block that contains literal Liquid tag syntax (e.g. writing `{% paginate %}` inside a `#`-comment explaining why you're *not* using it) can corrupt parsing of the rest of that block** — again with no error anywhere, variables after the corrupted point just silently never get assigned. Cost real debugging time to track down (had to add a temporary HTML comment rendering intermediate variable values directly on the live page to see it, since theme check and the Liquid error surface both stayed silent). Avoid literal `{%`/`%}`-looking text inside `{% liquid %}` comments — describe tags in prose instead.

### Homepage structure (`templates/index.json`)
Three sections in order: `hero_lbty01` (heading + "We're collectors first" text + "Shop now" button, background image `PokeLiberty_Hero_Banner.png`) → `category_tiles_lbty` (the 3 `category-tile` blocks, its own section so the hero's background image isn't stretched across their height) → `product_list_fa6P9H` ("Showcase" carousel, see below).

### Showcase carousel
`product_list_fa6P9H` is pointed at the `showcase` collection (automated: tag `showcase`, created via Admin API — see the publishing gotcha above, already published), `layout_type: carousel`, 4 columns, `autoplay: true` / `autoplay_speed: 30` (added to `sections/product-list.liquid`'s schema — off by default everywhere else, see git history for why: the underlying `slideshow-component` already supported autoplay, it just wasn't threaded through `resource-list-carousel.liquid` until this). **To change which products show in the Showcase carousel:** add or remove the `showcase` tag on any product in Shopify Admin — no theme edit needed. A product keeps showing under its real category collection/page either way; the tag only controls Showcase membership.

### JSON template block IDs must be unique per section, including nested ones
Block IDs (the JSON object keys under a section's/block's `"blocks"`) must be unique across the **entire** section, not just unique among siblings — that includes static sub-blocks nested inside other blocks. Reusing the same ID in two different blocks (e.g. two `collection-card` blocks that each nest a static `_collection-card-image` block under the identical key) causes Shopify to conflate which block's content belongs to which instance, and content silently fails to render for the colliding ones — no error, just a blank result. Give every block instance in a template/section JSON file, at every nesting level, its own unique key.

### Locales
`locales/*.json` are storefront-facing translation strings; `locales/*.schema.json` are the parallel translations for section/block schema labels (`t:` references in `{% schema %}` blocks resolve against these, not the non-`.schema` files — a schema `"label"`/`"name"` field can also just be a literal string if there's no matching `.schema.json` entry). `en.default.json` / `en.default.schema.json` are the source-of-truth English files.

### Test / dummy data
The store's product catalog is currently mostly placeholder: ~24 fake products (fictional names, no real product photography — flat on-brand SVG/PNG icon placeholders instead) were created via the Admin API to exercise collection/grid layouts before real inventory exists. Every one of them is tagged **`test-data`**, split evenly across `product_type` "Sealed Product" / "Singles" / "Grading-Ready Pulls". To bulk-remove them once real inventory arrives: filter the Shopify admin product list by `tag:test-data` (or query `products(query: "tag:test-data")` via the Admin API) and delete in bulk. The 3 real collections (`sealed-product`, `singles`, `grading-ready-pulls`) and the `blocks/category-tile.liquid` homepage tiles are not test data — they stay.

**Depends on the test data:** 6 of those 24 are additionally tagged `showcase` and feed the homepage's "Showcase carousel" section (`product_list_fa6P9H`, `collection: "showcase"`) — an automated collection ruled on tag `showcase`. Deleting the test-data products will empty that carousel; tag real products `showcase` (any count) to replace them, or the section will just render with nothing in it.

**Collections created via the Admin API need an explicit publish step** — see the gotcha further down (`publishablePublish`) — this will matter again once real inventory/collections get created through the API rather than the Admin UI.
