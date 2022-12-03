---
title: "Plotting Geospatial Data"
date: 2022-12-02T22:23:00+00:00
author: "Emily Selwood"
draft: false
slug: "plotting"
tags: ["geospatial", "plotting"]
comments: false     # set false to hide Disqus comments
share: true        # set false to share buttons
ShowPostNavLinks: true
---

As useful as data is on its own, its really important for people to be able to see it. For a recent project I needed to show some [World Cover](https://esa-worldcover.org/en) images with the area of interest marked, along side the table of results. People respond better to being able to see it. This how I did that.



Lets start with a list of tools we used:

* GeoPandas
* Rasterio
* MatPlotLib

First we need to open our image data. I'm going to skip over fetching the data, there are pretty good examples on the [World Cover data page](https://esa-worldcover.org/en/data-access) we have a variable `data_path` that is a filesystem path to our cropped tiff file. A word of warning about the WMS server that has the world cover data is not suitable for pixel counting types of analysis. There are colour mixing artifacts between the classes.

```python
    import rasterio
    ds = rasterio.open(data_path)
    array = ds.read()
```

A fun thing about the world cover data is that it is a classification. The input contains a single band where each number means a different land cover type. Things like "forest" or "urban" etc. This isn't your traditional colours, so plotting it as is won't give us a nice result, just shades of dark gray. We need to create a colour map in MatPlotLib parlance.

```python
    import matplotlib.colors as colors
    from matplotlib.colors import ListedColormap

    world_cover_colours_2021 = [
        '#FFFFFF', '#006400', '#ffbb22', '#ffff4c', '#f096ff', '#fa0000', 
        '#b4b4b4', '#f0f0f0', '#0064c8', '#0096a0', '#00cf75', '#fae6a0' 
    ]
    world_cover_codes = [
        0, 10, 20, 30, 40, 50, 60, 60, 70, 80, 90, 95, 100
    ]
    world_cover_cmap = ListedColormap(world_cover_colours_2021)
    world_cover_norm = colors.BoundaryNorm(world_cover_codes, 12)
```

Next we need our area of interest to plot over the top of the image we will create from the land cover data. Ours was contained in a pair of GeoPandas Data Frames called `location_frame` and `area_frame`. How we got those is outside the scope of this post.

Now that we have all our data and our colour maps setup we can start to plot our data. For this we are going to use MatPlotLib. This is a graphing library. What we are about to create would be a very bad chart, but it has all the tools we need to do it so it makes it easy for us. First we need to create a `figure` and `axis` for our "graph". To do that we need to know the size we want. One wrinkle is this might not be square. So we work this out as a ratio of the size of our source image.

```python
    import matplotlib.pyplot as plt

    dpi = 96

    ratio = array.shape[1] / array.shape[2]
    width = 400 / dpi
    height = (400  * ratio) / dpi

    fig = plt.figure()
    fig.set_size_inches((width, height))
    ax = plt.Axes(fig, [0.0, 0.0, 1.0, 1.0])
    ax.set_axis_off()
    fig.add_axes(ax)
```

The `set_size_inches` method on the figure uses inches funnily enough so we need to know the Dots per inch (DPI) we want to be able to calculate the size we need. Once we've set the size of the figure, which will be the size we want as a ratio of our data images shape. We create an axes, a matplotlib figure can contain many charts so we have to create the axes separately. We tell it that this one will cover the entire image. Just before we add the axis to the figure we turn off the axis on the axes. This stops the markers down the side of the image.

Now we can start plotting the data. We will do the area of interest polygon first.

```python
    area_frame.to_crs("EPSG:4326").plot(ax=ax, facecolor='none')
```

Three things to note here. First the conversion of the CRS to match the source image data. If you don't do this things won't line up in the plot. MatPlotLib is not a geospatial library and doesn't understand coordinate systems at all. Second is passing in our Axes we created earlier. If you don't do this a brand new figure will be created inside this function and you won't get the output you expect at the end. Third the `facecolor` setting of none makes the polygon unfilled. If you want a shaded area you can set it to a semi transparent colour.

Next is the center point with an almost identical line of code.

```python
location_frame.to_crs("EPSG:4326").plot(ax=ax, c='r')
```

This geo data frame contains a point so will only show a single mark on the result. The `c='r'` bit is a short hand for setting the colour of the point to `r`ed.

Next we need to plot our image. To do this, there is one last thing we need to set up, so that our image ends up in the right place. Just like the polygons needing to be in the same CRS we need to tell matplotlib where our image sits. To do that we need the bounds of the image from the rasterio data source.

```python
    bounds = (ds.bounds.left, ds.bounds.right, ds.bounds.bottom, ds.bounds.top)
    ax.imshow(array[0,:], cmap=world_cover_cmap, norm=world_cover_norm, extent=bounds)
```

The `imshow` method on the Axes adds a bitmap image from the provided 2d array. The output of a rasterio array is always 3d so we have to slice off one off the first axis. Then we provide the colour map and normalization objects we set up earlier. Finally we provide the bounds to move our array into the right place on the chart.

With all of our data plotted, all that is left to do is save the figure, making sure we tell it our dpi. This will save the resulting chart to disk at `out_path`

```python
    plt.savefig(out_path, dpi=dpi)
```

I'll freely admit I'm mostly writing this so I have the notes next time. Hopefully this helps you. Even if you are future me. 