---
name: postgis-intersects
description: Spatial joins and point-in-polygon with ST_Intersects, including index-friendly bbox prefilter patterns.
---

# ST_Intersects (Spatial Join / Point-in-Polygon)

Use this skill when you need to test whether two geometries overlap in any way:

- point-in-polygon lookups (county, service area, rate center)
- spatial joins between lines/polygons
- “does this feature touch/overlap this other feature?”

## Core rules

- Both geometries must be in the **same SRID**
- Prefer an index-friendly bbox prefilter (`&&`) before `ST_Intersects`
- Ensure GiST indexes exist on geometry columns used in joins

## Canonical spatial join (index-friendly)

    SELECT a.*
    FROM a
    JOIN b
      ON a.geom && b.geom
     AND ST_Intersects(a.geom, b.geom);

## Point-in-polygon (point against polygons)

    SELECT p.id, poly.*
    FROM points p
    JOIN polygons poly
      ON p.geom && poly.geom
     AND ST_Intersects(p.geom, poly.geom);

## Different SRIDs (transform one side)

Transform the smaller dataset (or the point) to match the other side’s SRID.

    SELECT p.id, poly.*
    FROM points p
    JOIN polygons poly
      ON ST_Intersects(
        ST_Transform(p.geom, ST_SRID(poly.geom)),
        poly.geom
      );

If you do this frequently, consider storing points in the same SRID as your polygon reference layer.

## Bounding box filter for map requests (envelope)

When the client provides a bbox, filter polygons/points before intersects.

    -- params: :minLng, :minLat, :maxLng, :maxLat in EPSG:4326
    WITH bbox AS (
      SELECT ST_MakeEnvelope(:minLng, :minLat, :maxLng, :maxLat, 4326) AS geom
    )
    SELECT t.*
    FROM my_table t, bbox
    WHERE t.geom && bbox.geom
      AND ST_Intersects(t.geom, bbox.geom);

## Index requirements (don’t skip this)

Ensure GiST indexes on geometry columns:

    CREATE INDEX IF NOT EXISTS a_geom_gix ON a USING gist (geom);
    CREATE INDEX IF NOT EXISTS b_geom_gix ON b USING gist (geom);

## Common mistakes

- Calling `ST_Intersects` without SRID alignment
- Omitting `&&` and forcing full-table geometry checks
- Transforming both sides in the join condition (kills index usage)
- Using `ST_Intersects` when you actually want `ST_DWithin` (distance)

## Summary

- Use `a.geom && b.geom` + `ST_Intersects(a.geom, b.geom)` for joins
- Align SRIDs (transform one side if needed)
- Index geometry columns with GiST
