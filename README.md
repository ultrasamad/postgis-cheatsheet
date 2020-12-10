# PostGIS Cheatsheet
A curated list of useful PostGIS commands with PostgreSQL.\
We will be primaryly dealing with **GEOGRAPHY** data types.

Note: A **GEOGRAPHY** type is same as **GEOMETRY** type with SRID of *4326*\. So anywhere you see *4326* that refers to *SRID*\
If you run these commands in [pgAdmin4](https://www.pgadmin.org/download/), you get the opportunity to view these geo data in a map.

## Table of contents
- [Why use PostGIS](#why-postgis)
- [Installing PostGIS](#installation) 
- [Enable extension](#enable-extension)
- [Verify installation](#verify-installation)
- [Upgrade PostGIS](#upgrade-postgis)
- [Create table](#create-table)
- [Add a Geo column to an existing table](#add-column)
- [Insert data into table](#insert-data)
- [Build point from coordinates](#point-from-coordinates)
- [Build polygon from coordinates](#polygon-from-coordinates)
- [Build circle from a point and radius](#circle-from-radius)
- [Parse WKB](#parse-wkb)
- [Point in polygon example](#point-in-polygon)
- [Bounding box example](#bounding-box-example)
- [Useful links](#useful-links)
***
## Why PostGIS
If your application has any geofence feature where you provide information about a place to users or requires that you save some geolocation data to your database, you might consider using PostGIS if you are not already using it.\
This is an interesting article comparing Geofencing from within code vs using PostGIS.\
[Beating Uber geofencing with PostGIS](https://www.cybertec-postgresql.com/en/beating-uber-with-a-postgresql-prototype/)
***
## Installation
To install PostGIS on ubuntu, run this command\
`sudo apt install postgis`

***
## Enable extension
Enable PostGIS extension for the currently selected database \
`CREATE EXTENSION postgis;`
***
## Verify installation
Verify that the PostGIS extension is enabled properly\
`SELECT postgis_full_version();` \
OR\
 `SELECT postgis_version();`

***
## Upgrade PostGIS
```sql 
ALTER EXTENSION postgis UPDATE TO "VERSION_NUMBER";
```
***
## Create table
This is a sample SQL command to create a table that has geography supported column

```sql
CREATE TABLE regions(
    id SERIAL PRIMARY KEY,
    name VARCHAR(50),
    center  GEOMETRY(POINT, 4326),
    boundary GEOMETRY(POLYGON, 4326)
);
```
OR
```sql
CREATE TABLE regions(
    id SERIAL PRIMARY KEY,
    name VARCHAR(50),
    center  GEOGRAPHY(POINT),
    boundary GEOGRAPHY(POLYGON)
);
```
***

## Add Column
Use this command to add a geo column type to an existing table.

```sql
ALTER TABLE cities ADD COLUMN center GEOMETRY(POINT, 4326);
```
OR

```sql 
AddGeometryColumn(cities, center, 4326, POINT, 2);
```
***

## Insert data
```sql
INSERT INTO regions(center, boundary)
VALUES(
    ST_GeogFromText('POINT(-0.2577338 5.5548604)'),
    ST_GeogFromText('POLYGON((-0.269149 5.548880, -0.266059 5.571860, -0.246404 5.566393, -0.240053 5.554946, -0.255159 5.543413, -0.269149 5.548880))')
);
```
If you are using [Laravel Eloquent](https://laravel.com/docs/8.x/eloquent), you can save a geometry type like this
```php
$geom = sprintf('POINT(%f %f)', $lon, $lat);
$city = new City;
$city->name = 'Accra';
$city->center = DB::raw("ST_GeogFromText($geom)");
$city->save();
```
***

## Point from coordinates
Given a pair of coordinates ie. *latitude* and *longitude*, you can construct a geometry point using below query

```sql 
SELECT ST_GeogFromText('POINT(longitude latitude)');
```
OR
```sql
SELECT ST_GeomFromText('POINT(longitude latitude)', 4326);
```
OR

```sql 
SELECT ST_SetSRID(ST_MakePoint(longitude, latitude), 4326);
```
OR
```sql
SELECT ST_GeomFromEWKT('SRID=4326;POINT(-2.5441288 10.0611456)');
```
## Polygon from coordinates
You can construct a polygon from a set of latitudes and longitudes. The starting set points must be same as the last points. ie The polygon must be closed.

```sql
SELECT ST_GeogFromText('POLYGON((lon1 lat1, lon2 lat2, lon3 lat3, lon1 lat1))');
```
***

## Circle from radius
This example shows how to construct a circle given a lat and lon coordinates together with a radius
```sql
SELECT ST_Buffer(ST_MakePoint(lon, lat)::geography, radius);
```
Note: Radius is in meters
***
#Parse WKB
These are some examples showing how to parse Well Known binaries(WKB)
- Parse to text

```sql
SELECT ST_AsEWKT('0101000020E610000012D90759164CD0BF3F19E3C3EC351640');
```
- Parse to json
```sql
SELECT ST_AsGeoJSON('0101000020E610000012D90759164CD0BF3F19E3C3EC351640');
```
***
## Point in Polygon
There instaces where you want to determine if a point is inside a polygon. Use this query

```sql
SELECT ST_Contains(polygon_column, point_column);
```
Example: 
```sql
SELECT * FROM cities WHERE ST_Contains(boundary, 'SRID=4326;POINT(lon lat)');
```

Select points ordered by how close they are to a specific point
```sql
SELECT * FROM towns
ORDER BY geom_column <-> (SETSRID(ST_MakePoint(lon, lat)), 4326) LIMIT 10
```
***
## Bounding box example
One common use case when working with geolocation data is, you may want to return all points that lie within a bounding box.\
Maybe you want to render points onto an area of a Google or Mapbox that is currently visible to the user.

- Use this query when your coordinates are not stored as geo types ie you have float type columns for lat and lon.
```sql
SELECT * FROM cities
WHERE ST_Contains(ST_MakeEnvelope(minLon, minLat, maxLon, maxLat, 4326), ST_SetSRID(ST_MakePoint(longitude, latitude), 4326));
```

Note: If your column types is **VARCHAR**, you will have to cast them to floats like `longitude::float` and `latitude::float`

- In the example below, the *center* column is already a geometry type

```sql
SELECT * FROM cities
WHERE ST_Contains(ST_MakeEnvelope(minLon, minLat, maxLon, maxLat, 4326), cities.center);
```
If you are using leafletjs on your frontend, you can get the *min* and *max* coordinates using below snippet

```javascript
const boundBox = map.getBounds()
const northEast = boundBox.getNorthEast()
const southWest = boundBox.getSouthWest()

const minLat = southWest.lat
const maxLat = northEast.lat
const minLng = northEast.lng
const maxLng = southWest.lng
```
You can then send this via ajax to your backend.
