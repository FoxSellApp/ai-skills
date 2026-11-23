---
name: bundle-line-items
description: Add bundle/combo component display under line items on a Shopify theme's cart page and cart drawer, using item.item_components (Shopify's native bundles API). Use whenever the user wants cart line items to show what's inside a bundle, combo, or multi-component product, mentions item_components, or asks to "show bundle contents in cart". Covers locating the right theme files, inserting the Liquid, and adding matching CSS.
---

Shopify's native bundles feature attaches child line items to a parent line item via `item.item_components` (each has `.title`, `.quantity`, `.variant`, `.url`, etc.). Themes don't render this by default, so cart pages and cart drawers show the bundle as one opaque line. This skill wires up the display in both places.

## 1. Find the real line-item loop

Theme file names vary even among Dawn forks, so don't assume paths — grep for them:

```bash
grep -rl "cart.items" sections/ snippets/ --include="*.liquid"
grep -rl "item_components\|item\.quantity" sections/ snippets/ --include="*.liquid"
```

You're looking for two loops (they may live in shared snippets included by both the page and the drawer, in which case one edit covers both):
- The **cart page** loop, usually `sections/main-cart-items.liquid` or a snippet it includes (e.g. `snippets/cart-items.liquid`, `snippets/cart-item.liquid`).
- The **cart drawer** loop, usually `sections/cart-drawer.liquid` or `snippets/cart-drawer-item.liquid`.

If the theme already shares one snippet (`{%- render 'cart-item', item: item -%}`) between page and drawer, add the bundle block there once instead of duplicating it.

## 2. Insert the bundle block

Place this immediately after the line item's title/variant markup, inside the `for item in cart.items` loop, in each location found above:

```liquid
{%- if item.item_components.size != 0 -%}
  <ul class="cart-items__bundle list-unstyled">
    {%- for component in item.item_components -%}
      <li>
        {{- component.title | escape -}}
        {%- if component.quantity > 1 -%}
          <span> × {{ component.quantity }}</span>
        {%- endif -%}
      </li>
    {%- endfor -%}
  </ul>
{%- endif -%}
```

Escape component and variant titles at the HTML output boundary so characters such as `<`, `>`, `&`, and quotes render as text rather than markup. Keep the raw variant title for the `Default Title` comparison.

Match the surrounding markup's indentation and whitespace-control style (`{%-`/`-%}`) rather than copy-pasting blind — themes are inconsistent about this and stray whitespace shows up in the cart.

Reuse the theme's existing list-reset class (`list-unstyled`, `list-unstyled-cart`, etc.) if one already exists elsewhere in the theme instead of introducing a new one.

## 3. Add minimal CSS

Bundle components should read as a subordinate detail, not a second line item. Add a small, clearly-scoped rule to the theme's main stylesheet (e.g. `assets/base.css`), matching the theme's existing naming convention (BEM, utility classes, whatever it already uses):

```css
/* Bundle component list under cart line items */
.cart-items__bundle {
  margin: 0.25rem 0 0;
  padding: 0;
  font-size: 0.85em;
  color: rgba(var(--color-foreground), 0.75);
}
```

Keep it to spacing/size/color only — don't restyle layout. If the theme has no CSS custom property like `--color-foreground`, use its existing muted-text convention instead (check how it styles e.g. variant text or SKU text on the same line item, and match that).

## 4. Optional: show variant options per component

If the bundle's components have variant options worth surfacing (e.g. "Color: Red"), extend the `<li>`:

```liquid
<li>
  {{- component.title | escape -}}
  {%- if component.variant.title != "Default Title" -%}
    <span> ({{ component.variant.title | escape }})</span>
  {%- endif -%}
  {%- if component.quantity > 1 -%}
    <span> × {{ component.quantity }}</span>
  {%- endif -%}
</li>
```

Only add this if the user asks for it or the store's bundles actually vary by option — it's noise otherwise.

## 5. Verify

Verify the rendered snippets with representative title and quantity fixtures, then check the theme integration by hand:

1. `shopify theme dev` (or push to a development theme) and preview the store.
2. Add a bundle product to the cart.
3. Open the cart drawer — confirm components list under the parent line, quantities show only when >1, and non-bundle line items are unaffected.
4. Go to `/cart` — confirm the same for the full cart page.
5. Check a cart with a bundle whose component quantity is 1 and one with quantity >1, to confirm the `×` suffix only appears when expected.
6. Check mobile width — the component list shouldn't overflow or collide with price/remove controls.
7. Use component and variant titles containing `<b>Sample</b>`, `&`, and quotes in a development fixture. Confirm both the basic and optional variant examples display literal text without creating elements; `Default Title` stays hidden and quantity behavior is unchanged.
