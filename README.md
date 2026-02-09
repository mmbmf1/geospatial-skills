# Geospatial Skills

PostGIS and GeoJSON patterns as agent skills.

## Install

Install all skills:

    npx skills add https://github.com/mmbmf1/geospatial-skills

Install a single skill:

    npx skills add https://github.com/mmbmf1/geospatial-skills --skill geojson-points

## Skills

### GeoJSON

- geojson-points — lat/lng rows → GeoJSON
- geojson-wkt — WKT strings → GeoJSON
- geojson-postgis — PostGIS geometry rows → GeoJSON

### PostGIS

- postgis-transform — SRID handling with ST_Transform

## License

MIT
