---
name: greenbrier-tank-car-gauge-table-lookup
description: >-
  Look up the certified gauge table for a specific tank car from Greenbrier's public Gauge Table
  Directory — confirm the reporting mark is covered, confirm the car number is held, retrieve the
  innage/outage capacity curve, and export it as Pdf, Csv or Txt in gallons or liters.
api: Greenbrier Tank Car Gauge Table API
base_url: https://tankcar.gbrx.com
operations:
  - listReportingMarks
  - listValidCarNumbers
  - getGaugeTable
  - listGaugeTables
  - exportGaugeTable
generated: '2026-09-12'
method: generated
source: openapi/greenbrier-cos-gauge-table-api-openapi.yml
---

# Look up a tank car gauge table

A gauge table converts a measured liquid depth inside a tank car into a volume. It is the document a
loader, a shipper and a regulator use to agree how much product is in a car. Greenbrier serves its
gauge tables as public JSON with no key and no signup.

## Before you start

- **No authentication.** Every call below is a plain anonymous `GET`. Do not send an Authorization
  header; there is nothing to send.
- **No rate-limit signal.** Nothing in the response tells you your headroom. Be conservative and
  back off on any 403 or reset, because no `Retry-After` will be sent.
- **The status code lies.** `getGaugeTable` answers **HTTP 200 with a JSON `null` body** for a car it
  does not hold. Parse the body and test for null. The range operations express the same condition as
  an empty array.

## Steps

1. **Confirm the reporting mark is covered.** Call `listReportingMarks`
   (`GET /api/car/marks`). It returns a flat array of AAR reporting marks — 144 of them as of
   2026-09-12, including `GBRX`, `GATX`, `UTLX`, `NATX`, `ACFX` and `SHPX`. If the caller's mark is
   not in that array, stop and say so: Greenbrier holds nothing for it. Do not guess a near match —
   `SHQX`, `SHPX`, `SHP `, `SHQ` and `SHX` are all distinct marks on this list.

2. **Confirm the car number is held.** Call `listValidCarNumbers`
   (`GET /api/car/{mark}/{start}/-/{end}/valid-car-number-list`). The literal `-` between the two
   numbers is the range separator and is required. Each entry is `{number, carNumberForDisplay}`.
   An empty array `[]` is a correct, expected answer meaning Greenbrier holds no table in that range —
   it is not an error.

   **Bound the range.** This operation is unpaginated and has no server-side page size.
   `GBRX 700000-724234` returned 982 KB and 20,049 records in one response. If you only need to check
   one car, ask for a range of one: `/api/car/GBRX/700000/-/700000/valid-car-number-list`.

3. **Retrieve the table.** Call `getGaugeTable`
   (`GET /api/car/{mark}/{carNumber}/gauge-table`). You get back:
   - `carNumber` — mark and number joined by a space, e.g. `"GBRX 700000"`
   - `tareWeight` — empty weight in pounds
   - `shellFillCapacity` — shell full capacity in gallons
   - `gaugeTableDate` — the date the table was certified; treat an old date as normal, not as an error
   - `capacities[]` — the curve, one entry per quarter-inch of height, each `{height, innage, outage}`

   Test the parsed body for `null` before touching any field.

4. **Read a depth off the curve.** `innage` is the volume of product *below* a measured height;
   `outage` is the empty space *above* it. Pick the entry whose `height` matches the gauged
   measurement. The curve is quarter-inch granular — if the measurement falls between two entries,
   interpolate and say that you did. Do not round silently: on a 30,100-gallon car a quarter inch is
   real money.

5. **Export, if a document is wanted.** Call `exportGaugeTable`
   (`GET /export/{fileType}`) with `fileType` one of `Pdf`, `Csv`, `Txt`, plus query parameters
   `capacityType` (`Innage` or `Outage`), `unitType` (`Gallons` or `Liters`), `mark`, `start` and
   `end`. Those six values are the exact set Greenbrier's own export form offers; anything else is a
   guess. The Csv output is `"CAR NUMBER","DISTANCE","INNAGE"` with one row per increment.

6. **For a whole fleet range**, `listGaugeTables`
   (`GET /api/car/{mark}/{start}/-/{end}/gauge-tables`) returns the full table for every held car in
   the range. Same warning: unpaginated, and each table is roughly 34 KB, so a hundred cars is
   several megabytes. Prefer `listValidCarNumbers` first, then fetch individually.

## What to tell the user when it misses

Say which of the two walls you hit — "Greenbrier does not publish gauge tables for the `XXXX` mark"
is a different answer from "the mark is covered but car number 12345 is not in Greenbrier's set".
Both are honest, useful answers. Neither is an API failure.

## What this API cannot do

There is no join from a gauge table back to the railcar catalog. A `GaugeTable` carries no product
or model reference, and a catalog `Railcar` carries no reporting mark. If the caller asks "what
model is GBRX 700000", this API cannot answer it and neither can the catalog API.
