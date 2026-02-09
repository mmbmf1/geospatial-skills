---
name: postgis-buffer
description: Create buffer geometries with ST_Buffer, with correct units and safe patterns for geometry vs geography.
---

# ST_Buffer (Buffers / Proximity Shapes)

Use this skill when you need to create a polygon “buffer” around a geometry (point/line/polygon), typically for:

- proximity searches
- topology/contiguity validation
- “within X distance” visualization
- making thin lines easier to intersect (line → corridor)

## Core rules

- Buffer units depend on the input type:
  - `geometry` buffers use the SRID’s coordinate units (degrees/meters/feet)
  - `geography` buffers use **meters**
- Do not buffer in EPSG:4326 degrees unless you _intend_ degrees
- Prefer buffering in a **projected** CRS or using **geography**

## Canonical patterns

## Buffer in meters (recommended, easiest): geography

Good default when your geometry is in EPSG:4326 and you want meters.

    SELECT
      ST_Buffer(geom::geography, 250)::geometry AS geom_buffer
    FROM my_table;

## Buffer in projected CRS (recommended for large jobs)

Use a projected SRID for consistent planar units.

    SELECT
      ST_Transform(
        ST_Buffer(ST_Transform(geom, 3857), 250),
        4326
      ) AS geom_buffer_4326
    FROM my_table;

Use your client/project SRID instead of 3857 when you have one.

## Buffer and union (common validation pattern)

Useful for “contiguity” checks or merging segment corridors.

    SELECT
      ST_Union(ST_Buffer(geom, 0.5)) AS buffered_union
    FROM line_segments
    WHERE id = $1;

## Buffer for safer intersects (line corridor)

Turn a line into a corridor to catch near-misses.

    SELECT
      ST_Buffer(line.geom, 0.5) AS line_corridor
    FROM lines line;

Then intersect points against `line_corridor`.

## Notes on negative buffers

Negative buffers shrink polygons (can produce empties):

    SELECT ST_Buffer(poly.geom, -5)
    FROM polygons poly;

Handle empty results if you use negative buffers.

## Common mistakes

- Buffering in EPSG:4326 (degrees) when you intended meters/feet
- Assuming `ST_Buffer(geometry, x)` is meters (it is not)
- Not transforming to a projected SRID for planar work
- Buffering huge datasets without limits (expensive)

## Summary

- Use `geom::geography` when you want meters
- Use projected SRIDs for consistent planar buffering
- Remember geometry units come from the SRID
