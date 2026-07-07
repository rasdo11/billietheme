# CLAUDE.md — repo context for developers & code sessions

Orientation for anyone (human or AI) touching this repo. Read this before making changes.

## What this is

A **custom Shopify theme** for **"Olnian / ØLNIAN"** — a DTC supplements brand
(tagline: _"Pure supplements. Made the week you order."_). The design language is
Billie-inspired (playful, warm, conversion-focused PDP), but this is **not** a Billie theme and
**not** based on Dawn. The repo name `billietheme` is historical — the brand is ØLNIAN
(see `config/settings_schema.json` → `theme_name: "Olnian"`, brand `ØLNIAN`).

## Architecture & conventions

- **No build step, no bundler, not Dawn.** You edit Liquid, CSS, and JS directly and deploy the
  theme as-is. There is nothing to compile.
- **Two monolithic asset files** hold essentially all styling and behavior:
  - `assets/theme.css` (~57 KB) — every style in the theme.
  - `assets/theme.js` (~22 KB) — all interactivity.
  Because they're large, **search within them** rather than scanning top to bottom.

### CSS
- Design tokens are CSS custom properties declared in `:root` at the top of `assets/theme.css`:
  `--tangerine` (coral, the primary/accent), `--espresso` (text), `--blush` (background),
  `--gold`, `--mint`, plus `--display-font` / `--body-font`.
- These tokens are **re-declared from Theme Editor settings** inline in `layout/theme.liquid`
  (`:root { --tangerine: {{ settings.color_accent_primary }}; ... }`). So the CSS file has static
  fallback values, and the live values come from theme settings. Change brand colors/fonts in the
  Theme Editor (or `config/settings_data.json`), not by hardcoding.
- Class naming is **BEM-ish**: `block__element--modifier` (e.g. `product__atc-row`,
  `pdp-gallery__thumb--active`). Shared buttons: `btn`, `btn--primary`, `btn--outline`, `btn--ghost`.
- Responsive breakpoint is **`max-width: 900px`** for the mobile layout; a `min-width: 901px` block
  handles desktop-only PDP gallery layout.

### JavaScript
- Vanilla JS in a single IIFE (`assets/theme.js`), no framework, no dependencies.
- Helpers: `$` / `$$` (querySelector / querySelectorAll-to-array).
- Organized as **module objects** (`CartAPI`, `Drawer`) plus **`init*()` functions**:
  `initProductForm`, `initGallery` (+ lightbox), `initStickyBar`, `initExpertModal`,
  `initProductAccordions`, `initMobileNav`, `initSwatchCarousels`, `initCartDrawerEvents`.
- Everything is wired up in the `DOMContentLoaded` handler near the bottom of the file.
- Behavior is bound via **`data-*` attribute hooks** — e.g. `data-product-form`,
  `data-product-submit`, `data-cart-open` / `data-cart-close`, `data-accordion`,
  `data-lightbox-open/close/prev/next`, `data-sticky-atc`. No inline `onclick` handlers.
  When adding interactivity, add a `data-*` hook in Liquid and handle it in an `init*()` function.

## Repo layout

- `layout/theme.liquid` — the HTML shell: maps settings → CSS variables, loads Google Fonts,
  renders `header-group` / `footer-group`, and the cart drawer. Global `<head>`.
- `sections/` — all theme sections (see map below).
- `templates/` — page templates. JSON templates (`*.json`) compose sections; Liquid templates
  (`*.liquid`) are used for account/utility pages.
- `config/` — `settings_schema.json` (setting **definitions**) and `settings_data.json`
  (setting **values**).
- `assets/` — `theme.css` and `theme.js`.

## Templates & sections map

**Product templates:** `templates/product.json` is the **default PDP**. `product.magnesium.json`
and `product.brain-bundle.json` are **alternate PDP templates** you assign per-product in the
Shopify admin (Product → Theme template). `index.json` is the homepage; `collection.json` the
collection page.

**Notable sections:**
- `main-product` — the PDP: image gallery + lightbox, buy box, subscribe card, sticky cross-sell
  bar, and the "Talk to an expert" modal. Most conversion logic lives here.
- `product-science`, `product-faq`, `testing-strip` — supporting PDP sections (stats, FAQ, QA badges).
- `header`, `footer`, `cart-drawer`, `announcement-bar` — chrome.
- `hero-editorial`, `how-it-works`, `sku-row`, `stat-strip`, `proof`, `batch-story`, `newsletter`,
  `quiz`, `main-collection` — homepage / marketing sections.

## PDP notes — please don't regress these

The product page (`sections/main-product.liquid` + PDP rules in `assets/theme.css`) was
deliberately tuned. Before changing it, know:

- **The variant/color ("pick a color") picker was intentionally removed.** The product is sold as a
  single configuration; the hidden `<input name="id">` in the buy form carries the variant id. Do
  **not** re-add a variant/color picker unless there's a real multi-variant requirement.
- **Mobile above-the-fold order is intentional.** In `assets/theme.css` under `@media (max-width:
  900px)`, the buy form is `display: flex; flex-direction: column` with explicit `order:` values on
  its children so the **Add-to-cart button sits directly under the price**, while the gift banner,
  quantity stepper, subscribe card, and express-checkout button drop below it. The mobile hero
  height is capped and the grid gap reduced so the CTA clears the fold on ~390px screens. If you
  reorder the buy box, keep these `order:` rules coherent.
- **Subscribe & Save UI is data-gated.** The subscribe card and the `/mo` price line only render
  when the product has a selling plan (`_first_selling_plan` in the Liquid). With no selling plan
  they hide gracefully.
- **Sticky bar is the fallback CTA.** `#PdpStickyBar` slides up once the main Add-to-cart scrolls
  out of view — it's the safety net for the buy action on long pages.

## Shopify data dependencies (admin-side)

Some on-page elements require store configuration to appear:
- **Product metafields:** `custom.eyebrow` (kicker above the title) and `custom.ingredients`
  (feeds the Ingredients accordion; leave empty to hide).
- **Subscriptions:** a selling plan (via the Shopify Subscriptions app) is required for the
  Subscribe & Save card / sub-price.
- **Section content:** rating, tagline, benefit pills, "what it supports", cross-sell product, and
  gift banner are set through the Theme Editor on the section.
- **Template assignment:** products must be assigned a template that renders `main-product`.

## Local dev & testing

- There is **no build, lint, or test step**, and **Liquid cannot be rendered locally**.
- To preview real pages, use the Shopify CLI (`shopify theme dev`) or a theme preview in the admin.
- For **CSS-only** behavior you can validate without Shopify, load `assets/theme.css` into a static
  HTML harness that mirrors the relevant DOM and screenshot it (e.g. verify the PDP mobile
  above-the-fold at ~390px width). This is how the PDP mobile ordering was verified.

## Editing & deploy caveats

- `templates/*.json` and `config/*.json` are **auto-generated** and can be overwritten by the
  Shopify Theme Editor (note the warning header in `templates/product.json`). Prefer making
  content/settings changes in the Theme Editor; hand-edit generated JSON only when necessary.
- Develop on a branch and push; the theme is deployed through Shopify (GitHub integration /
  `shopify theme push`), not from a local build artifact.
