---
name: Discover GeoInsight collections and grids
description: >-
  Find out what data GeoInsight actually holds and which Discrete Global Grid Reference Systems it can
  resolve that data onto, before attempting any zone query. Always run this first - neither collection_id
  nor dggrs_id is enumerated in the OpenAPI, so the contract alone will not tell you what is valid.
api: openapi/geoinsight-ogc-api-dggs-openapi.yml
base_url: https://api.geoinsight.ai
operations:
  - root_handler
  - utoipa_doc
  - collection_id_utoipa_doc
  - dggrs_id_utoipa_doc
generated: '2026-08-20'
method: generated
source: openapi/geoinsight-ogc-api-dggs-openapi.yml
---

# Discover GeoInsight collections and grids

## Why this comes first

`collection_id` and `dggrs_id` are typed as bare strings in the contract with no `enum`. There is no way
to learn the valid values except by asking the service. Guessing an id costs you a 400 or a 404.

## Steps

1. **Fetch the service root.**

       GET https://api.geoinsight.ai/

   Backing operation: `root_handler` (`GET /`). Returns `RootResponse` with `title`, `description`,
   `attribution` and `links[]`. Use it to confirm the service is up. Do **not** use `HEAD` for this -
   `HEAD /` returns 404 while `GET /` returns 200.

   Ignore the `conformance` links in this document. They point at `www.example.com` and are unreplaced
   placeholders, and the real `/conformance` path returns HTTP 500.

2. **List the collections.**

       GET https://api.geoinsight.ai/collections?f=json

   Backing operation: the `GET /collections` operation (`CollectionsResponse`). Bind by path, not by
   `operationId` - its id `utoipa_doc` is shared with `GET /dggs`.

   Each entry carries `id`, `title`, `description`, `item_type`, `attribution`, `crs`, `extent` and
   `links[]`. Read `extent.spatial.bbox` and `extent.temporal.interval` and check that your area and
   period of interest fall inside them before going further. Most of the imagery collections are single
   scenes covering one tile, not global coverage.

3. **Optionally fetch one collection.**

       GET https://api.geoinsight.ai/collections/{collection_id}?f=json

   Backing operation: `GET /collections/{collection_id}` (`CollectionIdResponse`). Bind by path - the id
   `collection_id_utoipa_doc` is shared with `GET /collections/{collection_id}/dggs`.

4. **List the grids.**

       GET https://api.geoinsight.ai/dggs?f=json

   Backing operation: `GET /dggs` (`DggsResponse`). Observed values on 2026-08-20: `H3`,
   `ISEA3HDGGRID`, `ISEA3HDGGAL`, `IGEO7`, `IVEA3H`. Note that `title` and `uri` come back as empty
   strings in the registry listing, so the `id` is the only usable field.

5. **Fetch the grid definition you intend to use.**

       GET https://api.geoinsight.ai/dggs/{dggrs_id}?f=json

   Backing operation: `GET /dggs/{dggrs_id}` (`DggrsIdResponse`). Bind by path - `dggrs_id_utoipa_doc` is
   shared with `GET /dggs/{dggrs_id}/zones`.

   Read `default_depth`, `max_refinement_level` and `max_relative_depth`. For `H3` these are 4, 16 and 6.
   These three numbers bound every zone query you can make: `zone-depth` cannot exceed
   `max_relative_depth`, and the zone ids you supply must be valid at or below `max_refinement_level`.

6. **Check which grids a specific collection supports.**

       GET https://api.geoinsight.ai/collections/{collection_id}/dggs?f=json

   Not every collection is resolvable onto every grid. Do this before assuming H3 works for your
   collection.

## Rules

- Read-only. Every operation here is a `GET`; nothing you do changes state.
- No rate-limit headers exist. Space these calls out; the service cannot tell you to slow down.
- On any error read the `hint` field of the `ApiError` body - it names the offending parameter.
- Do not retry a 5xx from this API without reading `hint` first. Client input errors surface as 500 here.
