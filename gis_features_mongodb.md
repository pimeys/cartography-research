# MongoDB GIS Features

Docs: https://www.mongodb.com/docs/manual/geospatial-queries/

Everything works as described from version 4.2 forwards. No extensions
needed, vanilla MongoDB is enough.

### Types

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

MongoDB stores types as GeoJSON. All geometries are either considered to be
on Earth using WGS84 reference system (SRID: 4326), or on an Euclidean plane.

The reference system is decided using the index type.

Types can also be of legacy format. This means that for data that
takes some of the given forms:

```json
<field>: [ <x>, <y> ]

<field>: [ <longitude>, <latitude> ]

<field>: { <field1>: <x>, <field2>: <y>> }

or

<field>: { <field1>: <logitude>, <field2>: <latitude> }
```

in combination with a spatial index can utilize the geospatial
queries.

### An example data model

```prisma
model geom {
  id   Int          @id @default(auto()) @map("_id")
  g    Geometry
  pt   Point        @default(Point(1 1))
  ls   LineString
  pg   Polygon
  ps   Point[]
  lss  LineString[]
  pgs  Polygon[]
  gs   Geometry[]
}
```

We have to validate the defaults. E.g. a Polygon has to connect
eventually.

### Index

MongoDB provides the following geospatial index types:

- `2dsphere` for geographical data, WGS84, SRID 4326
- `2d` for geometrical data on an Eucledian plane

A spatial index is created by defining the type to the indexed field:

```json
{ <location field>: "2dsphere" | "2d" }
```

An index can, naturally, hold other fields of different type, to make
it more exciting this is possible:

```json
{ <location field>: "2dsphere" | "2d", <other field>: -1 }
```

But, luckily this is not:

```json
{ <location field>: "2dsphere" | "2d", <other field>: "text" }
```

or this:

```json
{ <location field>: "2dsphere",  <other field>: "2d" }
```

Creating a spatial index enables some of the geospatial queries.

### List of functions

Queries:

| Requested | Name             | Index type       | Description                                             |
|-----------|------------------|------------------|---------------------------------------------------------|
| X         | `$geoIntersects` | `2dsphere`       | Geometries intersecting the given GeoJSON.              |
| X         | `$geoWithin`     | `2dsphere`, `2d` | Geometries within a bounding GeoJSON.                   |
| X         | `$near`          | `2dsphere`, `2d` | Geospatial objects in proximity to a point.             |
| X         | `$nearSphere`    | `2dsphere`, `2d` | Geospatial objects in proximity to a point on a sphere. |

Aggregations:

| Requested | Name       | Index type | Description                                                                          |
|-----------|------------|------------|--------------------------------------------------------------------------------------|
|           | `$geoNear` | `2d`       | Returns an ordered stream of documents based on the proximity to a geospatial point. |
