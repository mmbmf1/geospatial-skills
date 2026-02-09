---
name: postgis-nearest
description: Find nearest features efficiently using PostGIS KNN (<->) and distance ordering (with SRID/unit guidance).
---

# Nearest Feature (KNN / `<->`)

Use this skill when you need the **nearest** feature to a point/geometry (e.g., nearest feeder, nearest segment, nearest hub).

PostGIS supports fast nearest-neighbor searches using the **KNN operator** `<->` when a GiST index exists.

## When to use

- “Find the nearest X to this point”
- “Rank candidates by proximity”
- “Pick the closest feature per input row”

## Core rules

- KNN `<->` works best with **GiST** indexes on geometry columns
- `<->` orders by distance in the geometry’s coordinate system (SRID units)
- For meaningful distances, use a projected SRID or `geography` (meters)
- Keep queries index-friendly: avoid wrapping the indexed column in transforms inside ORDER BY when possible

## Index requirement

    CREATE INDEX IF NOT EXISTS features_geom_gix ON features USING gist (geom);

Without this, KNN will not be fast.

## Canonical pattern: nearest feature to a single point

    -- point is EPSG:4326 here
    WITH p AS (
      SELECT ST_SetSRID(ST_MakePoint($1, $2), 4326) AS geom
    )
    SELECT f.*
    FROM features f, p
    ORDER BY f.geom <-> p.geom
    LIMIT 1;

## Nearest N features

    WITH p AS (
      SELECT ST_SetSRID(ST_MakePoint($1, $2), 4326) AS geom
    )
    SELECT f.*
    FROM features f, p
    ORDER BY f.geom <-> p.geom
    LIMIT 10;

## Add an exact distance (optional)

Compute the distance separately (don’t replace the KNN order):

    WITH p AS (
      SELECT ST_SetSRID(ST_MakePoint($1, $2), 4326) AS geom
    )
    SELECT
      f.*,
      ST_Distance(f.geom::geography, p.geom::geography) AS distance_m
    FROM features f, p
    ORDER BY f.geom <-> p.geom
    LIMIT 1;

Here:

- ordering stays fast via KNN
- distance is computed in meters via geography

## Recommended: constrain candidates with ST_DWithin

For large tables, reduce work and avoid weird global matches:

    WITH p AS (
      SELECT ST_SetSRID(ST_MakePoint($1, $2), 4326) AS geom
    )
    SELECT f.*
    FROM features f, p
    WHERE ST_DWithin(f.geom::geography, p.geom::geography, 5000) -- 5km
    ORDER BY f.geom <-> p.geom
    LIMIT 1;

## Different SRIDs

If your features are stored in a projected/client SRID, build the point in that SRID (or transform once):

    WITH p AS (
      SELECT ST_Transform(
        ST_SetSRID(ST_MakePoint($1, $2), 4326),
        $3  -- client_srid
      ) AS geom
    )
    SELECT f.*
    FROM features f, p
    ORDER BY f.geom <-> p.geom
    LIMIT 1;

Prefer storing and indexing `features.geom` in the SRID you query most often.

## Common mistakes

- No GiST index (query is slow)
- Using `<->` on EPSG:4326 and assuming the distance is meters
- Transforming `f.geom` inside ORDER BY (can kill index usage)
- Not bounding the search (use ST_DWithin when appropriate)

## Summary

- Use `ORDER BY geom <-> point LIMIT 1` for nearest neighbor
- Ensure GiST index on `geom`
- Use geography or projected SRID for real-world distance values
- Add ST_DWithin to keep searches local and fast
