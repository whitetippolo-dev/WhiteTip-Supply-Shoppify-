# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What this repo is

A Shopify theme for **WhiteTip Polo Supply** (an Australian canoe polo gear retailer), built on **Shopify Dawn** (not the newer block-first Skeleton Theme — an earlier version of this repo mistakenly contained Skeleton Theme and was corrected by replacing it with a real clone of `Shopify/dawn`). All WhiteTip-specific work lives on top of commit `8705c20` ("Replace Skeleton Theme with official Shopify Dawn as the real base"); everything from that commit forward is custom. Diff against it (`git diff 8705c20 dev`) to see exactly what's WhiteTip vs stock Dawn.

This repo is connected to a live store via Shopify's GitHub integration (two-way sync on `dev`): edits made in the Theme Editor land here as "Update from Shopify for theme ..." commits, and merges to `dev` sync back out to the storefront. **Always `git fetch` and check for divergence before pushing** — `origin/dev` moves on its own between sessions, from Admin edits and from other Claude sessions working the same branch. Merge, don't force-push, if it has diverged.

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

There is no JS/CSS build step, no package.json, and no test suite — Dawn is deliberately dependency-free (vanilla JS custom elements, native CSS, server-rendered Liquid). `shopify theme check` is the only automated correctness gate in this repo, and CI runs it via `shopify/theme-check-action` on every push (`.github/workflows/ci.yml`, needs `checks: write` permission or the action can't post its result), alongside a Lighthouse CI job (needs `SHOP_CLIENT_ID`/`SHOP_CLIENT_SECRET` repo secrets from a Shopify Dev Dashboard app — skips cleanly if absent).

**Doing live visual QA:** run `shopify theme dev --store=<store>.myshopify.com` (device-code browser login first time) and it prints a local preview URL plus a `?preview_theme_id=` share URL. Run `shopify theme list --store=<store>.myshopify.com` to see which theme is actually `[live]` (published) vs `[development]`/`[unpublished]` — on this store the GitHub-connected `dev` branch theme is the live one, so a push here goes straight to production, not a staging copy. The browser's Shopify preview bar/cookie is shared across all tabs in the same profile and persists after navigating away, so "visiting the plain store URL" in that browser doesn't prove you're seeing the published theme — to actually confirm a fix shipped, fetch pages with a cookie-free request (`curl -A "Mozilla/5.0" <url>`) and check for your change (see the `{% stylesheet %}` vs `{% style %}` note below for where to look).

**Baseline to maintain:** 9 pre-existing Dawn warnings, 0 errors. Any new `theme check` offense introduced by a change should be fixed before committing — that was the standard held throughout this theme's development and should continue.

**`theme check` passing is not sufficient — Shopify's GitHub sync does its own, stricter, silent validation.** A file that passes `theme check` can still be rejected on sync with no visible error in this repo (it just doesn't show up live, or an existing file goes missing/blank). Known rejection causes hit so far:
- Schema block `"name"` values over roughly 25 characters, or containing em dashes (`—`) — keep block names short and plain (`"Custom field: text"`, not `"Custom field — text"`).
- A setting `"default": ""` (empty string) on some setting types.
- Referencing a setting id that doesn't actually exist on that section type (e.g. `caption_size` vs the real `text_size` on rich-text, `image_height` vs the real `height` on image-with-text) — theme-check doesn't catch invented ids, Shopify's schema validation does.
- A numeric setting value outside its declared `min`/`max`/`step` range (e.g. `products_per_page` must land on the declared step, not just inside the min/max).
- If a whole section/template silently stops appearing live (404 or blank), suspect one of its blocks failed validation and check for the above before anything else.

`.theme-check.yml` disables two rules repo-wide: `MatchingTranslations` and `TemplateLength`.

## Branches

- `main` / `dev` on `origin` — do not force-push either. `dev` is the one connected to the live store via GitHub sync.
- `dawn-base` — local-only branch, was the working branch for the initial WhiteTip build before merging into `dev`.
- Other `claude/*` branches appear on `origin` from separate Claude Code sessions working this same repo (merged into `dev` via PRs) — expect `origin/dev` to move between sessions; always fetch and reconcile (merge, not force-push) rather than assuming local history is current.
- Never push without the user explicitly asking — this has been a standing constraint throughout this project's history.

## Architecture

### Dawn's section/template model (unmodified foundations)
- `sections/*.liquid` — reusable, schema-driven page sections (JSON `{% schema %}` block defines settings/blocks/presets editable in the Theme Editor).
- `snippets/*.liquid` — non-schema partials, included via `{% render 'name', ... %}`. Liquid disallows filter chains inline inside `{% render %}` arguments or `{% for x in Y %}` loop expressions — precompute with `{% assign %}` first or you'll hit `LiquidSyntaxError`/`UnsupportedFilterArguments`.
- `templates/*.json` — assign sections (with settings/block overrides) to a route. `page.<handle>.json` auto-applies to a Shopify Page with that exact handle (e.g. `page.contact.json` → Page handle `contact`).
- `config/settings_schema.json` (definitions) + `config/settings_data.json` (current values) — global theme settings and color schemes.
- `locales/en.default.json` — translation strings, referenced via `{{ 'key.path' | t }}`.
- `{% style %}` is Liquid-evaluated (can read `section.settings.*`); `{% stylesheet %}` is CSS scoped per-file — reusing a class across two files' `{% stylesheet %}` blocks trips theme-check's `ValidScopedCSSClass`. When a component's styles are genuinely shared by two sections (e.g. the pre-order note in both `main-cart-items.liquid` and `cart-drawer.liquid`), duplicate the small rule block in each file rather than hoisting to a global stylesheet — that's the pattern already in use.
- On the **published** theme, `{% stylesheet %}` content compiles into one cached `compiled_assets/styles.css` CDN bundle (versioned by a `?v=` query param) — it is not inline in the page HTML. `{% style %}` content does stay inline. If you're checking "did my CSS fix actually ship" by fetching the live HTML, grep the compiled bundle for `{% stylesheet %}` rules and the raw page HTML for `{% style %}` rules — looking in the wrong one makes a real fix look missing.
- Dawn's floating-label input pattern (`.field__input` + sibling `.field__label`, base.css) shrinks the label via `.field__input:not(:placeholder-shown) ~ .field__label` — a state only text/textarea inputs can match. Reusing `field__label` on a `<select>` or a `<fieldset><legend>` never triggers the shrink, so the label sits permanently on top of the value and (for a `<select>` with `appearance: none`) the value text gets hard-clipped by the fixed box height. Don't fight this with padding math — switch to Dawn's own static-label components instead: `.select` + `.select__select` + a plain `.form__label` above it (see `product-variant-picker.liquid`'s dropdown branch, and `contact-form.liquid`'s `custom_select` block) for selects; a statically-positioned custom class (not `field__label`) for fieldset legends (see `.wt-checkbox-group__legend`).
- A section/block setting with a non-blank schema default (e.g. `multicolumn`'s `button_label` defaults to literal "Button label") renders live even when nobody configured it, if the JSON template doesn't explicitly override it — including with an empty `button_link`, producing a dead placeholder CTA. When adding a block/section instance in a `templates/*.json` file, explicitly set `""` for any optional label/text setting you don't want shown, don't rely on it being absent.
- A raw `<a href>` with no `.link`/button class and no explicit `color` in its stylesheet falls back to the browser's default blue — Dawn doesn't reset anchor color globally. Any new custom link element needs its own `color` rule (see `.wt-logo-list__link`, `.wt-breadcrumb a`).
- `sections/header-group.json` and `sections/footer-group.json` carry a header comment saying they're auto-generated and may be overwritten by the Theme Editor — true in practice (see "two-way sync" above). Expect hand edits to these two files specifically to get clobbered or reformatted by the next Admin sync; treat remote changes to them as authoritative over local assumptions.
- A Page assigned a block-based JSON template (e.g. `page.shipping-returns.json` → `wt-shipping-returns` section with `topic` blocks) has an empty `page.content` in Liquid — the body lives in the blocks' settings, not the Page's rich-text body. Don't write other sections that read `pages[handle].content` and expect it to reflect what's visually on that page; it won't. (This caused the PDP's "Shipping" tab to render empty until it was given its own hardcoded summary instead of pulling from the page.)

### WhiteTip brand layer
- `assets/whitetip-tokens.css` — the single source of truth for brand tokens (colors, spacing scale, type scale, button/input/focus tokens, effects), linked in `layout/theme.liquid` right after `base.css`. Layers on top of, rather than replacing, Dawn's own `--color-*`/`--font-*` custom properties.
- Dawn's `scheme-1` is locked to the Ocean palette and `scheme-2` to Ocean-900 (`config/settings_data.json`) — use these two existing schemes for new sections needing brand colors rather than inventing new ones or hardcoding hex values outside the token file.
- Fonts use Dawn's native Google-Fonts picker (`settings.type_header_font`/`type_body_font`, both set to Montserrat) rather than self-hosted woff2 — keeps Theme Editor font controls live. Don't reintroduce self-hosted fonts without a specific reason.

### Centralized stock/pre-order logic
Pre-order detection is a single rule applied everywhere, never re-derived per caller: `available == true AND inventory_management == 'shopify' (tracked) AND inventory_quantity <= 0`. Any tracked, still-sellable, zero-stock item is a pre-order — full stop. There is **no lead-time metafield or display anymore**: an earlier version gated this on `custom.lead_time_weeks` and showed "Pre-order · N wks", but lead time changes too often to promise on the product page, so that metafield and its display were removed everywhere (badge, buy-button note, variant sub-labels). Don't reintroduce a lead-time weeks display without the user asking for it back.

The three-state stock badge (in-stock / low-stock "Only N left" / pre-order "Pre-order") is centralized in `snippets/wt-card-badge.liquid`, which takes raw facts (`available`, `inventory_tracked`, `inventory_quantity`, `low_stock_threshold`, `id`, `sold_out_color_scheme`) and computes the state internally. Callers (`card-product.liquid`, `main-product.liquid`, `main-cart-items.liquid`, `cart-drawer.liquid`) must stay thin and pass only facts — badge-state branching was deliberately moved out of `card-product.liquid` into this snippet after it tripped theme-check's `LiquidComplexity` rule (120-line cap). Keep new call sites to the same pattern instead of reintroducing inline branching.

`settings.low_stock_threshold` (range 1–20, default 5, in the "WhiteTip" settings group in `config/settings_schema.json`) controls the low-stock cutoff globally.

### Custom sections (no Dawn equivalent existed)
- `sections/wt-logo-list.liquid` — supplier-brand logo grid (About page).
- `sections/wt-shipping-returns.liquid` — sticky TOC (desktop) / pill-row TOC (mobile) generated from `topic` blocks, not from parsing `page.content` HTML (deliberately more robust than heading-scraping).
- `snippets/wt-size-guide-modal.liquid` — native `<dialog>` modal (`showModal()`/`close()` via inline `onclick`, zero JS files) reading a flat Shopify Page keyed by `product.metafields.custom.size_guide_page.value`.

### Native-Dawn extensions (fixes made directly to stock Dawn sections, not WhiteTip-prefixed)
- `sections/rich-text.liquid` gained a `buttons` block type (`button_label_1/_2`, etc. — a real pair-of-buttons block) because the About page CTA was built against a block type Dawn doesn't actually ship; it didn't surface in `theme check`, only on live sync. Use this `buttons` type for any future two-button rich-text block rather than inventing another one.
- `sections/main-product.liquid` gained a `wt_description` accordion block type that renders `product.description` directly — Dawn's PDP has no native description display either.

### Extensible contact form
`sections/contact-form.liquid` extends Dawn's native `{% form 'contact' %}` with block types `custom_text`, `custom_select`, `custom_checkboxes` — all render into Dawn's existing `.contact__fields` grid. Any `contact[Field Name]` input automatically flows into Shopify's native contact notification email with no backend/app code. An optional details column (settings-driven, not blocks) switches `.contact` to a `5fr/7fr` grid above 990px when `details_heading` is set. This section is reused for both `page.contact.json` and the Club Orders enquiry form (`page.club-orders.json`), each configuring different custom block fields.

### Admin-side dependencies (not in theme code — can't be fixed by editing files here)
The Liquid references these; someone with Shopify Admin access has to define/populate them:
- **Metafields**: `custom.card_spec` (single line text, product), `custom.size_guide_page` (single line text, a Page handle, product), `custom.specs` (list.metaobject_reference → `product_spec` metaobject with `label`/`value` fields, product). (`custom.lead_time_weeks` is no longer referenced anywhere — see pre-order logic above.)
- **Pages**: handles `club-orders`, `contact`, `about`, `shipping-returns`, plus any `size-guide-*` pages referenced by `custom.size_guide_page`. These now appear live (live-sync commits reference real catalog/collection data), but their content is owned in Admin, not here.
- **Navigation**: `main-menu` (header, populated) and `footer` (footer link list, wired up via `footer-group.json`'s `quick-links` block, but currently empty — see "Known flagged gaps").

### Known flagged gaps
- About page's "Our story" section is single-column rich-text, not a true 2-column split (Dawn's `rich-text` section has no 2-col layout option).
- `card_collection.all_products_count` (used in `snippets/card-collection.liquid`) was written from memory, not confirmed against a live store.
- Search & Discovery app's filter UI is left with native Dawn styling, not reskinned to match brand.
- No catalog data currently exercises the low-stock or sold-out badge states, the size-guide modal, or the PDP specs/description tabs (no product is below the low-stock threshold or truly unavailable, and `custom.specs`/`custom.size_guide_page`/`product.description` aren't populated on any checked product) — that logic is implemented and was verified by reading the code, but hasn't been seen rendering live. Verify against real data before assuming it's broken if it looks wrong later.
- The footer's `footer` navigation menu exists and is wired up (`footer-group.json`'s `quick-links` block) but currently has zero links in Admin — renders as a heading with nothing under it. Content fix in Admin, not a theme bug.
- `theme check` still doesn't catch sync-time-only rejections (see the `{% stylesheet %}`/`{% style %}` note above) — a full live-QA pass (via `shopify theme dev` + browser automation, confirmed against the published theme via cookie-free `curl`) was done once and found several real rendering bugs that `theme check` missed entirely (clipped/overlapping form fields, dead placeholder buttons from unset schema defaults, unstyled links). Don't assume a green `theme check` means a page renders correctly — if you can run a live preview, do.
- `review/` holds non-theme content-tracking spreadsheets (e.g. a product-content review sheet) — not theme code, safe to ignore when working on Liquid/JSON.
