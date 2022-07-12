# CockroachDB GIS Features

Docs: https://www.cockroachlabs.com/docs/v22.1/spatial-features

Everything as described from version 20.2.19 forwards. No extensions
needed, vanilla CockroachDB is enough. Should implement most of the
features of the PostGIS extension.

## Types

The database will allow two different types:

| Native type | Prisma type | Description                                                 |
|-------------|-------------|-------------------------------------------------------------|
| `GEOMETRY`  | `Geometry`  | A flat, infinite cartesian plane. Can store all subclasses. |
| `GEOGRAPHY` | `Geometry`  | Earth. Can store all subclasses.                            |

The following types can be stored to the geometry/geography columns:

| Type                 | GeoJSON Support |
|----------------------|-----------------|
| `Point`              | yes             |
| `LineString`         | yes             |
| `Polygon`            | yes             |
| `MultiPoint`         | yes             |
| `MultiLineString`    | yes             |
| `MultiPolygon`       | yes             |
| `GeometryCollection` | yes             |

Types can also be holding data for:

- A third dimension coordinate Z (POINTZ et.al.).
- A measure coordinate M (POINTM et.al.).
- Both a third dimension and a measure coordinate (POINTZM et.al.).

### An example data model

```prisma
model geom {
  id  Int      @id
  g   Geometry @db.Geography @default(Point(..))
  l   Geometry @db.Geography @default(LineString(..)
  pg  Geometry @db.Geography @default(Polygon((..))
  mp  Geometry @db.Geography @default(MultiPoint(..))
  mls Geometry @db.Geography @default(MultiLineString((..), (..)))
  mpg Geometry @db.Geography @default(MultiPolygon((..), (..))
}
```

We have to validate the defaults. E.g. a Polygon has to connect
eventually.

### Migration

```sql
CREATE TABLE geom (
  id int IDENTITY PRIMARY KEY ,
  geog geography,
  geom geometry
);
```

The SRID is defined in data insertion.

CDB allows setting a default value to geography and geomatry columns:

```sql
CREATE TABLE test (
    id int PRIMARY KEY,
    location geography DEFAULT ST_GeogFromText('SRID=4326;POINT(1 1)'),
    position geometry DEFAULT ST_GeomFromText('SRID=4326;LINESTRING(1 1, 1 2)')
);
```

See the different function depending on the underlying column type.

### Index

The CDB spatial index of choice is `Gin`:

```sql
CREATE INDEX ON test
USING GIN (location)
```

```prisma
model test {
  id       Int      @id
  location Geometry @db.Geography
  position Geometry

  @@index([location], type: Gin)
}
```

The geographical column must be the last column in the index. There
can be more than one column defined, e.g. this is possible:

```prisma
model test {
  id       Int      @id
  location Geometry @db.Geography
  position Geometry
  value    String

  @@index([string, location], type: Gin)
}
```

But this not:

```prisma
model test {
  id       Int      @id
  location Geometry @db.Geography
  position Geometry

  @@index([position, location], type: Gin)
}
```

### Storing data

The SRID value is defined on insertion:

```sql
INSERT INTO test (id, location, position)
VALUES (1,
        st_geogfromtext('SRID=4326;POINT(2 3)'),
        st_geomfromtext('SRID=4326;LINESTRING(1 1, 1 3)'));
```

Again, pay attention to use the correct functions to the corresponding
column type.

### Selecting data

CDB supports GeoJSON results:

```sql
SELECT
    st_asgeojson(location),
    st_srid(location),
    st_asgeojson(position),
    st_srid(position)
FROM test;
```

### List of functions

Too many to list here. Everything people might ever want for GIS are available in CDB.

You can find all `ST_` functions from the documentation:

https://www.cockroachlabs.com/docs/stable/functions-and-operators.html
