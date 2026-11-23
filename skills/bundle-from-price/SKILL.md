---
name: bundle-from-price
description: Add "From"/"Starting at" pricing for FoxSell dynamic add-ons bundle products wherever a theme prints price Liquid (product cards, price.liquid, product page). Use whenever the user mentions FoxSell bundles, dynamic bundle pricing, bundle price strategy, or wants bundle products to show a "from" price like variant-varying products do. Covers reading the bundle's price strategy metafield and reusing the theme's existing locale string instead of inventing new copy.
---

FoxSell's dynamic add-ons bundle stores its pricing strategy in a product metafield. When the strategy is `dynamic_pricing` (anything other than `fixed_pricing`), the bundle's price can vary by selected add-ons, so it should render as a "From ${price}" the same way Shopify already does for products whose variants vary in price (`product.price_varies`). Most themes already have the locale string for that case — this skill wires FoxSell's flag into the same code path instead of adding new UI.

## 1. The core snippet

This is the logic to drop in wherever a theme computes the price to render. The app namespace below is constant across installs — don't look it up, just use it:

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

Then wherever the theme currently gates its existing "from price" locale string on `product.price_varies`, OR it in:

```liquid
if product.price_varies or foxsell_show_from_price
  assign money_price = money_price | money
  assign money_price = 'products.product.price.from_price_html' | t: price: money_price
endif
```

Don't copy that `t:` key verbatim — find the theme's actual key first (step 3).

## 2. Where to add it

This is scoped to **product cards** (collection grids, featured-product sections, search results) — not full product-page variant re-render logic, and not the cart. Find the card's price-rendering snippet:

```bash
grep -rl "price_varies\|from_price" snippets/ sections/ --include="*.liquid"
```

That's almost always `snippets/price.liquid` in Dawn-family themes, included by the product card snippet (`snippets/card-product.liquid` or similar). If the theme renders price inline in the card snippet instead of through a shared `price.liquid`, add the block there directly.

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
2. View the collection/grid page: the dynamic-strategy bundle's card should show "From $X"; the fixed-strategy bundle should show a normal price.
3. Confirm a plain non-bundle product with varying variant prices still shows "From $X" (i.e. you OR'd into the existing condition rather than replacing it).
4. Confirm a plain non-bundle product with a single fixed price shows no "From" prefix.
5. Check any other place the same price snippet is reused (search results, featured collection, quick-view) picked up the change for free.
