---
title: "Day 10 - North America"
date: 2023-11-10T09:47:30Z
draft: false
tags: ['python','datashader', 'plotly', 'biological records', '30daymapchallenge']
---

Pretty simple one for today, NBN occurence records of one of the most notorious of US imports, the Eastern Grey Squirrel. As is often the case with biological records, the data is heavily skewed by survey effort. This is the first time I've used the [Datashader](https://datashader.org/) library, it's a useful visualisation tool when working with large datasets that would be impractical to plot as vectors. It works with plotly as below but seems easier to use with GeoViews/HoloViews and Bokeh. A key thing to consider when working with Plotly mapbox plots is that the coordinates must be in 4326, but you need to reproject to avoid [projection distotion](https://github.com/plotly/plotly.py/issues/2710), hence why the script below chops between 3857 and 4326.

{{< load-plotly >}}
{{< plotly json="/30daymapchallenge2023/day10.json" height="800px" >}}

Data: NBN Trust (2023). The National Biodiversity Network (NBN) Atlas. https://ror.org/00mcxye41.

```python
import io
import requests
import zipfile
import pandas as pd
import datashader as ds
import plotly.graph_objects as go
from colorcet import fire
from pyproj import Transformer


params = {
    'reasonTypeId': 3,
    'q':'Sciurus carolinensis'
}

url = "https://records-ws.nbnatlas.org/occurrences/index/download"

response = requests.get(url, params=params).content

zipped_file = zipfile.ZipFile(io.BytesIO(response))

csv_file_name = zipped_file.namelist()[0]

with zipped_file.open(csv_file_name) as file:
    df = pd.read_csv(file, low_memory=False)

df.rename(columns={"Latitude (WGS84)": "lat", "Longitude (WGS84)": "lon"}, inplace=True, errors="raise")

df = df[['lat','lon']] 

t3857_to_4326 = Transformer.from_crs(3857, 4326, always_xy=True)


df.loc[:, "longitude_3857"], df.loc[:, "latitude_3857"] = ds.utils.lnglat_to_meters(
    df.lon, df.lat
)

RESOLUTION=1000
cvs = ds.Canvas(plot_width=RESOLUTION, plot_height=RESOLUTION)
agg = cvs.points(df, x="longitude_3857", y="latitude_3857")
img = ds.tf.shade(agg, cmap=fire).to_pil()

fig = go.Figure(go.Scattermapbox())
fig.update_layout(
    mapbox={
        "style": "carto-darkmatter",
        "zoom": 5,
        "layers": [
            {
                "sourcetype": "image",
                "source": img,
                # plotly needs coords in EPSG:4326
                "coordinates": [
                    t3857_to_4326.transform(
                        agg.coords["longitude_3857"].values[a],
                        agg.coords["latitude_3857"].values[b],
                    )
                    for a, b in [(0, -1), (-1, -1), (-1, 0), (0, 0)]
                ],
            }
        ],
        "center": 
            {"lat": 54.826008, "lon": -3.8122559}
    },
    height=800,
    margin={"l": 0, "b": 0, "t": 0, "r": 0},
)
fig.show()
```
