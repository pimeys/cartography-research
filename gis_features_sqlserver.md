# SQL Server GIS Features

Docs: https://dev.mysql.com/doc/refman/8.0/en/spatial-type-overview.html

Everything works as described since forever.

## Types

The database will allow two different types:

| Native type | Prisma type | Description                                                 |
|-------------|-------------|-------------------------------------------------------------|
| `GEOMETRY`  | `Geometry`  | A flat, infinite cartesian plane. Can store all subclasses. |
| `GEOGRAPHY` | `Geometry`  | Earth. Can store all subclasses.                            |

The following types can be stored to the geometry/geography columns:

| Type              | GeoJSON Support |
|-------------------|-----------------|
| `Point`           | yes             |
| `LineString`      | yes             |
| `Polygon`         | yes             |
| `MultiPolygon`    | yes             |
| `MultiLineString` | yes             |
| `MultiPoint`      | yes             |
| `CircularString`  | no              |
| `CompoundCurve`   | no              |
| `CurvePolygon`    | no              |

### An example data model

```prisma
model geom {
  id  Int      @id
  g   Geometry @db.Geography @default(Point(..))
  p   Geometry @db.Geography @default(CurvePolygon(...))
  l   Geometry @db.Geography @default(LineString(..)
  pg  Geometry @db.Geography @default(Polygon((..))
  mp  Geometry @db.Geography @default(MultiPoint(..))
  mls Geometry @db.Geography @default(MultiLineString((..), (..)))
  mpg Geometry @db.Geography @default(MultiPolygon((..), (..))
  gc  Geometry @db.Geography @default(CircularString(..))
  ad  Geometry               @default(CompoundCurve(..))

  @@index([p], type: Spatial)
  @@index([ad], type: Spatial, boundingBox: [-10, -10, 10, 10])
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

Compared to MySQL or SQLite, SQL Server does allow one to define
default values for geometry and geography columns:

```sql
CREATE TABLE geom (
  id int IDENTITY PRIMARY KEY ,
  geog geography DEFAULT geography::STGeomFromText('LineString(1 1, 1 2)', 4326),
  geom geometry DEFAULT geometry::STGeomFromText('LineString(1 1, 1 2)', 4326)
);
```

### Index

The `CREATE SPATIAL INDEX` documentation is [quite
daunting](https://docs.microsoft.com/en-us/sql/t-sql/statements/create-spatial-index-transact-sql?view=sql-server-ver16)...

A `SPATIAL INDEX` can only be created to a single column, no compound
keys allowed.

If the column data type is `GEOMETRY`, the data doesn't by definition
have any boundaries. The cartesian plane is infinite, but an index
cannot and needs to be bound to a certain limit. This doesn't affect
`GEOGRAPHY` types due to Earth having certain boundaries.

A `SPATIAL INDEX` needs a bounding box setting in index creation, that
is defined as x minimum and maximum with y minimum and maximum.

We need a new change to allow index type on SQL Server:

```prisma
model geom {
  id Int      @id @default(autoincrement())
  p  Geometry

  @@index([p], type: Spatial, boundingBox: [-10, -10, 10, 10])
}
```

Currently a spatial index is not visible.

### Storing data

As with MySQL, the SRID value is defined on insertion:

```sql
INSERT INTO geom (geog, geom)
VALUES (
  geography::STGeomFromText('Point(1 1)', 4326),
  geometry::STGeomFromText('LineString(1 1, 1 2)', 4326),
)
```

It's important to use the functions either from the `geography` or `geometry` namespaces.

#### Selecting data

SQL Server doesn't support GeoJSON by default. It is however possible
to create a function to render the result in the correct format:

```sql
CREATE FUNCTION dbo.geometry2json( @geo geometry)
RETURNS nvarchar(MAX) AS
BEGIN
RETURN (
'{' +
(CASE @geo.STGeometryType()
WHEN 'POINT' THEN
'"type": "Point","coordinates":' +
REPLACE(REPLACE(REPLACE(REPLACE(@geo.ToString(),'POINT ',''),'(','['),')',']'),' ',',')
WHEN 'POLYGON' THEN
'"type": "Polygon","coordinates":' +
'[' + REPLACE(REPLACE(REPLACE(REPLACE(REPLACE(REPLACE(@geo.ToString(),'POLYGON ',''),'(','['),')',']'),'], ',']],['),', ','],['),' ',',') + ']'
WHEN 'MULTIPOLYGON' THEN
'"type": "MultiPolygon","coordinates":' +
'[' + REPLACE(REPLACE(REPLACE(REPLACE(REPLACE(REPLACE(@geo.ToString(),'MULTIPOLYGON ',''),'(','['),')',']'),'], ',']],['),', ','],['),' ',',') + ']'
WHEN 'MULTIPOINT' THEN
'"type": "MultiPoint","coordinates":' +
'[' + REPLACE(REPLACE(REPLACE(REPLACE(REPLACE(REPLACE(@geo.ToString(),'MULTIPOINT ',''),'(','['),')',']'),'], ',']],['),', ','],['),' ',',') + ']'
WHEN 'LINESTRING' THEN
'"type": "LineString","coordinates":' +
'[' + REPLACE(REPLACE(REPLACE(REPLACE(REPLACE(REPLACE(@geo.ToString(),'LINESTRING ',''),'(','['),')',']'),'], ',']],['),', ','],['),' ',',') + ']'
ELSE NULL
END)
+'}')
END
```

Now we can query the table:

```sql
SELECT
    JSON_QUERY(dbo.geometry2json([val])) AS [geometry],
    [val].STSrid AS [srid]
FROM g;
```

The ugly thing about this strategy is we need a separate function for
geography columns, the one written in the example works only for
geometry.

#### Joining

A quite common use case is to do spatial joins. If we have the
following DDL:

```sql
CREATE TABLE [user] (
    id int IDENTITY PRIMARY KEY,
    location geometry NOT NULL
);

CREATE SPATIAL INDEX user_location ON [user]([location]) WITH (BOUNDING_BOX = (-10, -10, 10, 10));

CREATE TABLE city (
    id int IDENTITY PRIMARY KEY,
    area geometry NOT NULL
);

CREATE SPATIAL INDEX city_area ON [city]([area]) WITH (BOUNDING_BOX = (-10, -10, 10, 10));
```

Our user is in the following location: `x = 2, y = 2`:

```sql
INSERT INTO [user] (location)
VALUES (geometry::STPointFromText('POINT(2 2)', 4326));
```

A city, shaped a a rectangle, is surrounding the point from `1,1 ->
1,3 -> 3,3 -> 3,1 -> 1,1`:

```sql
INSERT INTO [city] ([area])
VALUES (geometry::STPolyFromText('POLYGON((1 1, 1 3, 3 3, 3 1, 1 1))', 4326));
```

To find the city where the user is currently, we could join the city
area:

```sql
SELECT [user].[id], [city].[id]
FROM [user]
INNER JOIN [city]
    ON [city].[area].STContains([user].[location]) = 1;
```

### List of functions

| Requested | Name               | Description                                                                                                                                                                                                                                                  |
|-----------|--------------------|--------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| X         | STArea             | Returns the total surface area of a geography instance                                                                                                                                                                                                       |
| X         | STBuffer           | Returns a geography object that represents the union of all points whose distance from a geography instance is less than or equal to a specified value.                                                                                                      |
| X         | STContains         | Specifies whether the calling geography instance spatially contains the geography instance passed to the method.                                                                                                                                             |
| X         | STDistance         | Returns the shortest distance between a point in a geography instance and a point in another geography instance.                                                                                                                                             |
| X         | STEnvelope         | Returns the minimum axis-aligned bounding rectangle of the instance.                                                                                                                                                                                         |
| X         | STIntersection     | Returns an object that represents the points where a geography instance intersects another geography instance.                                                                                                                                               |
| X         | STIntersects       | Returns 1 if a geography instance intersects another geography instance. Returns 0 if it does not.                                                                                                                                                           |
| X         | STIsValid          | Returns true if a geography instance is well-formed and recognized as a valid geography object based on its Open Geospatial Consortium (OGC) type.                                                                                                           |
| X         | STWithin           | Returns 1 if a geography instance is spatially within another geography instance; otherwise, returns 0.                                                                                                                                                      |
|           | STAsBinary         | Returns the Open Geospatial Consortium (OGC) Well-Known Binary (WKB) representation of a geography instance.                                                                                                                                                 |
|           | STAsText           | Returns the Open Geospatial Consortium (OGC) Well-Known Text (WKT) representation of a geography instance.                                                                                                                                                   |
|           | STConvexHull       | Returns an object that represents the convex hull of a geography instance.                                                                                                                                                                                   |
|           | STCurveN           | Returns the curve specified from a geography instance that is a LineString, CircularString, or CompoundCurve.                                                                                                                                                |
|           | STCurveToLine      | Returns a polygonal approximation of a geography instance that contains circular arc segments.                                                                                                                                                               |
|           | STDifference       | Returns an object that represents the point set from one geography instance that lies outside another geography instance.                                                                                                                                    |
|           | STDimension        | Returns the maximum dimension of a geography instance.                                                                                                                                                                                                       |
|           | STDisjoint         | Returns 1 if a geography instance is spatially disjoint from another geography instance. Returns 0 if it is not.                                                                                                                                             |
|           | STEndpoint         | Returns the end point of a geography instance.                                                                                                                                                                                                               |
|           | STEquals           | Returns 1 if a geography instance represents the same point set as another geography instance. Returns 0 if it does not.                                                                                                                                     |
|           | STGeometryN        | Returns a specified geography element in a GeometryCollection or one of its subtypes. When STGeometryN() is used on a subtype of a GeometryCollection, such as MultiPoint or MultiLineString, this method returns the geography instance if called with N=1. |
|           | STGeometryType     | Returns the Open Geospatial Consortium (OGC) type name represented by a geography instance.                                                                                                                                                                  |
|           | STIsClosed         | Returns 1 if the start and end points of the given geography instance are the same. Returns 1 for geography collection types if each contained geography instance is closed. Returns 0 if the instance is not closed.                                        |
|           | STIsEmpty          | Returns 1 if a geography instance is empty. Returns 0 if a geography instance is not empty.                                                                                                                                                                  |
|           | STLength           | Returns the total length of the elements in a geography instance or the geography instances within a GeometryCollection.                                                                                                                                     |
|           | STNumCurves        | Returns the number of curves in a one-dimensional geography instance.                                                                                                                                                                                        |
|           | STNumGeometries    | Returns the number of geometries that make up a geography instance.                                                                                                                                                                                          |
|           | STNumPoints        | Returns the total number of points in each of the figures in a geography instance.                                                                                                                                                                           |
|           | STOverlaps         | Returns 1 if a geography instance spatially overlaps another geography instance, or 0 if it does not.                                                                                                                                                        |
|           | STPointN           | Returns the specified point in a geography instance.                                                                                                                                                                                                         |
|           | STSrid             | STSrid is an integer representing the spatial reference identifier (SRID) of the instance.                                                                                                                                                                   |
|           | STStartPoint       | Returns the start point of a geography instance.                                                                                                                                                                                                             |
|           | STSymDifference    | Returns an object that represents all points that are either in one geography instance or another geography instance, but not those points that lie in both instances.                                                                                       |
|           | STUnion            | Returns an object that represents the union of a geography instance with another geography instance.                                                                                                                                                         |
|           | STGeomCollFromText | Returns a geography instance from an Open Geospatial Consortium (OGC) Well-Known Text (WKT) representation, augmented with any Z (elevation) and M (measure) values carried by the instance.                                                                 |
|           | STGeomCollFromWKB  | Returns a GeometryCollectioninstance from an Open Geospatial Consortium (OGC) Well-Known Binary (WKB) representation.                                                                                                                                        |
|           | STGeomFromText     | Returns a geography instance from an Open Geospatial Consortium (OGC)Well-Known Text (WKT) representation augmented with any Z (elevation) and M (measure) values carried by the instance.                                                                   |
|           | STGeomFromWKB      | Returns a geography instance from an Open Geospatial Consortium (OGC) Well-Known Binary (WKB) representation.                                                                                                                                                |
|           | STLineFromText     | Returns a geography instance from an Open Geospatial Consortium (OGC) Well-Known Text (WKT) representation, augmented with any Z (elevation) and M (measure) values carried by the instance.                                                                 |
|           | STLineFromWKB      | Returns a LineString geography instance from an Open Geospatial Consortium (OGC) Well-Known Binary (WKB) representation.                                                                                                                                     |
|           | STMLineFromText    | Returns a geography instance from an Open Geospatial Consortium (OGC) Well-Known Text (WKT) representation, augmented with any Z (elevation) and M (measure) values carried by the instance.                                                                 |
|           | STMLineFromWKB     | Returns a geographyMultiLineString instance from an Open Geospatial Consortium (OGC) Well-Known Binary (WKB) representation.                                                                                                                                 |
|           | STMPointFromText   | Returns a geography instance from an Open Geospatial Consortium (OGC) Well-Known Text (WKT) representation, augmented with any Z (elevation) and M (measure) values carried by the instance.                                                                 |
|           | STMPointFromWKB    | Returns a geographyMultiPoint instance from an Open Geospatial Consortium (OGC) Well-Known Binary (WKB) representation.                                                                                                                                      |
|           | STMPolyFromText    | Returns a geography instance from an Open Geospatial Consortium (OGC) Well-Known Text (WKT) representation, augmented with any Z (elevation) and M (measure) values carried by the instance.                                                                 |
|           | STMPolyFromWKB     | Returns a geographyMultiPolygon instance from an Open Geospatial Consortium (OGC) Well-Known Binary (WKB) representation.                                                                                                                                    |
|           | STPointFromText    | Returns a geography instance from an Open Geospatial Consortium (OGC) Well-Known Text (WKT) representation, augmented with any Z (elevation) and M (measure) values carried by the instance.                                                                 |
|           | STPointFromWKB     | Returns a geographyPoint instance from an Open Geospatial Consortium (OGC) Well-Known Binary (WKB) representation.                                                                                                                                           |
|           | STPolyFromText     | Returns a geography instance from an Open Geospatial Consortium (OGC) Well-Known Text (WKT) representation augmented with any Z (elevation) and M (measure) values carried by the instance.                                                                  |
|           | STPolyFromWKB      | Returns a geographyPolygon instance from an Open Geospatial Consortium (OGC) Well-Known Binary (WKB) representation.                                                                                                                                         |
