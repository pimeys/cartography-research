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
| `MULTIPOINT`         | `Point[]`      |                                                  |
| `MULTILINESTRING`    | `LineString[]` |                                                  |
| `MULTIPOLYGON`       | `Polygon[]`    |                                                  |
| `GEOMETRYCOLLECTION` | `Geometry[]`   |                                                  |

### An example data model

```prisma
model geom {
  id  Int           @id
  g   Geometry      @srid(4326) @default(Point(1 1))
  p   Point         @srid(4326) @default(Point(2 1))
  l   LineString    @srid(4326) @default(LineString(1 1, 1 2)
  pg  Polygon       @srid(0)    @default(Polygon((1 1, 1 2, 1 1))
  mp  Point[]       @srid(4326) @default(MultiPoint(1 1, 1 2))
  mls LineString[]  @srid(4326) @default(MultiLineString((1 1, 1 2), (1 2, 2 3)))
  mpg Polygon[]     @srid(4326) @default(MultiPolygon((1 1, 1 2, 1 1), (1 1, 1 2, 1 1))
  gc  Geometry[]    @srid(4326) @default(GeometryCollection(Point(1 1), LineString(1 1, 1 2)))

  @@index([p], type: Spatial)
}
```

We have to validate the defaults. E.g. a Polygon has to connect
eventually.

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

The spatial index is only used if the indexed column is in a `WHERE` clause. For example the query in the 

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
