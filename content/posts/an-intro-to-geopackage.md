---
title: "An Intro to Geopackage"
date: 2023-03-23T08:32:20Z
draft: false
tags: ['python','geoackage','gdal']
---

Geopackage is a format that will be familiar to many who work with spatial data, particularly those that use QGIS. In the majority of use cases it will be treated as little more than a shapefile replacement - i.e. a means to manage and share vector data. But what if I told you that by using a geopackage you could have access to a whole range of additional features and functionality more typically associated with a relational database management system (RDMS) like PostgreSQL/PostGIS? In this three part series we will go on a deep dive into some of the finer details of geopackage - but first lets start with the fundemental question - what is a GeoPackage?

## What is a geopackage?
The geopackage format is simply a container for the ubiquitous SQLite file format. What is SQLite? Well, quoting [sqlite.org](https://sqlite.org/index.html):
> _SQLite is a C-language library that implements a small, fast, self-contained, high-reliability, full-featured, SQL database engine._

In simple terms, SQLite is a SQL database engine contained within a single file - geopackage builds on this through the implementation of a standard that allows users to store spatial data. A key point here - unlike the shapefile, which many compare it to, geoackage is not just limited to vector (points, lines, polygons) data, but also raster - more on that later. In addition, it also incldes a suite of spatial functions. To better understand what a geoackage is, let's create one and have a look around.

## Making a GeoPackage
For this part we will use the python GDAL library to generate a geopackage with a few different vector tables. For those not aware, [GDAL](https://gdal.org/), or Geospatial Data Abstraction Library, is the swiss army knife of dealing with geospatial data formats, allowing you to convert raster and vector data between hundreds of formats. If you've not used the python bindings before - there's a bit of a learning curve but there are some good resources out there - notably [this Cookbook](https://pcjericks.github.io/py-gdalogr-cookbook/index.html).

**Note** - Depending on your OS, GDAL can be tricky to install - the easiest way is probably to [use Conda](https://opensourceoptions.com/blog/how-to-install-gdal-with-anaconda/), if you just want to run the script below you can also just do so using the QGIS python console.

Using this script below, we will create an empty geopackage in the current directory called `sample.gpkg`.

```python
from osgeo import ogr

filename = './sample.gpkg'
driver = ogr.GetDriverByName('GPKG')
data_source = driver.CreateDataSource(filename)
data_source = None
```
As a geopackage is an extension of sqlite, we can inspect it using the `sqlite3` command line shell and `.tables` command (if you want you could also connect to the geopackage using the dbmanager in QGIS).

```bash
sqlite3 sample.gpkg
sqlite> .tables
gpkg_contents          gpkg_ogr_contents      gpkg_tile_matrix     
gpkg_geometry_columns  gpkg_spatial_ref_sys   gpkg_tile_matrix_set
```
You will see there are six tables included in the geopackage, for now we will focus on `gpkg_contents`, `gpkg_geometry_columns` and `gpkg_spatial_ref_sys` - each of which is summarised below.

### `gpkg_contents`
This table does more or less what you would expect given the name, it's a contents table for a given GeoPackage that lists all geometry tables. A full specification can be seen on the [table definition on geopackage.org](http://www.geopackage.org/spec120/#_contents). If you were to query this table now:

```bash
sqlite> select * from gpkg_contents;
```

It will not return anything, as we don't yet have any spatial tables in the geopackage.

### `gpkg_spatial_ref_sys`
Much like `gpkg_contents`, I'm sure you can hazard a guess as to what you this table contains. If you guessed coordinate/spatial reference systems, then well done. Note that by default this table contains 3 rows - 4326, -1 for undefined Cartesian CRS's and 0 for undefined geographic CRS's, if you wish to work with a CRS other than 4326 then you will need to add it to this table.

You can take a look at the default rows:

```bash
sqlite> select srs_name from gpkg_spatial_ref_sys;
Undefined Cartesian SRS
Undefined geographic SRS
WGS 84 geodetic
```

### `gpkg_geometry_columns`
Lastly - `gpkg_geometry_columns` contains metadata relating to the geometry column for a given spatial table - including geometry type and CRS. As with `gpkg_contents`, this will currently be empty.

## Adding tables
So now let's try adding some spatial tables to the geopackage we just made.

```python
from osgeo import ogr, osr

filename = './sample.gpkg'
driver = ogr.GetDriverByName('GPKG')

srs = osr.SpatialReference()
srs.ImportFromEPSG(27700)

data_source = driver.Open(filename, 1)

point_layer = data_source.CreateLayer("points", srs, ogr.wkbPoint)
line_layer = data_source.CreateLayer("lines", srs, ogr.wkbLineString)
poly_layer = data_source.CreateLayer("polygons", srs, ogr.wkbPolygon)

data_source = None
```
Above we have created three spatial tables, `points`, `lines` and `polygons`, each of which is projected as EPSG:27700 or Britsh National Grid. We can again inspect the geopackage using the `sqlite3`:

```zsh
sqlite3 sample.gpkg
sqlite> .tables
gpkg_contents               rtree_lines_geom_node     
gpkg_extensions             rtree_lines_geom_parent   
gpkg_geometry_columns       rtree_lines_geom_rowid    
gpkg_ogr_contents           rtree_points_geom         
gpkg_spatial_ref_sys        rtree_points_geom_node    
gpkg_tile_matrix            rtree_points_geom_parent  
gpkg_tile_matrix_set        rtree_points_geom_rowid   
lines                       rtree_polygons_geom       
points                      rtree_polygons_geom_node  
polygons                    rtree_polygons_geom_parent
rtree_lines_geom            rtree_polygons_geom_rowid 
```

Now we can see there are a significantly greater number of tables. In addition to the three we just created, there are a number of `rtree` tables - what are they? R-trees are data structures based on the concept of trees that are widely used within spatial databases and GIS to index multi-dimensional data. Within SQLite, R-trees are implemented by the [rtree module](https://www.sqlite.org/rtree.html). Each parent node represents a bounding box enclosing a set of child objects, this pattern is repeated down to the leaf nodes that contain the actual data. In short, R-trees enable efficient range queries, nearest neighbour searches and spatial joins - but realistically, you will probably never need to worry yourself about them.

A quick note, while you can connect to a geopackage using `sqlite3` and create tables with DDL commands directly, you will end up needing to manually add to the geopckage metadata tables and generate R-Trees - it's therefore much easier to use `ogr` which undertakes many of the steps for you.

## Adding data to tables
Let's try populating our tables with some sample data.

```python
from osgeo import ogr

filename = './sample.gpkg'
driver = ogr.GetDriverByName('GPKG')

data_source = driver.Open(filename, 1)
point_layer = data_source.GetLayerByName('points')
line_layer = data_source.GetLayerByName('lines')
poly_layer = data_source.GetLayerByName('polygons')

feature = ogr.Feature(point_layer.GetLayerDefn())
point = ogr.Geometry(ogr.wkbPoint)
point.AddPoint(358776, 172559)
feature.SetGeometry(point)
point_layer.CreateFeature(feature)
feature = None

feature = ogr.Feature(line_layer.GetLayerDefn())
line = ogr.Geometry(ogr.wkbLineString)
line.AddPoint(356204, 175408)
line.AddPoint(361514, 171273)
feature.SetGeometry(line)
line_layer.CreateFeature(feature)
feature = None

feature = ogr.Feature(poly_layer.GetLayerDefn())
ring = ogr.Geometry(ogr.wkbLinearRing)
ring.AddPoint(356540, 170849)
ring.AddPoint(359257, 176251)
ring.AddPoint(363364, 173020)
ring.AddPoint(356540, 170849)

polygon = ogr.Geometry(ogr.wkbPolygon)
polygon.AddGeometry(ring)

feature.SetGeometry(polygon)
poly_layer.CreateFeature(feature)
feature = None

data_source = None
```

While this may look like a lot, all it's doing is creating a geometry feature for each of the tables we created - try opening the data in QGIS. We can now inspect the tables using `sqlite3`

```zsh
sqlite3 sample.gpkg
sqlite> select AsText(CastAutomagic(geom))as geom from points;
geom                
--------------------
POINT(358776 172559)
```

One additonal thing to note - we can inspect the `gpkg_contents` table.

```zsh
sqlite> select * from gpkg_contents;
table_name  data_type  identifier  description  last_change               min_x     min_y     max_x     max_y     srs_id
----------  ---------  ----------  -----------  ------------------------  --------  --------  --------  --------  ------
points      features   points                   2023-03-23T19:26:10.493Z  358776.0  172559.0  358776.0  172559.0  27700 
lines       features   lines                    2023-03-23T19:26:10.506Z  356204.0  171273.0  361514.0  175408.0  27700 
polygons    features   polygons                 2023-03-23T19:26:10.516Z  356540.0  170849.0  363364.0  176251.0  27700 
```
You can see that the `min_x`, `min_y`, `max_x` and `max_y` have all adjusted automatically inline with the extents of the new features.

That's probably enough of an introduction to geopackage, in the next part we will look into how we can leverage procedures like `TRIGGER` to move spatial processing logic from desktop GIS into the geopackage as well as how we can integrate this into QGIS projects.




