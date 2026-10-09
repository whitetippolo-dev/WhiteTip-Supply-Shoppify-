# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What this repo is

A Shopify theme for **WhiteTip Polo Supply** (an Australian canoe polo gear retailer), built on **Shopify Dawn** (not the newer block-first Skeleton Theme — an earlier version of this repo mistakenly contained Skeleton Theme and was corrected by replacing it with a real clone of `Shopify/dawn`). All WhiteTip-specific work lives on top of commit `8705c20` ("Replace Skeleton Theme with official Shopify Dawn as the real base"); everything from that commit forward is custom. Diff against it (`git diff 8705c20 dev`) to see exactly what's WhiteTip vs stock Dawn.

Nothing in this repo talks to a live store by itself — it's theme source only. Store-side setup (navigation menus, Pages, metafield/metaobject definitions) has to happen in Shopify Admin and is tracked separately (see "Admin-side dependencies" below).

## Commands

```bash
# Validate theme code — the standard gate after every change
shopify theme check --path .

# Connect to a real store, live-reload preview (opens browser OAuth)
shopify theme dev --store=your-store.myshopify.com

# Upload as a new unpublished theme (no live sync)
shopify theme push --store=your-store.myshopify.com --unpublished

# First-time CLI auth
shopify auth login
```

There is no JS/CSS build step, no package.json, and no test suite — Dawn is deliberately dependency-free (vanilla JS custom elements, native CSS, server-rendered Liquid). `shopify theme check` is the only automated correctness gate in this repo, and CI runs it via `shopify/theme-check-action` on every push (`.github/workflows/ci.yml`), alongside a Lighthouse CI job against a configured store.

**Baseline to maintain:** 9 pre-existing Dawn warnings, 0 errors. Any new `theme check` offense introduced by a change should be fixed before committing — that was the standard held throughout this theme's development and should continue.

`.theme-check.yml` disables two rules repo-wide: `MatchingTranslations` and `TemplateLength`.

## Branches

- `main` / `dev` on `origin` — do not force-push either.
- `dawn-base` — local-only branch, was the working branch for the WhiteTip build before merging into `dev`. `dev` is currently a fast-forward of it.
- Never push without the user explicitly asking — this has been a standing constraint throughout this project's history.

## Architecture

### Dawn's section/template model (unmodified foundations)
- `sections/*.liquid` — reusable, schema-driven page sections (JSON `{% schema %}` block defines settings/blocks/presets editable in the Theme Editor).
- `snippets/*.liquid` — non-schema partials, included via `{% render 'name', ... %}`. Liquid disallows filter chains inline inside `{% render %}` arguments or `{% for x in Y %}` loop expressions — precompute with `{% assign %}` first or you'll hit `LiquidSyntaxError`/`UnsupportedFilterArguments`.
- `templates/*.json` — assign sections (with settings/block overrides) to a route. `page.<handle>.json` auto-applies to a Shopify Page with that exact handle (e.g. `page.contact.json` → Page handle `contact`).
- `config/settings_schema.json` (definitions) + `config/settings_data.json` (current values) — global theme settings and color schemes.
- `locales/en.default.json` — translation strings, referenced via `{{ 'key.path' | t }}`.
- `{% style %}` is Liquid-evaluated (can read `section.settings.*`); `{% stylesheet %}` is CSS scoped per-file — reusing a class across two files' `{% stylesheet %}` blocks trips theme-check's `ValidScopedCSSClass`. When a component's styles are genuinely shared by two sections (e.g. the pre-order note in both `main-cart-items.liquid` and `cart-drawer.liquid`), duplicate the small rule block in each file rather than hoisting to a global stylesheet — that's the pattern already in use.

### WhiteTip brand layer
- `assets/whitetip-tokens.css` — the single source of truth for brand tokens (colors, spacing scale, type scale, button/input/focus tokens, effects), linked in `layout/theme.liquid` right after `base.css`. Layers on top of, rather than replacing, Dawn's own `--color-*`/`--font-*` custom properties.
- Dawn's `scheme-1` is locked to the Ocean palette and `scheme-2` to Ocean-900 (`config/settings_data.json`) — use these two existing schemes for new sections needing brand colors rather than inventing new ones or hardcoding hex values outside the token file.
- Fonts use Dawn's native Google-Fonts picker (`settings.type_header_font`/`type_body_font`, both set to Montserrat) rather than self-hosted woff2 — keeps Theme Editor font controls live. Don't reintroduce self-hosted fonts without a specific reason.

### Centralized stock/pre-order logic
Pre-order detection is a single rule applied everywhere, never re-derived per caller: `available == true AND inventory_management == 'shopify' (tracked) AND inventory_quantity <= 0 AND custom.lead_time_weeks present`.

The three-state stock badge (in-stock / low-stock "Only N left" / pre-order "Pre-order · N wks") is centralized in `snippets/wt-card-badge.liquid`, which takes raw facts (`available`, `inventory_tracked`, `inventory_quantity`, `lead_time_weeks`, `low_stock_threshold`, `id`, `sold_out_color_scheme`) and computes the state internally. Callers (`card-product.liquid`, `main-product.liquid`, `main-cart-items.liquid`, `cart-drawer.liquid`) must stay thin and pass only facts — badge-state branching was deliberately moved out of `card-product.liquid` into this snippet after it tripped theme-check's `LiquidComplexity` rule (120-line cap). Keep new call sites to the same pattern instead of reintroducing inline branching.

`settings.low_stock_threshold` (range 1–20, default 5, in the "WhiteTip" settings group in `config/settings_schema.json`) controls the low-stock cutoff globally.

### Custom sections (no Dawn equivalent existed)
- `sections/wt-logo-list.liquid` — supplier-brand logo grid (About page).
- `sections/wt-shipping-returns.liquid` — sticky TOC (desktop) / pill-row TOC (mobile) generated from `topic` blocks, not from parsing `page.content` HTML (deliberately more robust than heading-scraping).
- `snippets/wt-size-guide-modal.liquid` — native `<dialog>` modal (`showModal()`/`close()` via inline `onclick`, zero JS files) reading a flat Shopify Page keyed by `product.metafields.custom.size_guide_page.value`.

### Extensible contact form
`sections/contact-form.liquid` extends Dawn's native `{% form 'contact' %}` with block types `custom_text`, `custom_select`, `custom_checkboxes` — all render into Dawn's existing `.contact__fields` grid. Any `contact[Field Name]` input automatically flows into Shopify's native contact notification email with no backend/app code. An optional details column (settings-driven, not blocks) switches `.contact` to a `5fr/7fr` grid above 990px when `details_heading` is set. This section is reused for both `page.contact.json` and the Club Orders enquiry form (`page.club-orders.json`), each configuring different custom block fields.

### Admin-side dependencies (not in theme code — can't be fixed by editing files here)
The Liquid already references these; someone with Shopify Admin access has to define them before real data flows through:
- **Metafields**: `product.metafields.custom.lead_time_weeks` (integer), `custom.card_spec` (single line text), `custom.size_guide_page` (single line text, a Page handle), `custom.specs` (list.metaobject_reference → `product_spec` metaobject with `label`/`value` fields).
- **Pages**: handles `club-orders`, `contact`, `about`, `shipping-returns`, plus any `size-guide-*` pages referenced by `custom.size_guide_page`.
- **Navigation**: the `main-menu` and footer linklists referenced by `sections/header.liquid` / `sections/footer.liquid`.

### Known flagged gaps
- Club Orders hero button links to `#enquiry`, which won't precisely match Dawn's auto-generated JSON-template section id.
- About page's "Our story" section is single-column rich-text, not a true 2-column split (Dawn's `rich-text` section has no 2-col layout option).
- `card_collection.all_products_count` (used in `snippets/card-collection.liquid`) was written from memory, not confirmed against a live store.
- Search & Discovery app's filter UI is left with native Dawn styling, not reskinned to match brand.
- No live browser/visual QA or Lighthouse run has been performed on any of this — there's no connected dev server or store in the environment this was built in. `theme check` verifies Liquid/JSON correctness, not visual correctness.
