# MySQL GIS Features

Docs: https://dev.mysql.com/doc/refman/8.0/en/spatial-type-overview.html

Everything works as describer from version 5.7 forwards. No extensions
needed, vanilla MySQL is enough.

## Types

| Native type          | Prisma type    | Description                                      |
|----------------------|----------------|--------------------------------------------------|
| `GEOMETRY`           | `Geometry`     | A superclass, can store any of the other values. |
| `POINT`              | `Point`        |                                                  |
| `LINESTRING`         | `LineString`   |                                                  |
| `POLYGON`            | `Polygon`      |                                                  |
| `MULTIPOINT`         | `MultiPoint`   |                                                  |
| `MULTILINESTRING`    | `LineString[]` |                                                  |
| `MULTIPOLYGON`       | `Polygon[]`    |                                                  |
| `GEOMETRYCOLLECTION` | `Geometry[]`   |                                                  |

### Migration

```sql
CREATE TABLE geom
(
    id  INT PRIMARY KEY AUTO_INCREMENT,
    g   GEOMETRY           NOT NULL SRID 4326,
    p   POINT              NOT NULL SRID 4326,
    l   LINESTRING         NOT NULL SRID 4326,
    pg  POLYGON            NOT NULL SRID 4326,
    mp  MULTIPOINT         NOT NULL SRID 4326,
    mls MULTILINESTRING    NOT NULL SRID 4326,
    mpg MULTIPOLYGON       NOT NULL SRID 4326,
    gc  GEOMETRYCOLLECTION NOT NULL SRID 4326
);
```

Default SRID is 0, which is an infinite flat Cartesian plain. 4326 is
the same as GPS.

#### Index

If needing to `CREATE SPATIAL INDEX` to a geometry column, the column
must be `NOT NULL` and SRID must be explicitly defined.

A `SPATIAL INDEX` can only be created to a single column, no compound
keys allowed.

#### Storing data

If a column has a SRID value defined, the insert has to define the
SRID per inserted value. This means it's quite possible we have to use
WKT values, because the only ST conversion function that works in
MySQL 5.7 and MariaDB requires a WKT representation.

An example insert for the table above would look like this:

```prisma
INSERT INTO geom (g, p, l, pg, mp, mls, mpg, gc)
VALUES (
    ST_PointFromText('POINT(15 20)', 4326),

    ST_PointFromText('POINT(20 25)', 4326),

    ST_LineFromText('LINESTRING(15 20, 20 25)', 4326),

    ST_PolygonFromText(
            'POLYGON((0 0,10 0,10 10,0 10,0 0),(5 5,7 5,7 7,5 7, 5 5))',
            4326),

    ST_MultiPointFromText('MULTIPOINT(15 20, 40 30)', 4326),

    ST_MultiLineStringFromText(
            'MULTILINESTRING((15 20, 20 25), (20 25, 30 35))',
            4326),

    ST_MultiPolygonFromText(
            'MULTIPOLYGON(((0 0,10 0,10 10,0 10,0 0)),((5 5,7 5,7 7,5 7, 5 5)))',
            4326),

    ST_GeomCollFromtext(
            'GEOMETRYCOLLECTION(POINT(10 10), POINT(30 30), LINESTRING(15 15, 20 20))',
            4326)
);
```

#### Selecting data

The data by default comes back as binary. MySQL provides a function
for conversion to a geojson format, that can be mapped to
corresponding types in Rust and TypeScript.

An example select to return the inserted data:

```sql
SELECT ST_AsGeoJSON(g),
       ST_AsGeoJSON(p),
       ST_AsGeoJSON(l),
       ST_AsGeoJSON(pg),
       ST_AsGeoJSON(mp),
       ST_AsGeoJSON(mls),
       ST_AsGeoJSON(mpg),
       ST_AsGeoJSON(gc)
FROM geom;
```
