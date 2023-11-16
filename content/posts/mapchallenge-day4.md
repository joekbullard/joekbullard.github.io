---
title: "Day 4 - A bad map"
date: 2023-11-05T22:43:36Z
draft: false
tags: ['python','overpass', '30daymapchallenge']
---

Not the finest of creations, but I guess that's the point. Pretty simple really, this map shows the location of all villages/towns/cities that are named "Bad". For a reason I can't quite figure out, Switzerland seems to be the baddest of them all.
{{< load-plotly >}}
{{< plotly json="/30daymapchallenge2023/day04.json" height="400px" >}}

```python
import geopandas as gpd
import plotly.express as px
import requests
from shapely.geometry import Point

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
d = {"name": [], "type": [], "geometry": []}
for element in data["elements"]:
    if element["type"] == "node":
        lon = element["lon"]
        lat = element["lat"]
        name = element["tags"]["name"]
        settlement_type = element["tags"]["place"]

    elif "center" in element:
        lon = element["center"]["lon"]
        lat = element["center"]["lat"]

    d["name"].append(name)
    d["type"].append(settlement_type)
    d["geometry"].append(Point(lon, lat))


gdf = gpd.GeoDataFrame(d, crs="EPSG:4326")
  
lat = float(gdf.dissolve().centroid.y.iloc[0])
lon = float(gdf.dissolve().centroid.x.iloc[0])

fig = px.scatter_mapbox(
    gdf,
    lat=gdf.geometry.y,
    lon=gdf.geometry.x,
    center=dict(lat=lat, lon=lon),
    zoom=2,
    opacity=0.6,
    hover_data=["name", "type"],
    mapbox_style="carto-darkmatter",
    width=600,
    height=400,
)

fig.update_layout(
    margin={"r": 0, "t": 0, "l": 0, "b": 0},
)

fig.show()
```