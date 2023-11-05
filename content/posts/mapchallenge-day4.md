---
title: "Day 4 - A bad map"
date: 2023-11-05T22:43:36Z
draft: false
tags: ['python','overpass', '30daymapchallenge']
---

Not the finest of creations, but I guess that's the point. Pretty simple really, this map shows the location of all villages/towns/cities that are named "Bad". For a reason I can't quite figure out, Switzerland seems to be the baddest of them all.
{{< load-plotly >}}
{{< plotly json="/day04.json" height="600px" >}}

```python
import geopandas as gpd
import numpy as np
import requests
from shapely.geometry import Point
import plotly.express as px

overpass_url = "http://overpass-api.de/api/interpreter"
overpass_query = """
[out:json];
(
  node["place"="city"]["name"~"^Bad$"];
  node["place"="town"]["name"~"^Bad$"];
  node["place"="village"]["name"~"^Bad$"];
  node["place"="hamlet"]["name"~"^Bad$"];
  node["place"="isolated_dwelling"]["name"~"^Bad$"];
);
out center;
"""
response = requests.get(overpass_url, 
                        params={'data': overpass_query})
data = response.json()

# Collect coords into list
coords = []
for element in data['elements']:
  if element['type'] == 'node':
    lon = element['lon']
    lat = element['lat']
    coords.append((lon, lat))
  elif 'center' in element:
    lon = element['center']['lon']
    lat = element['center']['lat']
    coords.append((lon, lat))
  
coord_array = np.array(coords)

geometry = [Point(xy) for xy in coord_array]

gdf = gpd.GeoDataFrame(geometry=geometry, columns=['geometry'])

lat = float(gdf.dissolve().centroid.y.iloc[0])
lon = float(gdf.dissolve().centroid.x.iloc[0])

fig = px.density_mapbox(gdf, lat=gdf.geometry.y, lon=gdf.geometry.x, radius=20, center = dict(lat = lat, lon = lon),
                        zoom = 2, color_continuous_scale = 'viridis',
                        mapbox_style = 'carto-darkmatter', title='A useless map showing all locations with the name "Bad"', width=800, height=600)

fig.update_layout(coloraxis_colorbar=dict(title={'text':'Badness'}))

fig.show()
```