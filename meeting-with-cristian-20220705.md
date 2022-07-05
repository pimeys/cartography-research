Indices
- Postgres we think we have it covered
    - But all of them have specific indexes.

- 3 layers — only postgres supports more than one
    - second layer is raster
    - Cristian: we can do just one layer first
- There are simple and complex types
  - Simple: point, line, polygon
  - Complex: collections of these

- MySQL: since 5.7
- MSSQL: long time, no issue
- SQLite: SpatiaLite
    - https://www.gaia-gis.it/fossil/libspatialite/index
    - re-check license!!!!
    - probably feature flag in rusqlite
- SQLserver you have spatial index, with params, you put the width and position of the index
- PostGIS has 7 additional index _types_ (not only operators)
- Text representation WKT: use that for defaults
- Do we need to support the binary representation? (WKB)
- We will need to understand all the representations (xml in sql server, wkt, wkb, mysql spatial format) in introspection :tada:

- Functions (st_....)
  - Return different types
  - Probably not common in defaults

- Multiline == Line[]?
  - Cristian: NO! In a multiline,t he end of the multi[i] is the start of multi[i+1]. They need to be connected.

- On postgres, sqlite and sqlserver, you create a column and you can support the SRID for the column. GEOMETRY(srid = 4371)
  - It's like a collation on a text column

- Client: new namespace?

- Breaking change
  - Client (names)
  - New index types that we might have introspected as regular indexes

- Queries users want to write:

    SELECT FROM Users
        JOIN Cities
            ON st_contains(Cities.area, Users.position)
