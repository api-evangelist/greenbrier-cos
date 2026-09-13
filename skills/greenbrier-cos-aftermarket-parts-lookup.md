---
name: greenbrier-aftermarket-parts-lookup
description: >-
  Find a Greenbrier aftermarket railcar part by keyword, SKU, category or attribute, and read its
  price, stock availability, dimensions and variations from the public parts catalog.
api: Greenbrier Aftermarket Parts Store API
base_url: https://www.gbrx.com/wp-json
operations:
  - listStoreProductCategories
  - listStoreProductAttributes
  - listStoreProducts
  - getStoreProduct
  - getStoreCollectionData
generated: '2026-09-12'
method: generated
source: openapi/greenbrier-cos-parts-store-api-openapi.yml
---

# Look up a Greenbrier aftermarket part

Greenbrier's aftermarket parts catalog — gaskets, outlet gates, hatch covers, valves, pipe
assemblies and other replacement components for Greenbrier-built equipment — is served by the
WooCommerce Store API on www.gbrx.com. 207 products across 140 categories and 33 attribute
vocabularies as of 2026-09-12.

## Before you start

- **No authentication for reads.** The product, category and attribute routes answer anonymously.
- **Do not touch cart or checkout.** `/wc/store/v1/cart` and `/wc/store/v1/checkout` are
  session-scoped, require a nonce, and are deliberately outside this catalogued contract. This skill
  is read-only: it finds parts, it does not buy them. Hand the user the `permalink` and let a human
  complete the order at https://shop.gbrx.com/s/.
- **SKUs are not clean.** Real SKUs observed include `1023003`, `101445-06` and `#101445-06` — the
  same part family appears with and without a leading `#`. Search on the number without punctuation
  and compare results, rather than trusting one exact-match query.

## Steps

1. **Search by keyword first.** Call `listStoreProducts`
   (`GET /wc/store/v1/products?search=gasket&per_page=20`). Read `X-WP-Total` for the true count.
   Worked example: `search=gasket` returned `X-WP-Total: 4`.

2. **Or narrow by category.** Call `listStoreProductCategories`
   (`GET /wc/store/v1/products/categories?per_page=100`) to get the tree. Each node carries `parent`
   (0 at the root) and `count`. Then filter with
   `GET /wc/store/v1/products?category={id}`.

3. **Or narrow by attribute.** Call `listStoreProductAttributes`
   (`GET /wc/store/v1/products/attributes`) for the 33 global vocabularies — dimensions, material,
   nominal thickness, o-ring, PSI, steam connections, thickness, weight. Filter with `attributes` and
   control AND/OR semantics with `attribute_relation`.

4. **Understand what came back.** Each product carries:
   - `type` — `simple` or `variable`. A `variable` product is a family; its real orderable children
     are listed in `variations[]` and each child points back through `parent`.
   - `prices` — an object, not a number: `price`, `regular_price`, `sale_price`, `currency_code`,
     and `currency_minor_unit`. **Read `currency_minor_unit` before formatting.** The Store API
     returns prices in the currency's minor unit, so a `price` of `"100"` with
     `currency_minor_unit: 2` is $1.00 — divide by 10^`currency_minor_unit` before showing a figure to
     anyone. `price_html` was empty on the products sampled on 2026-09-12, so there is no
     provider-rendered string to fall back on.
   - `is_in_stock`, `is_on_backorder`, `low_stock_remaining`, `stock_availability`
   - `sku`, `dimensions`, `weight`, `formatted_dimensions`, `formatted_weight`
   - `permalink` — the page a human should be sent to

5. **Fetch the detail.** Call `getStoreProduct` (`GET /wc/store/v1/products/{id}`) for one part.

6. **Facet, if the user is browsing.** `getStoreCollectionData`
   (`GET /wc/store/v1/products/collection-data`) returns `price_range`, `attribute_counts`,
   `rating_counts`, `stock_status_counts` and `taxonomy_counts` for a query — the aggregates behind
   the store's own filters, and much cheaper than fetching every product to count them yourself.

## Honest limits

- **No fitment data.** Nothing in this catalog says which railcar model a part fits. That information
  lives only in the human-readable description. Never assert fitment the API did not state.
- **Stock is a website field, not a live inventory contract.** Treat it as indicative.
- Product tags are served but empty (0 terms as of 2026-09-12); do not build a flow on them.
