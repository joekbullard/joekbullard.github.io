---
title: "Day 11 - Retro"
date: 2023-11-11T10:58:41Z
draft: false
tags: ['python', '30daymapchallenge']
---

This one makes use of the brilliant [prettymaps](https://github.com/marceloprates/prettymaps/tree/main) library by [Marcelo Prates](https://github.com/marceloprates). This library makes it incredibly simple to create aesthetically pleasing maps using Open Street Map data.

![Retro Bristol](/30daymapchallenge2023/retro_bristol.png)

```python
import prettymaps
from matplotlib.font_manager import FontProperties
import matplotlib.pyplot as plt

plot = prettymaps.plot(
    'Cabot Tower, Bristol, United Kingdom', 
    preset='macao',
    credit={
        'text':"data © OpenStreetMap contributors",
        'y': 0
    }
)

plot.fig.patch.set_facecolor('#F2F4CB')
# Add title
plot.ax.set_title(
    'Bristol',
    fontproperties = FontProperties(
        fname = 'BrightFont.otf',
        size = 70
    )
)

plt.show()
```
