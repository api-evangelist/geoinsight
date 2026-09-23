---
name: Query a GeoInsight collection with CQL2
description: >-
  Search a collection's GeoParquet lake through the OGC API - Features surface using CQL2 attribute
  filters, a CRS84 bounding box and RFC 3339 temporal filters, and page correctly through the result set
  given that the API publishes no next-page links.
api: openapi/geoinsight-ogc-api-dggs-openapi.yml
base_url: https://api.geoinsight.ai
operations:
  - items_utoipa_doc
  - item_id_utoipa_doc
generated: '2026-08-20'
method: generated
source: openapi/geoinsight-ogc-api-dggs-openapi.yml
---

# Query a GeoInsight collection with CQL2

Not every collection has an items surface. A collection with no GeoParquet behind it returns **404** with
`message` "No parquet data in the lake for this collection". That is a normal, expected answer - use the
DGGS zone routes for raster-backed collections instead.

## Steps

1. **Inspect the physical layout before querying.**

       GET https://api.geoinsight.ai/collections/{collection_id}/items?f=json&limit=1

   Backing operation: `items_utoipa_doc`. With `f=json` this returns `RawItemsResponse` — the raw
   inspection view carrying `files[]` (each with `file` and `rows`), `columns[]`, `numberMatched`,
   `numberReturned`, `limit`, `offset` and WKB geometry previews. Read `columns[]` to learn the exact
   column names you can filter on. **You need this before you can write a CQL2 filter**, because the
   queryable column names are not in the OpenAPI.

2. **Run the query.**

       GET https://api.geoinsight.ai/collections/{collection_id}/items
             ?f=geojson
             &bbox=-123.5,49.0,-122.5,49.5
             &datetime=2024-01-01/2025-01-01
             &filter=population > 10000 AND name LIKE 'A%'
             &filter-lang=cql2-text
             &limit=1000

   - `f` — `geojson` (default, returns `FeatureCollectionResponse`), `json` (raw inspection view),
     `html`. Raster formats are rejected with **406** "Unsupported format (png, tiff)".
   - `bbox` — `min_x,min_y,max_x,max_y` in CRS84 lon/lat. Selects rows whose geometry intersects the box.
   - `datetime` — RFC 3339 instant or interval, including `start/..` and `../end`.
   - `filter` — a CQL2 expression over the parquet columns.
   - `filter-lang` — `cql2-text` (default) or `cql2-json`.
   - `properties` / `exclude-properties` — comma-separated column lists. `properties` is applied first.
     Geometry is retained for `geojson` and `geoparquet` unless you exclude it explicitly.

3. **Page through the results yourself.**

   `limit` defaults to 10 and caps at **1000**. There are **no `rel=next` or `rel=prev` links** in the
   response, so you must compute your own pages:

       read numberMatched from the first response
       for offset in range(0, numberMatched, limit):
           GET .../items?limit={limit}&offset={offset}&...   # repeat every filter parameter

   Repeat the full filter set on every page. `offset` is a 0-based row offset into the parquet.

4. **Fetch a single feature.**

       GET https://api.geoinsight.ai/collections/{collection_id}/items/{feature_id}?f=geojson

   Backing operation: `item_id_utoipa_doc`. **`feature_id` is a row offset, not a stable identifier.**
   It addresses position N in the parquet and can point at a different feature after the underlying data
   is rewritten. Never store it as a durable key or a foreign key. Out-of-range values return **404**.

## Error handling

| Status | Meaning | What to do |
|---|---|---|
| 400 | Invalid collection id or parameters | Re-check `collection_id` against `/collections`; validate `bbox`, `datetime` and CQL2 syntax |
| 404 | No parquet in the lake, or `feature_id` out of range | Use the DGGS zone routes instead, or stay inside `numberMatched` |
| 406 | Unsupported format | Use `geojson`, `json` or `html` on these routes |

All three arrive in the `ApiError` envelope: `{"code", "message", "hint", "redirect_location"}`, served as
`application/json`. This is **not** RFC 9457 problem+json — there is no `type` URI and no stable error
code, so branch on the HTTP status and read `hint` for the specifics.

## Rules

- Read-only. Nothing here changes state.
- No rate-limit headers, no `429`, no `Retry-After`. Pace yourself.
- Bind to method and path, not `operationId` — ids are not unique in this contract.
