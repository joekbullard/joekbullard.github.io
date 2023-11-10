---
title: "Day 10 - North America"
date: 2023-11-10T09:47:30Z
draft: true
---

Pretty simple one for today, all biological records of one of the most notorious of US imports, the Eastern Grey Squirrel.

For the plot I used plotly and datashader.

{{< load-plotly >}}
{{< plotly json="/day10.json" height="800px" >}}

```python
import requests
import zipfile
import pandas as pd
import datashader as ds
import io
import datashader as ds, colorcet
from pyproj import Transformer
import colorcet
import plotly.graph_objects as go

params = {
    'reasonTypeId': 10,
    'q':'*:*',
    'fq':'genus:Sciurus'
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
img = ds.tf.shade(agg, cmap=colorcet.fire).to_pil()

fig = go.Figure(go.Scattermapbox())
fig.update_layout(
    mapbox={
        "style": "carto-darkmatter",
        "zoom": 5,
        "layers": [
            {
                "sourcetype": "image",
                "source": img,
                # Sets the coordinates array contains [longitude, latitude] pairs for the image corners listed in
                # clockwise order: top left, top right, bottom right, bottom left.
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
            {"lat": 53.943155, "lon": -3.8122559}
    },
    height=800,
    margin={"l": 0, "b": 0, "t": 0, "r": 0},
)
```
