---
title: "Using docker with Gdal"
date: 2023-07-23T22:32:05+01:00
draft: false
tags: ['docker','gdal']
---

Note - this assumes you have docker installed and are broadly familiar with using it from the command line.

Anyone who works with GIS will be familiar with the Geospatial Data Abstraction Library, or GDAL as it's more commonly know. It's the swiss army knife of GIS tools - allowing users to read, write and convert between numerous raster and vector formats. For those who aren't familiar there are plenty of resources around that give an introduction, a good place to start is the [official GDAL tutorial page.](https://gdal.org/tutorials/index.html)

For most users, GDAL is something that comes bundled with QGIS and accessed either through OSGeo4W Shell (Windows) or the terminal (MacOS/Linux). This is usually fine, however, the version of GDAL that is bundled with QGIS may often be a few releases behind and you might be missing out on newer features that you need or want to try. One option is to install the newer version directly, but this can often be a hassle, with a need to redfine system variables. Another option is to use Docker to run a container pre-built with a specific version of GDAL, this negates any need to set system variables.

A full list of docker images is [available here](https://github.com/OSGeo/gdal/pkgs/container/gdal/versions?filters%5Bversion_type%5D=tagged). If you're just seeking to get a container up and running quickly for using the command line tool then either `alpine-small` or `ubuntu-small` will do. The command to spin up a container and start a shell session is like so:
```docker
docker run -ti --rm  ghcr.io/osgeo/gdal:alpine-small-latest sh
```
Note, the `--rm` flag which means the container will be removed upon exit. This will open a terminal prompt with the docker container, try using `ogrinfo --version` to see which version of GDAL is installed:
```console
$ ogrinfo --version
GDAL 3.8.0dev-6e268bb61c95462d2704dca2ec0cad7d95d32df1, released 2023/07/20
```
Exit the container and let's now test using some spatial data. First let's copy the below into a json file in the current directory (note this will only work on Unix systems, if on windows you can copy paste into a json file)

```console
$ echo '{ "type": "FeatureCollection", "features": [{"type": "Feature","properties": {"name": "Bristol"},"geometry": {"coordinates": [-2.597643713766729, 51.45504657056361],"type": "Point"}}]}' > test.json
```
Next we need to adjust the docker command to mount the current directory as a volume:
```docker
docker run -ti --rm -v $PWD:/data/ ghcr.io/osgeo/gdal:alpine-small-latest sh
```
This will open another terminal, you can then navigate to the mounted directory and inspect the file using ogrinfo:
```console
$ cd data
$ ogrinfo test.json
INFO: Open of `test.json'
      using driver `GeoJSON' successful.
1: test (Point)
```
Now we're happy it recognises it as a spatial file let's try converting it using an ogr2ogr command:
```console
$ ogr2ogr -f "ESRI Shapefile" test.shp test.json -t_srs "EPSG:27700" 
```
The above command converts the json file into a shapefile with EPSG:27700. Because we have mounted the drive, it will be accessible both within the container and the host machine.