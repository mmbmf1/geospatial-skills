---
name: postgis-extract-xy
description: Extract longitude and latitude from PostGIS geometries using ST_X and ST_Y safely.
---

# Extract X / Y from Geometry (ST_X, ST_Y)

Use this skill when you need to **extract numeric longitude/latitude values** from a PostGIS geometry.

This is common when:

- returning tabular data alongside geometry
- exporting points to CSV
- debugging coordinate issues
- integrating with non-GIS systems

## Core rules

- `ST_X` and `ST_Y` **only work on POINT geometries**
- Units and meaning depend on the geometry’s SRID
- For web use, extract coordinates **after transforming to EPSG:4326**

## Canonical patterns

## Extract from point geometry (already in EPSG:4326)

    SELECT
      ST_X(geom) AS lng,
      ST_Y(geom) AS lat
    FROM points
    WHERE geom IS NOT NULL;

- `ST_X` → longitude
- `ST_Y` → latitude

## Transform before extracting (recommended)

If geometry is stored in a projected or client SRID:

    SELECT
      ST_X(ST_Transform(geom, 4326)) AS lng,
      ST_Y(ST_Transform(geom, 4326)) AS lat
    FROM points;

Always extract in 4326 when values are meant for humans or maps.

## Non-point geometry (use centroid)

For lines or polygons, extract from the centroid:

    SELECT
      ST_X(ST_Centroid(geom)) AS lng,
      ST_Y(ST_Centroid(geom)) AS lat
    FROM features;

Be aware:

- Centroid may fall outside the polygon for concave shapes
- For guaranteed interior points, use `ST_PointOnSurface`

## Guaranteed interior point (polygons)

    SELECT
      ST_X(ST_PointOnSurface(geom)) AS lng,
      ST_Y(ST_PointOnSurface(geom)) AS lat
    FROM polygons;

## JSON-friendly extraction

Useful when building API responses:

    SELECT
      jsonb_build_object(
        'lng', ST_X(ST_Transform(geom, 4326)),
        'lat', ST_Y(ST_Transform(geom, 4326))
      )
    FROM points;

## Common mistakes

- Calling `ST_X` / `ST_Y` on non-point geometries
- Forgetting to transform before extraction
- Swapping latitude and longitude
- Assuming projected coordinates are degrees

## Summary

- Use `ST_X` for longitude, `ST_Y` for latitude
- Transform to EPSG:4326 before extracting for web use
- Use centroids or point-on-surface for non-point geometry
