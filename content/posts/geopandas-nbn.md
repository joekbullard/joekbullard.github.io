---
title: "Manipulating NBN species data with Geopandas"
date: 2022-11-21T01:07:07Z
draft: true
---

Geopandas has established itself a great python library for working with spatial data, building on the much loved pandas library and adding the ability to perform spatial operations. In this post I'll run through some of the capabilities of the library. I'm using some data from the NBN Atlas - a great resource for species records. This particular dataset contains all mammal records recorded in the locality of Bristol, UK. The code can be found on my github.

Before we start, we need to have the following dependencies installed (Note that while I'm using venv with pip, it might be easier for you to use Anaconda but that's your call):
```shell
pip install geopandas 
pip install matplotlib
pip install contextily
pip install osgb
```
To start off, we will actually use pandas to read in the csv data:
```python
import pandas as pd

df = pd.read_csv('mammal_data.csv')
```
You can have a quick look at the dataframe using `df.head()`. Now we have the dataframe loaded, let's clean and drop some of the rows and columns we're not interested in. For this exercise I'll be focusing on bat records, so let's start by getting rid of all species that aren't bats. You may or may not know that bats are order Chiroptera, so let's filter the dataframe using that column:
```python
df = df[df['Order'] == 'Chiroptera']
len(df)
```
Now we're left with around 3000 records. Next, let's remove some of the older records, for the purpose of this exercise I'm going to pick an arbitrary cut off point of 2010.
```python
df = df[df['Start date year'] >= 2010]
len(df)
```
Now we're down to about 1500 records. Next let's remove some of the unwanted columns, given how many there are, it's easier to name the columns we want to keep then drop the rest. Let's do that by making a list of the columns we do want then use Index.intersection to drop the rest.
```python
cols_to_keep = ['Scientific name', 'Common name', 'Start data year', 'OSGR']
df = df[df.columns.intersection(cols_to_keep)]
```
Now we have a more manageable 4 columns in our dataframe. One last step we need to do with pandas is to convert the OSGR into eastings and northings. For those who don't know, OSGR stands for Ordnance Survey Grid Reference and comprises a grid based system that covers the entirety of the UK. It takes the format AB 123 123, where AB refers to a grid and the subsequent numbers indicate the easting and northings from the origin of the grid. Unfortunately, grid references are not readable by GIS and need to be converted into XY format - for that we will use the osgb library. While we could use the existing lat/lon columns, this way is a little more fun and will come in handier later on when working out accuracy of the spatial data. This one liner uses list comprehension to return a tuple of (easting, northing) values that are assigned to the newly created columns in the dataframe.
```python
import osgb

df[['easting', 'northing']] = [osgb.parse_grid(x) for x in df['OSGR']]
```
One last thing to do before we convert to a geodataframe, let's work out the spatial accuracy of the records. To do that we will need to make a simple function to assign accuracy based on the length of the value in the OSGR column.
```python
def osgr_accuracy(osgr):
    if len(osgr) == 4:
        return 10000
    elif len(osgr) == 6:
        return(1000)
    elif len(osgr) == 8:
        return (100)
    elif len(osgr) == 10:
        return (10)
    elif len(osgr) == 12:
        return 1

df['precision'] = df['OSGR'].apply(lambda x: osgr_accuracy(x))
```
Now we have easting northing coordinates as well as spatial accuracy of each record. We're now ready to read convert the dataframe into a geodataframe using geopandas.
```python
import geopandas as gpd

gdf = gpd.GeoDataFrame(
    df, geometry=gpd.points_from_xy(df.easting, df.northing))
```
Let's try plotting the data using the built in plot() function of geopandas:
```python
gdf.plot()
```

That will do for now, but first let's save our geodataframe as a file so we can pick up easily next time. Geopandas can output in many different formats, but for now we'll use geopackage/gpkg - because we really need to stop using shapefiles.
```python
gdf.to_file('bristol_bats.gpkg', driver='GPKG')
```
Done, next time we will get stuck into the spatial side of things.