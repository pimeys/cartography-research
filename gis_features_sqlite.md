# SQLite GIS Features

Docs: https://www.gaia-gis.it/gaia-sins/index.html

GIS features can be added through an extension called SpatiaLite. It
can be provided in a few ways to an application using SQLite:

- As a compiled library. The extension is loaded by giving a path to
  the SpatiaLite so/dylib/dll file.
- The whole application can be compiled against SpatiaLite and
  SQLite. All of the library is then statically linked to the binary
  file.

The first option is possible already with the current engines. It is a
bit tricky from the developer experience point of view: the connection
string must provide the path to the library, and the user must know
the location and be able to install the correct version to their
operating system.

The second option requires more work from Prisma. There are no existing
bindigs that enable compilation of all the needed libraries with the
engines. It is not also very tricky to build, but there needs to be
either a fork of rusqlite, or we need to provide a pull request for
them to use SpatiaLite instead of SQLite.

The license differs from SQLite. They provide three different
licenses: Mozilla Public License, GNU General Public License v2.0 and
GNU Lesser General Public License v2.1. If we'd statically link against
SpatiaLite, we have to be sure we respect the license terms and that
our existing code doesn't change from Apache 2.0 to the SpatiaLite
license.

If we'd compile SpatiaLite directly to engine binaries, it would add
about eight megabytes to all engine binaries.

## Types

Basically all standard GIS types are available, and SpatiaLite should
support the same things as PostGIS does currently.

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

Additionally SpatiaLite supports geometries in three dimensions.
These types are marked as `Z` in the end, e.g. a point in three
dimensions would be of type `POINTZ`.

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

First we create the table without any GIS columns:

```sql
CREATE TABLE geom
(
    id INTEGER PRIMARY KEY
)
```

The GIS columns are added calling a special function:

```sql
SELECT AddGeometryColumn('test', 'pt', 4326, 'POINT', 'XY', 0);
```

Function parameters explained:

- `'test'` is the name of the table we modify
- `'pt'` is the name of the column added
- `4326` is the SRID of the column
- `POINT` is the GIS type
- `XY` sets 2d geometry
- `0` marks can the value be `NULL`. `0` is nullable, `1` is not.

Dropping columns in SpatiaLite is tricky, it's not yet supported as it
is with the latest SQLite versions:

```sql
-- The next statement turns a geometry column into a normal BLOB:
spatialite> select DiscardGeometryColumn('test', 'pt');
1
spatialite> alter table test drop column pt;
Error: error in trigger ISO_metadata_reference_row_id_value_insert: no such column: rowid
```

### Index

The spatial index is created with a special function:

```sql
SELECT CreateSpatialIndex('geom, 'pt');
```

Notice how we cannot specify a name for the spatial index. Dropping is
also a bit more complex operation:

```sql
SELECT DisableSpatialIndex('geom', 'pt');
```

The operation removes any triggers that would update the index when
adding/removing/modifying the data. The indexed data is stored into
four tables that are still available in the database:

```sql
idx_geom_pt
idx_geom_pt_node
idx_test_ls_parent
idx_test_ls_rowid
```

These tables must be manually removed after disabling the index triggers.

#### Storing data

Inserts work using the standard WKT functions as with the other databases:

```sql
INSERT INTO geom (id, pt)
VALUES (
    1,
    GeomFromText('Point(1 1)', 4326)
)
```

The SRID must be the same as in the DDL definition, or it will lead to
a runtime error.

#### Selecting data

All typical conversions work, for example to GeoJSON:

```sql
SELECT AsGeoJSON(pt) pt, ST_SRID(pt) id
FROM test;

{"type":"Point","coordinates":[1,1]}|4326
```

### List of functions

All standard `ST_` functions are available.
