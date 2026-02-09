---
name: postgis-transform
description: Reproject PostGIS geometry using ST_Transform with correct SRID handling and safe usage patterns.
---

# Reproject Geometry with ST_Transform

Use this skill when you need to **reproject geometry between coordinate systems (SRIDs)** in PostGIS.

This documents the **canonical, safe patterns** for using `ST_Transform`.

## When to use

- You need planar distance, buffering, or area calculations
- You are comparing geometries from different SRIDs
- You are preparing geometry for web maps (EPSG:4326)

## Core rules

- Geometry must have a valid SRID before transforming
- Use projected SRIDs for distance/buffer math
- Serve web maps in EPSG:4326
- Avoid transforming large tables inside WHERE clauses

## Basic transform

    SELECT
      ST_Transform(geom, 4326)
    FROM my_table;

## Assign SRID before transforming (required if SRID = 0)

    SELECT
      ST_Transform(
        ST_SetSRID(geom, 3857),
        4326
      )
    FROM my_table;

Never call `ST_Transform` on geometry with SRID = 0.

## Transform for distance or buffer calculations

Store geometry in 4326, transform only for math:

    SELECT
      ST_Distance(
        ST_Transform(a.geom, client_srid),
        ST_Transform(b.geom, client_srid)
      )
    FROM a
    JOIN b ON ...;

Do not compute distance or buffers in EPSG:4326.

## Transform only one side (performance)

When joining datasets, transform the **smaller** side:

    SELECT a.*
    FROM large_table a
    JOIN small_table b
      ON ST_Intersects(
        a.geom,
        ST_Transform(b.geom, ST_SRID(a.geom))
      );

Transforming both sides disables spatial index usage.

## Serving geometry to web clients

Always serve GeoJSON in EPSG:4326:

    ST_AsGeoJSON(ST_Transform(geom, 4326))::jsonb

## Common mistakes

- Transforming geometry without checking SRID
- Transforming both sides of a spatial join
- Buffering or measuring distance in EPSG:4326
- Serving projected coordinates directly to browsers

## Summary

- Store geometry consistently
- Transform explicitly
- Use projected SRIDs for math
- Use EPSG:4326 for maps
