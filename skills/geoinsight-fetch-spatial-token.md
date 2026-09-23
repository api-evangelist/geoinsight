---
name: Fetch a GeoInsight Spatial Token for a zone
description: >-
  Retrieve one collection's values aggregated onto one DGGS zone at a chosen relative depth - the core
  GeoInsight product. Covers raster collections resolved through COG statistics and STAC search
  strategies, and the format and property-selection parameters that control response size.
api: openapi/geoinsight-ogc-api-dggs-openapi.yml
base_url: https://api.geoinsight.ai
operations:
  - collection_id_dggrs_id_zone_id_data_utoipa_doc
  - collection_id_dggrs_id_zone_id_utoipa_doc
  - dggrs_id_zone_id_utoipa_doc
generated: '2026-08-20'
method: generated
source: openapi/geoinsight-ogc-api-dggs-openapi.yml
---

# Fetch a GeoInsight Spatial Token for a zone

Prerequisite: run **Discover GeoInsight collections and grids** first to get a valid `collection_id`,
`dggrs_id` and the grid's `max_relative_depth`.

## Steps

1. **Resolve your area of interest to a zone id.**

   Zone ids are the grid's own native indices - an H3 cell index for `H3`. Compute it client-side with the
   grid's own library (`h3` for H3), or inspect an existing zone:

       GET https://api.geoinsight.ai/dggs/{dggrs_id}/zones/{zone_id}?f=json

   Backing operation: `dggrs_id_zone_id_utoipa_doc` (`GET /dggs/{dggrs_id}/zones/{zone_id}`). Returns
   `ZoneIdResponse` with `level`, `geometry`, `centroid`, `bbox`, `crs`, `shapeType`, `areaMetersSquare`,
   `volumeMetersCube`, `temporalInterval`, `temporalDurationSeconds` and `statistics`.

   Add `?refined-geometry=true` to densify the zone polygon by interpolating points along its edges -
   relevant for `geo+json` and `geoparquet` output.

   A malformed zone id returns HTTP **500**, not 422, with the grid library's own diagnostic in `hint`
   (for example `H3o error: Invalid H3 zone ID abc`). Treat that 500 as a permanent client error and do
   not retry it.

2. **Fetch the zone data.**

       GET https://api.geoinsight.ai/collections/{collection_id}/dggs/{dggrs_id}/zones/{zone_id}/data?f=json&zone-depth=1

   Backing operation: `collection_id_dggrs_id_zone_id_data_utoipa_doc`. This is the richest operation in
   the contract - 18 parameters. Returns `DataResponse` with `dggrs`, `zoneId`, `depths`, `defaultDepth`,
   `dimensions` and `schema`.

   Read `schema.properties` on the response to learn what queryables the collection actually exposes;
   each `SchemaProperty` carries `x-ogc-definition`, the OGC definition URI that gives the property its
   meaning.

3. **Control the cost of the request.**

   - `zone-depth` — relative depth of child zones to aggregate. A single integer (`1`) or a
     comma-separated list (`1,2`). Defaults to the DGGRS `default_depth`. **This is the main cost dial.**
     Each additional depth multiplies the child-zone count by the grid's aperture, so raise it one step
     at a time.
   - `properties` — comma-separated queryables to include (e.g. `ndvi,evi`). Returns everything when
     omitted. Always set it once you know what you need.
   - `exclude-properties` — applied *after* `properties`.
   - `f` — `json`, `dggs+json`, `geo+json`, `html`, `parquet`, `geoparquet`. Omitting `f` falls back to
     `Accept` header content negotiation. Use `geoparquet` for bulk, `dggs+json` for the native token
     form.
   - `datetime` — RFC 3339 instant (`2024-01-01T00:00:00Z`) or interval (`2024-01-01/2025-01-01`), with
     open-ended forms supported.

4. **For raster-backed collections, tune the COG read.**

   - `cog-stat` — `count`, `valid`, `min`, `max`, `mean`, `median`. Comma-separated. Defaults to `mean`.
   - `cog-band` — comma-separated 1-based band indices. Defaults to `1`.
   - `cog-no-data-value` — override the no-data sentinel; matching pixels are excluded from statistics.
   - `cog-sampling-factor` — sampling density multiplier relative to zone count. Defaults to `8.0`.
     Higher is more accurate and more expensive.
   - `cog-all-touched` — include pixels only touched by the zone boundary. Defaults to `false`.

5. **For STAC-backed collections, choose a search strategy.**

   - `stac-search-strategy` — `single` (default), `latest`, or `mosaic`. `latest` takes the newest item
     with no intersection check; `single` takes the best single intersecting item; `mosaic` composites.
   - `intersection_percent` — 0–100. For `single`, the minimum the chosen item alone must cover. `0`
     disables the check.
   - `cloud_cover_percent` — 0–100 maximum `eo:cloud_cover`. Omit to disable. Overrides the
     collection-level default.

   Note the naming inconsistency: `intersection_percent` and `cloud_cover_percent` are snake_case while
   every other parameter on this operation is kebab-case. Send them exactly as written.

6. **Query an arbitrary COG without a collection.**

       GET https://api.geoinsight.ai/dggs/{dggrs_id}/zones/{zone_id}/data?cog-url={url}&cog-stat=mean,min,max

   `cog-url` bypasses collection lookup entirely. Use it to zonally summarise your own raster on
   GeoInsight's grid.

## Rules

- Read-only and safe. No idempotency key, no reversal, nothing to undo.
- Never call the zone **listing** routes (`.../zones` without a `{zone_id}`) unscoped on a large
  collection - they accept no `limit` parameter and an unbounded listing has been observed returning an
  nginx 502. Always address a specific zone.
- No `429` and no `Retry-After` exist. Back off on your own schedule.
- On `422` read `hint`; it names the missing or conflicting parameter (the contract's own example is
  "Parameter `parent-zone-level` is required when `parent-zone` is provided").
