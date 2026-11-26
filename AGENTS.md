Always refer to the FoxSell partner docs on bundle data structures before reading bundle configuration:

https://partner-docs.foxsell.app/data-structures/mix-and-match-bundles

FoxSell stores the bundle configuration in the `dynamic_add_ons_bundle` metaobject metafield (namespace `app--67872686081`, key `dynamic_add_ons_bundle`), covering categories, products, variants, options, add-ons, and pricing strategy. Template-specific settings (e.g. Step, Glow) live separately in the `additional_settings` JSON metafield on the same namespace, which supplements rather than replaces the base metafield. Use this doc as the source of truth for field names/shapes instead of guessing from the Liquid alone — it also covers add-to-cart line item properties and the Storefront API query shapes.

Build a FoxSell "Mix & Match" bundle section for this theme, starting from FoxSell's base template instead of building from scratch:

https://github.com/FoxSellApp/foxkit-bundle-templates/tree/main/templates/base

`templates/base` mirrors Shopify's own theme folder structure (`sections/`, `blocks/`, `snippets/`, `assets/`, `templates/`), so copy each file into the matching folder in this theme as-is — no restructuring needed. The main bundle section is the entry point; it pulls in the supporting blocks/snippets for the product card, options, bundle summary, and styling.

After copying the files in, adjust naming, markup, and CSS to match the theme's existing conventions (BEM, utility classes, whatever it already uses) rather than leaving the base files untouched.

The JS/CSS assets are compiled starting points, not off-limits — once the section is wired up and working, edit them as needed to match the user's requirements (behavior, styling, layout).

Respect the bundle's per-product settings, read from `product.metafields.custom.foxsell_additional_settings`:

- `quantity_rules.strategy`: `"any"` | `"fixed"`
- `quantity_rules.max`: `"cap_at_highest"` | `"no_cap"`
- `add_on_settings.strategy`: `"add_on_step"` | `"automatic_add"`

Deliverable: a working bundle section (section + blocks + snippets + schema) added to this theme, built on FoxSell's base structure, styled to match the theme, with the section added to the relevant product template.
