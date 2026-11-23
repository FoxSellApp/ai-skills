---
name: bundle-from-price
description: Add "From"/"Starting at" pricing for FoxSell dynamic add-ons bundle product cards, including shared price snippets called by cards. Use whenever the user wants FoxSell dynamic bundle cards to show a "from" price like variant-varying products do. Covers reading the bundle's price strategy metafield and reusing the theme's existing locale string instead of inventing new copy. Product-page variant pricing and cart prices are outside this skill's scope.
---

FoxSell's dynamic add-ons bundle stores its pricing strategy in a product metafield. When the strategy is `dynamic_pricing` (anything other than `fixed_pricing`), the bundle's price can vary by selected add-ons, so it should render as a "From ${price}" the same way Shopify already does for products whose variants vary in price (`product.price_varies`). Most themes already have the locale string for that case — this skill wires FoxSell's flag into the same code path instead of adding new UI.

## 1. The core snippet

First trace the card's price render call and identify the product object in that scope. The example below assumes the snippet receives `product`; for inline card markup using `card_product`, use `card_product` for both the metafield lookup and `price_varies` check. Do not fall back to the page's global `product`, which may refer to a different item.

Add this logic to the card's product-level price path. The app namespace below is constant across installs — don't look it up, just use it:

```liquid
{%- liquid
  assign foxsell_app_namespace = 'app--67872686081'
  assign foxsell_bundle_config = product.metafields[foxsell_app_namespace].dynamic_add_ons_bundle.value
  assign foxsell_bundle_price_strategy = foxsell_bundle_config.settings.value.price.strategy
  assign foxsell_show_from_price = false
  if foxsell_bundle_config != blank and foxsell_bundle_price_strategy != 'fixed_pricing'
    assign foxsell_show_from_price = true
  endif
-%}
```

Extend only the existing `price_varies` predicate with the bundle flag; preserve every surrounding selected-variant, target, and other eligibility guard. Use nested conditions to keep the guards outside the OR rather than relying on Liquid's operator precedence. For example, a theme whose original condition is `target == product and product.price_varies` becomes:

```liquid
if target == product
  if product.price_varies or foxsell_show_from_price
    assign money_price = 'products.product.price.from_price_html' | t: price: money_price
  endif
endif
```

This example assumes `money_price` is already formatted by the theme. Retain the existing formatting path, including `money_with_currency` when enabled, and pass that string directly to the translation. If the theme starts with a numeric price, format it once using its existing money filter; never apply `money` to an already-formatted string.

Don't copy that `t:` key verbatim — find the theme's actual key first (step 3). Likewise, use the theme's actual guards; do not introduce `target` into a theme that does not define it.

## 2. Where to add it

This is scoped to **product cards** (collection grids, featured-product sections, search results) — not full product-page variant re-render logic, and not the cart. Find the card's price-rendering snippet:

```bash
grep -rl "price_varies\|from_price" snippets/ sections/ --include="*.liquid"
```

That's almost always `snippets/price.liquid` in Dawn-family themes, included by the product card snippet (`snippets/card-product.liquid` or similar). If the theme renders price inline in the card snippet instead of through a shared `price.liquid`, add the block there directly.

If the price snippet is shared with product pages or quick-view variant pricing, pass an explicit card-only flag from the relevant card render calls and require that flag when enabling `foxsell_show_from_price`. Leave it false for all other callers. Keep the theme's existing non-bundle `price_varies` behavior and exact-variant safeguards unchanged.

## 3. Reuse the theme's existing locale key — never invent copy

Before touching any Liquid, check what "from price" string the theme already has:

```bash
grep -n "from_price\|starting_at\|price_varies" locales/en.default.json locales/en.default.schema.json 2>/dev/null
```

- If `products.product.price.from_price_html` (or similar) exists, use that exact key — it's already wired to whatever wording/styling the theme uses for variant-varying prices.
- If no such key exists, search more loosely for any "From" / "Starting at" phrasing already shipped in the locale file (themes sometimes nest it under a different scope, e.g. `products.product.price.regular_price` siblings) and reuse that key.
- Only if genuinely nothing exists should you add a new locale key — and if so, match the exact phrasing pattern of neighboring price keys in that file rather than writing new copy from scratch.

## 4. Verify

No automated way to test this against live metafield data, so check by hand in a dev store:

1. Set one product's FoxSell bundle strategy to a dynamic strategy (anything other than `fixed_pricing`) and another to `fixed_pricing`.
2. View the collection/grid page: the dynamic-strategy bundle's card should show "From $X"; the fixed-strategy bundle should retain the theme's normal behavior (including "From" if its variant prices vary).
3. Confirm a plain non-bundle product with varying variant prices still shows "From $X" (i.e. you OR'd into the existing condition rather than replacing it).
4. Confirm a plain non-bundle product with a single fixed price shows no "From" prefix.
5. Check other card callers (search results, featured collection) and verify that each looks up the displayed card's metafield, including cards rendered on a different product's page.
6. Check product pages and quick-view before and after selecting variants, for bundle and non-bundle products: existing exact-variant prices and "From" eligibility must remain unchanged.
7. Check the numeric amount and currency display with currency-code display enabled and disabled; the amount must be formatted once with the theme's existing filter.
