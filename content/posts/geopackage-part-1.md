---
title: "GeoPackage and Python - Part 1"
date: 2023-03-20T19:39:57Z
draft: false
tags: ['python','geopackage']
---

GeoPackage (GPKG) is a spatial format that many QGIS users will be familiar with, as a format it offers [many advantages](http://switchfromshapefile.org/#geopackage) over the ubiquitous shapefile format. In short, the Geopackage Encoding Standard enforces a set of rules and standards for storing vector features (Points, Lines, Polygons) and raster data in an SQLite database. While many people will use GPKG as a like-for-like shapefile replacement, through knowledge of some fairly basic SQL it can offer some of the advanced functionality typically associated with spatial databases such as PostgreSQL/PostGIS. In this post, I will go through some of the requirements of the format specification and how you can create and interact with a geopackage using python

## Requirements
* Python
* Virtual environment manager - I will be using [poetry](https://python-poetry.org/)
* QGIS

## Setting up a virtual environment
Before starting, we will need to set up a development environment. If you're using poetry, you can do so in the following way:

```shell
$ poetry new py-gpkg
```
Next we will install the python gdal bindings:

```shell
$ poetry add GDAL
```

Now we are good to go

## Creating a blank geopackage
There are a number of ways to create a geopackage, you can use QGIS or download an [empty geopackage](http://www.geopackage.org/data/empty.gpkg). In this tutorial we will be using python and the GDAL and SQLite libraries. To start off with, we will use GDAL to create a blank geopackage. Create a blank python file called `create_gpkg.py` and add the following:

**Note** - for those unfamiliar with the python GDAL bindings, it has a few quirks that may catch you out, it also uses camel case.

```python
import os
from osgeo import ogr

driver = "GPKG"
filename = "generated.gpkg"

driver = ogr.GetDriverByName(driver)

if os.path.exists(filename):
    driver.DeleteDataSource(filename)

data_source = driver.CreateDataSource(filename)

data_source = None
```

If you run the above file using `poetry run python create_gpkg.py` it should create an empty geopackage called `generated.gpkg`

## A brief word on the GeoPackage specification
Before we go any further, it's worth taking a moment to understand a few bits about the OGC GeoPackage standard - more specifically - the mandatory metadata tables [`gpkg_contents`](http://www.geopackage.org/spec120/#_contents) and [`gpkg_spatial_ref_sys`](http://www.geopackage.org/spec120/#spatial_ref_sys). In addition to the two mandatory tables, there's also one non-mandatory table that's worth knowing about - `gpkg_geometry_columns`. All three of these are summarised below.

### `gpkg_contents`
This table does more or less what you would expect given the name, it's a contents table for a given GeoPackage that lists all gemetry tables. Mandatory fields and their data type are shown below, more information can be seen on the [table definition on geopackage.org](http://www.geopackage.org/spec120/#_contents)

| Field         | Type        |
| ------------- | ----------- |
| `identfier`   | TEXT        |
| `description` | TEXT        |
| `min_x`       | DOUBLE      |
| `min_y`       | DOUBLE      |
| `max_x`       | DOUBLE      |
| `max_y`       | DOUBLE      |
| `srs_id`      | INTEGER     |


### `gpkg_spatial_ref_sys`
Much like `gpkg_contents`, I'm sure you can hazard a guess as to what you this table contains. If you guessed coordinate/spatial reference systems, then well done. Mandatory fields are shown below, and more information can be seen in the [table definition](http://www.geopackage.org/spec120/#spatial_ref_sys). Note that by default this table contains 3 rows - 4326, -1 for undefined Cartesian CRS's and 0 for undefined geographic CRS's, if you wish to work with a CRS other than 4326 then you will need to add it to this table.

| Field                      | Type        |
| ---------------------------| ----------- |
| `srs_name`                 | TEXT        |
| `srs_id`                   | INTEGER     |
| `organization`             | TEXT        |
| `organization_coordsys_id` | TEXT        |
| `definition`               | TEXT        |

### `gpkg_geometry_columns`
Lastly - `gpkg_geometry_columns` contains metadata relating to the geometry column for a given spatial table - including geometry type and CRS.

## Defining tables using `sqlite3`
Now let's connect to the GeoPackage and start creating some tables. Python has a built in library for interacting with SQLite databases - [sqlite3](https://docs.python.org/3/library/sqlite3.html).

To connect to the GeoPackage we just made:

```python
import sqlite3

conn = sqlite3.connect('generated.gpkg')
```

Now for the purpose of this topic, let's make two simple geometry tables, `point` and `polygon`. We will be working with EPSG:2770 for no reason other than it's the CRS I use most frequently myself. Following on from above, run the code below.

```python
srs_name = 'OSGB36 / British National Grid'
srs_id = 27700
srs_org = 'EPSG'
srs_org_id = 27700
srs_def = """PROJCS["OSGB36 / British National Grid",GEOGCS["OSGB36",DATUM["Ordnance_Survey_of_Great_Britain_1936",SPHEROID["Airy 1830",6377563.396,299.3249646],TOWGS84[375,-111,431,0,0,0,0]],PRIMEM["Greenwich",0,AUTHORITY["EPSG","8901"]],UNIT["degree",0.0174532925199433,AUTHORITY["EPSG","9122"]],AUTHORITY["EPSG","4277"]],PROJECTION["Transverse_Mercator"],PARAMETER["latitude_of_origin",49],PARAMETER["central_meridian",-2],PARAMETER["scale_factor",0.9996012717],PARAMETER["false_easting",400000],PARAMETER["false_northing",-100000],UNIT["metre",1,AUTHORITY["EPSG","9001"]],AXIS["Easting",EAST],AXIS["Northing",NORTH],AUTHORITY["EPSG","27700"]]"""
srs_desc = 'Easting / northing, units in meters'

cursor = conn.cursor()

cursor.execute('insert into gpkg_spatial_ref_sys VALUES (?, ?, ?, ?, ?, ?)', (srs_name, srs_id, srs_org, srs_org_id, srs_def, srs_desc))

cursor.executescript('''
    CREATE TABLE 'point' (
    id INTEGER PRIMARY KEY AUTOINCREMENT,
    geom POINT NOT NULL,
    name TEXT NOT NULL,
    description TEXT NOT NULL,
    buffer INTEGER NOT NULL
    );

    CREATE TABLE 'poly' (
    id INTEGER PRIMARY KEY AUTOINCREMENT,
    geom POLYGON NOT NULL,
    point_id INTEGER,
    FOREIGN KEY(point_id) REFERENCES point(id));
''')

cursor.execute('''insert into gpkg_contents values ('point', 'features','point','point','2019-02-19T10:49:06.022Z',NULL,NULL,NULL,NULL,27700);''')
cursor.execute('''insert into gpkg_contents values ('poly', 'features','poly','poly','2019-02-19T10:49:06.022Z',NULL,NULL,NULL,NULL,27700);''')

cursor.execute('''insert into gpkg_geometry_columns values ('point','geom','POINT',27700, 0, 0 )''')
cursor.execute('''insert into gpkg_geometry_columns values ('poly','geom','POLYGON',27700, 0, 0 )''')

conn.commit()
```
Let's run over what we've just done. The first few lines set out the mandatory fields required for the `gpkg_spatial_ref_sys` table to insert a row for EPSG:27700, we then create a `cursor` object using the previously used connection to the geopackage and insert the row into the `gpkg_spatial_ref_sys` table. 

Next up, we create two tables, `point` and `poly` uing `executescript` method - be sure to take a look at the constraints and types used. After that we insert entries for both tables into `gpkg_contents` and `gpkg_geometry_columns` metadata tables before commiting the changes to the database using `commit`. You should now be able to add the geopackage to QGIS and both `point` and `poly` will be recognised as valid layers.

That's it for part 1, next up we will go into some more advanced functionality, including triggers to automate logic with SQL that would otherwise be handled by scripting or using QGIS processing tools.

