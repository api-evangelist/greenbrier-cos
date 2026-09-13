---
name: greenbrier-railcar-catalog-search
description: >-
  Find Greenbrier railcar models by the commodity they haul, the equipment class they belong to, or
  their capacity, using the public railcar catalog and its four purpose-built taxonomies.
api: Greenbrier Railcar Catalog API
base_url: https://www.gbrx.com/wp-json
operations:
  - listCargoTypes
  - listRailcarTypes
  - listFluidCapacities
  - listCargoCapacities
  - listRailcars
  - getRailcar
generated: '2026-09-12'
method: generated
source: openapi/greenbrier-cos-railcar-catalog-api-openapi.yml
---

# Search the Greenbrier railcar catalog

Greenbrier's railcar product catalog is a WordPress custom post type served over the public REST
API: 85 published models, classified by four vocabularies Greenbrier built for this catalog —
railcar type, cargo type, fluid capacity and cargo capacity.

## Before you start

- **No authentication.** Anonymous `GET` only.
- **Resolve terms to IDs first.** Every filter takes numeric term IDs, not names. `cargo_type=grain`
  will not work; `cargo_type=180` will.
- **Trim the payload.** Add `_fields=id,slug,title,railcar_type,cargo_type` to every collection call.
  The default record embeds the model's fully rendered marketing HTML in `content.rendered`, which is
  usually not what you want and is much larger than what you do want.

## Steps

1. **Resolve the vocabulary.** Call `listRailcarTypes`
   (`GET /wp/v2/railcar_type?per_page=100&_fields=id,slug,name,count`) and/or `listCargoTypes`
   (`GET /wp/v2/cargo_type?per_page=100&_fields=id,slug,name,count`). Match the user's words against
   `name` and `slug`. The `count` on each term tells you how many railcars carry it — a term with
   `count: 0` (there are some, e.g. `mill-gondola`) will return nothing, and you should say so rather
   than reporting an empty result as a failure.

   The 18 railcar types as of 2026-09-12 include agricultural covered hoppers (12 models), chemical
   tank cars (13), gondolas (12), energy tank cars (9), industrial covered hoppers (8), agricultural
   tank cars (5), auto carriers (4), standard plate F boxcars (4), intermodal units (4) and
   pressurized differential covered hoppers (3). There are 66 cargo types.

2. **Filter the catalog.** Call `listRailcars`
   (`GET /wp/v2/railcars?cargo_type={id}&per_page=100&_fields=id,slug,title,link`). You can combine
   `railcar_type`, `cargo_type`, `fluid_capacity` and `cargo_capacity`, and each has an `_exclude`
   counterpart. Read `X-WP-Total` from the response headers for the count — do not count the array
   you were handed, because it is one page.

   Worked example: `GET /wp/v2/railcars?cargo_type=180&_fields=id,slug,title&per_page=3` returned
   `X-WP-Total: 2` — the Dura-Max™ and the 1150 Open-Top Hopper Railcar.

3. **Page through, if needed.** `per_page` maxes at 100 and `X-WP-TotalPages` tells you how many
   pages there are. There is also an RFC 5988 `Link` header with `rel="next"`.

4. **Fetch the model.** Call `getRailcar` (`GET /wp/v2/railcars/{id}`) for the full record. The
   marketing description is in `content.rendered` as HTML; the specification data Greenbrier attaches
   to the model sits in the `acf` block. Both are provider-authored and should be quoted, not
   paraphrased into numbers you did not read.

5. **Link the human to the page.** Every record carries `link` — the canonical gbrx.com URL for that
   model. Give it to the user; it is the page Greenbrier maintains.

## Honest limits

- The catalog says nothing about availability, lead time, price or whether a model is still in
  production. Greenbrier publishes no such field. Do not infer "available" from "published".
- There is no fitment relationship between the catalog and the aftermarket parts store, and no link
  from a model to a gauge table.
- `menu_order` is a display-ordering field for the website, not a ranking.
