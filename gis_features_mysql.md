# MySQL GIS Features

Docs: https://dev.mysql.com/doc/refman/8.0/en/spatial-type-overview.html

Everything works as described from version 5.7 forwards. No extensions
needed, vanilla MySQL is enough.

## Types

| Native type          | Prisma type    | Description                                      |
|----------------------|----------------|--------------------------------------------------|
| `GEOMETRY`           | `Geometry`     | A superclass, can store any of the other values. |
| `POINT`              | `Point`        |                                                  |
| `LINESTRING`         | `LineString`   |                                                  |
| `POLYGON`            | `Polygon`      |                                                  |
| `MULTIPOINT`         | `Point[]`      |                                                  |
| `MULTILINESTRING`    | `LineString[]` |                                                  |
| `MULTIPOLYGON`       | `Polygon[]`    |                                                  |
| `GEOMETRYCOLLECTION` | `Geometry[]`   |                                                  |

### An example data model

```prisma
model geom {
  id  Int           @id
  g   Geometry      @srid(4326)
  p   Point         @srid(4326)
  l   LineString    @srid(4326)
  pg  Polygon       @srid(0)
  mp  Point[]       @srid(4326)
  mls LineString[]  @srid(4326)
  mpg Polygon[]     @srid(4326)
  gc  Geometry[]    @srid(4326)

  @@index([p], type: Spatial)
}
```

We have to validate the defaults. E.g. a Polygon has to connect
eventually.

None of the geometry columns can have a default value.

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

### Index

If needing to `CREATE SPATIAL INDEX` to a geometry column, the column
must be `NOT NULL` and SRID must be explicitly defined.

A `SPATIAL INDEX` can only be created to a single column, no compound
keys allowed.

We need a new (breaking) change to allow index type on MySQL:

```prisma
model geom {
  id Int   @id @default(autoincrement())
  p  Point

  @@index([p], type: Spatial)
}
```

Currently a spatial index is introspected as:

```prisma
model geom {
  id Int                  @id @default(autoincrement())
  p  Unsupported("point")

  @@index([location(length: 32)])
}
```

MySQL does not let you to define the length, but we introspect it anyhow...

#### Storing data

If a column has a SRID value defined, the insert has to define the
SRID per inserted value. This means it's quite possible we have to use
WKT values, because the only ST conversion function that works in
MySQL 5.7 and MariaDB requires a WKT representation.

An example insert for the table above would look like this:

```sql
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

#### Joining

A quite common use case is to do spatial joins. If we have the
following DDL:

```sql
CREATE TABLE USER (
    id int PRIMARY KEY AUTO_INCREMENT,
    location point NOT NULL SRID 4326
);

CREATE SPATIAL INDEX user_location ON user(location);

CREATE TABLE city (
    id int PRIMARY KEY AUTO_INCREMENT,
    area polygon NOT NULL SRID 4326
);

CREATE SPATIAL INDEX city_area ON city(area);
```

Our user has the following location: `x = 2, y = 2`:

```sql
INSERT INTO user (location)
VALUES (ST_PointFromText('POINT(2 2)', 4326));
```

A city, shaped a a rectangle, is surrounding the point from `1,1 ->
1,3 -> 3,3 -> 3,1 -> 1,1`:

```sql
INSERT INTO city (area)
VALUES (ST_PolygonFromText('POLYGON((1 1, 1 3, 3 3, 3 1, 1 1))', 4326));
```

To find the city where the user is currently, we could join the city
area:

```sql
SELECT user.id, city.id
FROM user
INNER JOIN city
ON ST_Contains(city.area, user.location);
```

#### Finding all users in radius

The following query will find all users in a radius of 1000 meters:

```sql
SELECT user.id, ST_Distance_Sphere(
    user.location,
    ST_PointFromText('POINT(4.001 4.001)', 4326))
FROM user
WHERE ST_Distance_Sphere(
    user.location,
    ST_PointFromText('POINT(4.001 4.001)', 4326)) < 1000;
```

The sphere default is the radius of the Earth (6370986 meters). The user
can of course change that by defining whatever radius as a parameter:

```sql
ST_Distance_Sphere(g1, g2 [, radius])
```

Where `g1` and `g2` are points.

#### Intersection

From our GIS poll, intersection:

```sql
SELECT ST_AsGeoJSON(ST_Intersection(
    ST_GeomFromText('LineString(1 1, 3 3)', 4326),
    ST_GeomFromText('LineString(1 3, 3 1)', 4326)
));
```

Or, find all roads that intersect the given line:

```sql
SELECT
    road.name,
    ST_Intersection(ST_GeomFromText('LineString(1 3, 3 1)', 4326), road.geom)
WHERE
    ST_Intersects(ST_GeomFromText('LineString(1 3, 3 1)', 4326), road.geom);
```

#### Transforms

From our GIS poll, transforming to a different SRID:

```sql
SELECT ST_AsText(ST_Transform(ST_PointFromText('POINT(4.001 4.001)', 4326), 4123));
```

### List of functions

| Requested | Name                                                                               | Description                                                   | Introduced |
|-----------|------------------------------------------------------------------------------------|---------------------------------------------------------------|------------|
| X         | `ST_Area()`                                                                        | Return Polygon or MultiPolygon area                           |            |
| X         | `ST_Buffer()`                                                                      | Return geometry of points within given distance from geometry |            |
| X         | `ST_Contains()`                                                                    | Whether one geometry contains another                         |            |
| X         | `ST_Distance()`                                                                    | The distance of one geometry from another                     |            |
| X         | `ST_Distance_Sphere()`                                                             | Minimum distance on earth between two geometries              |            |
| X         | `ST_Envelope()`                                                                    | Return MBR of geometry                                        |            |
| X         | `ST_GeoHash()`                                                                     | Produce a geohash value                                       |            |
| X         | `ST_GeomFromGeoJSON()`                                                             | Generate geometry from GeoJSON object                         |            |
| X         | `ST_Intersection()`                                                                | Return point set intersection of two geometries               |            |
| X         | `ST_Intersects()`                                                                  | Whether one geometry intersects another                       |            |
| X         | `ST_IsValid()`                                                                     | Whether a geometry is valid                                   |            |
| X         | `ST_MakeEnvelope()`                                                                | Rectangle around two points                                   |            |
| X         | `ST_Transform()`                                                                   | Transform coordinates of geometry                             | 8.0.13     |
| X         | `ST_Within()`                                                                      | Whether one geometry is within another                        |            |
|           | `ST_AsBinary()`, `ST_AsWKB()`                                                      | Convert from internal geometry format to WKB                  |            |
|           | `ST_AsGeoJSON()`                                                                   | Generate GeoJSON object from geometry                         |            |
|           | `ST_AsText(), ST_AsWKT()`                                                          | Convert from internal geometry format to WKT                  |            |
|           | `ST_Buffer_Strategy()`                                                             | Produce strategy option for ST_Buffer()                       |            |
|           | `ST_Centroid()`                                                                    | Return centroid as a point                                    |            |
|           | `ST_Collect()`                                                                     | Aggregate spatial values into collection                      | 8.0.24     |
|           | `ST_ConvexHull()`                                                                  | Return convex hull of geometry                                |            |
|           | `ST_Crosses()`                                                                     | Whether one geometry crosses another                          |            |
|           | `ST_Difference()`                                                                  | Return point set difference of two geometries                 |            |
|           | `ST_Dimension()`                                                                   | Dimension of geometry                                         |            |
|           | `ST_Disjoint()`                                                                    | Whether one geometry is disjoint from another                 |            |
|           | `ST_EndPoint()`                                                                    | End Point of LineString                                       |            |
|           | `ST_Equals()`                                                                      | Whether one geometry is equal to another                      |            |
|           | `ST_ExteriorRing()`                                                                | Return exterior ring of Polygon                               |            |
|           | `ST_FrechetDistance()`                                                             | The discrete Fréchet distance of one geometry from another    | 8.0.23     |
|           | `ST_GeomCollFromText()`, `ST_GeometryCollectionFromText()`, `ST_GeomCollFromTxt()` | Return geometry collection from WKT                           |            |
|           | `ST_GeomCollFromWKB()`, `ST_GeometryCollectionFromWKB()`                           | Return geometry collection from WKB                           |            |
|           | `ST_GeometryN()`                                                                   | Return N-th geometry from geometry collection                 |            |
|           | `ST_GeometryType()`                                                                | Return name of geometry type                                  |            |
|           | `ST_GeomFromText()`, `ST_GeometryFromText()`                                       | Return geometry from WKT                                      |            |
|           | `ST_GeomFromWKB()`, `ST_GeometryFromWKB()`                                         | Return geometry from WKB                                      |            |
|           | `ST_HausdorffDistance()`                                                           | The discrete Hausdorff distance of one geometry from another  | 8.0.23     |
|           | `ST_InteriorRingN()`                                                               | Return N-th interior ring of Polygon                          |            |
|           | `ST_IsClosed()`                                                                    | Whether a geometry is closed and simple                       |            |
|           | `ST_IsEmpty()`                                                                     | Whether a geometry is empty                                   |            |
|           | `ST_IsSimple()`                                                                    | Whether a geometry is simple                                  |            |
|           | `ST_LatFromGeoHash()`                                                              | Return latitude from geohash value                            |            |
|           | `ST_Latitude()`                                                                    | Return latitude of Point                                      | 8.0.12     |
|           | `ST_Length()`                                                                      | Return length of LineString                                   |            |
|           | `ST_LineFromText()`, `ST_LineStringFromText()`                                     | Construct LineString from WKT                                 |            |
|           | `ST_LineFromWKB()`, `ST_LineStringFromWKB()`                                       | Construct LineString from WKB                                 |            |
|           | `ST_LineInterpolatePoint()`                                                        | The point a given percentage along a LineString               | 8.0.24     |
|           | `ST_LineInterpolatePoints()`                                                       | The points a given percentage along a LineString              | 8.0.24     |
|           | `ST_LongFromGeoHash()`                                                             | Return longitude from geohash value                           |            |
|           | `ST_Longitude()`                                                                   | Return longitude of Point                                     | 8.0.12     |
|           | `ST_MLineFromText()`, `ST_MultiLineStringFromText()`                               | Construct MultiLineString from WKT                            |            |
|           | `ST_MLineFromWKB()`, `ST_MultiLineStringFromWKB()`                                 | Construct MultiLineString from WKB                            |            |
|           | `ST_MPointFromText()`, `ST_MultiPointFromText()`                                   | Construct MultiPoint from WKT                                 |            |
|           | `ST_MPointFromWKB()`, `ST_MultiPointFromWKB()`                                     | Construct MultiPoint from WKB                                 |            |
|           | `ST_MPolyFromText()`, `ST_MultiPolygonFromText()`                                  | Construct MultiPolygon from WKT                               |            |
|           | `ST_MPolyFromWKB()`, `ST_MultiPolygonFromWKB()`                                    | Construct MultiPolygon from WKB                               |            |
|           | `ST_NumGeometries()`                                                               | Return number of geometries in geometry collection            |            |
|           | `ST_NumInteriorRing()`, `ST_NumInteriorRings()`                                    | Return number of interior rings in Polygon                    |            |
|           | `ST_NumPoints()`                                                                   | Return number of points in LineString                         |            |
|           | `ST_Overlaps()`                                                                    | Whether one geometry overlaps another                         |            |
|           | `ST_PointAtDistance()`                                                             | The point a given distance along a LineString                 | 8.0.24     |
|           | `ST_PointFromGeoHash()`                                                            | Convert geohash value to POINT value                          |            |
|           | `ST_PointFromText()`                                                               | Construct Point from WKT                                      |            |
|           | `ST_PointFromWKB()`                                                                | Construct Point from WKB                                      |            |
|           | `ST_PointN()`                                                                      | Return N-th point from LineString                             |            |
|           | `ST_PolyFromText()`, `ST_PolygonFromText()`                                        | Construct Polygon from WKT                                    |            |
|           | `ST_PolyFromWKB()`, `ST_PolygonFromWKB()`                                          | Construct Polygon from WKB                                    |            |
|           | `ST_Simplify()`                                                                    | Return simplified geometry                                    |            |
|           | `ST_SRID()`                                                                        | Return spatial reference system ID for geometry               |            |
|           | `ST_StartPoint()`                                                                  | Start Point of LineString                                     |            |
|           | `ST_SwapXY()`                                                                      | Return argument with X/Y coordinates swapped                  |            |
|           | `ST_SymDifference()`                                                               | Return point set symmetric difference of two geometries       |            |
|           | `ST_Touches()`                                                                     | Whether one geometry touches another                          |            |
|           | `ST_Union()`                                                                       | Return point set union of two geometries                      |            |
|           | `ST_Validate()`                                                                    | Return validated geometry                                     |            |
|           | `ST_X()`                                                                           | Return X coordinate of Point                                  |            |
|           | `ST_Y()`                                                                           | Return Y coordinate of Point                                  |            |
