# mapmate

[![Travis-CI Build Status](https://travis-ci.org/leonawicz/mapmate.svg?branch=master)](https://travis-ci.org/leonawicz/mapmate)
[![Coverage Status](https://img.shields.io/codecov/c/github/leonawicz/mapmate/master.svg)](https://codecov.io/github/leonawicz/mapmate?branch=master)

`mapmate` (map animate) is an R package for map animation. It is used to generate and save a sequence of plots to disk as a still image sequence intended for later use in data animation production.

# Examples using mapmate

## Historical and projected global temperature anomalies

<a href="https://www.youtube.com/watch?v=xhqEkyJDBho"><img src="https://img.youtube.com/vi/xhqEkyJDBho/0.jpg" width="200"></a>

## Global UAF/SNAP Shiny Apps web traffic

<a href="https://www.youtube.com/watch?v=uQYR91qixgo"><img src="https://img.youtube.com/vi/uQYR91qixgo/0.jpg" width="200"></a>

## Flat map great circle animation example

<a href="https://www.youtube.com/watch?v=yoyIUMvIP3Q"><img src="https://img.youtube.com/vi/yoyIUMvIP3Q/0.jpg" width="200"></a>

# Introduction to mapmate
Matthew Leonawicz  
`r Sys.Date()`  



The `mapmate` package is used for map- and globe-based data animation pre-production.
Specifically, `mapmate` functions are used to generate and save to disk a series of map graphics that make up a still image sequence,
which can then be used in video editing and rendering software of the user's choice. This package does not make simple animations directly within R, which can be done with packages like `animation`.
`mapmate` is more specific to maps, hence the name, and particularly suited to 3D globe plots of the Earth.
Functionality and fine-grain user control of inputs and outputs are limited in the current package version.

This introduction covers the following toy examples:

- Generate a data frame containing monthly map data (optionally seasonal or annual aggregate average data)
in the form of an n-year moving or rolling average based on an input data frame of raw monthly data.
- Generate a sequence of still frames of:
    - map data for use in a flat map animation.
    - dynamic/temporally changing map data projected onto a static globe (3D Earth)
    - static map data projected onto rotating globe
    - dynamic map data projected onto rotating globe
    - non-map data (time series line growth)

Other features and functionality will be added in future package versions.

## Computing moving averages

With a data frame containing monthly map data in the following form, a data frame of moving averages can be computed using `get_ma`.
The n-year moving average window is controlled by `size`.
The window size pertains to years whether the original monthly data remain as monthly data or are summarized on the fly to seasonal or annual averages.


```r
library(mapmate)
library(dplyr)
library(purrr)
data(monthlytemps)
monthlytemps
get_ma(monthlytemps, type = "seasonal", season = "winter")
get_ma(monthlytemps, type = "annual", size = 20)
```

## Generate a still image sequence

To keep the data size minimal, the `monthlytemps` data set only includes data from one map cell, which is sufficient for the moving average example.
But for mapping we need more complete map data.
The `annualtemps` dataset below is not subsampled, though it is highly spatially aggregated as well as consisting of annual averages of monthly data.
This made it small enough to not be a burden to include in the package, but complete enough to use for mapping examples.


```r
data(annualtemps)
annualtemps
```

The function below saves png files to disk as a still image sequence. Set your working directory or be explicit about your file destination.
Make sure to obtain the range of data across not only space (a single map) but time (all maps).
This is to ensure a constant color to value mapping with your chosen color palette across all maps in the set of png files.
For simplicity, set the number of still frames to be equal to the number of unique time points in the data frame, in this one frame for each year.

Since each point in this data frame represents a coarse grid cell, we use `type="maptiles"` in the call to `save_map`.
`save_map` is used for tiles, polygon lines (`type="maplines"`) and great circle arcs (`type="network"`).
One of these three types must be specified and the data must be appropriate for the type.

Tiles are the simplest, but while we are working with data frames, the data typically would originate from a rasterized object, hence the nice even spacing of the coordinates.
Last of all, but important, the data frame must contain a `frameID` column.
Here we will just add a `frameID` column based on the unique years in the data frame.
For these static examples, arguments pertaining specifically to image sequences can be ignored for now (`n.period`, `n.frames`, `z.range`)
and since we are not saving files to disk we can also ignore arguments like `dir` and `suffix`, but we will return to all these immediately after.

### Initial static 2D and 3D map examples


```r
library(RColorBrewer)
pal <- rev(brewer.pal(11, "RdYlBu"))
temps <- mutate(annualtemps, frameID = Year - min(Year) + 1)
frame1 <- filter(temps, frameID == 1)  # subset to first frame

save_map(frame1, ortho = FALSE, col = pal, type = "maptiles", save.plot = FALSE, 
    return.plot = TRUE)
save_map(frame1, col = pal, type = "maptiles", save.plot = FALSE, return.plot = TRUE)
```

![2D flat map and 3D globe](mapmate_files/figure-html/unnamed-chunk-4-1.png)![2D flat map and 3D globe](mapmate_files/figure-html/unnamed-chunk-4-2.png)

### Flat map sequence

#### Dynamic data, static map

`save_map` is not typically used in the manner above. The above examples are just to show static plot output,
but there is no convenience in using `mapmate` for that when we can just use `ggplot2` or other plotting packages directly.
Below are some examples of how `save_map` is usually called as a convenient still image sequence generator.

Notice the importance of passing the known full range of the data.
The optional filename `suffix` argument is also used.
For a flat map we still do not need to pay attention to the globe-specific arguments (`lon`, `lat`, `rotation.axis`)
and those dealing with displaying data while globe rotation occurs across the image sequence can also be ignored (`n.period`).


```r
rng <- range(annualtemps$z, na.rm = TRUE)
n <- length(unique(annualtemps$Year))
suffix <- "annual_2D"
temps <- temps %>% split(.$frameID)
walk(temps, ~save_map(.x, ortho = FALSE, col = pal, type = "maptiles", suffix = suffix, 
    z.range = rng))

```

### Globe sequence

#### Dynamic data, static map

For plotting data on the globe, set `ortho=TRUE` (default).
In this example the globe remains in a fixed position and orientation.
The default when passing a single, scalar longitude value to the `lon` argument is to use it as a starting point for rotation.
Therefore, we need to pass our own vector of longitudes defining a custom longitude sequence to override this behavior.
Since the globe is to remain in a fixed view, set `lon=rep(-70, n)`.
Note that a scalar `lat` argument does not behave as a starting point for rotation in the perpendicular direction so not explicitly repeating vector is needed.

Also pay attention to `n.period`, which defines the number of frames or plot iterations required to complete one globe rotation (the degree increment).
`n.period` can be any amount when providing a scalar `lon` value. The longitude sequence will propagate to a length of `n.period`.
However, if providing a vector representing a custom path sequence of longitudes to `lon`, there is no assumption that the sequence is meant to be cyclical (do your own repetition if necessary).
It could just be a single pass, custom path to change the globe view.
In this case, `n.period` must be equal to the length of the `lon` and `lat` vectors or an error will be thrown.
This is more a conceptual restriction than anything.
With a custom path defined by `lon` and `lat` vectors, which may not be at all cyclical, it only makes sense to define the length of the sequence as the period itself.



```r
suffix <- "annual_3D_fixed"
walk(temps, ~save_map(.x, lon = rep(-70, n), lat = 50, n.period = n, n.frames = n, 
    col = pal, type = "maptiles", suffix = suffix, z.range = rng))

```

#### Static data, dynamic map

Now let the globe rotate, drawing static data on the surface.
In this example the nation borders are drawn repeatedly while allowing the Earth to spin through the still image sequence from frame to frame.
`walk` from the `purrr` package is used to easily duplicate the `borders` data frame in a list while adding a `frameID` column that increments over the list.

It's quite redundant and this is worse if the dataset is large, but this is how `save_map` currently accepts the data.
Fortunately, it would make little difference if the data change from frame to frame, which is the main purpose of `save_map`.
Datasets which do not vary from frame to frame generally serve the role of providing a background layer to other data being plotted,
as with the nation borders here and as we will see later with a bathymetric surface.

In the case of repeating data passed to `save_map`, it only makes sense when plotting on a globe and when using a perspective that has some kind of periodicity to it.
This limits how many times the same data must be redrawn.
For example, if the Earth completes one rotation in 90 frames, it does not matter that maps of time series data may be plotted over a course of thousands of frames.
When layering those maps on top of a constant background layer on the rotating globe, only 90 plots of the background data need to be generated.
Those 90 saved images can be looped in a video editor until they reach the end of the time series maps image sequence.

The plot type has changed to `maplines`.
`z.range` can be ignored because it only applies to map tiles (essentially square grid polygon fill coloring) which have values associated with surface pixels.
It is irrelevant for polygon outlines as well as for `type="network"` plots of line segment data.
A single color can be used for the lines so the previously used palette for tiles has been replaced.


```r
data(borders)
borders <- map(1:n, ~mutate(borders, frameID = .x))
suffix <- "borders_3D_rotating"
walk(borders, ~save_map(.x, lon = -70, lat = 50, n.period = 30, n.frames = n, 
    col = "orange", type = "maplines", suffix = suffix))

```

Using one more example, return to the list of annual temperature anomalies data frames used previously.
Those were used to show changing data on a fixed-perspective globe plot.
Here we plot the first layer, repeatedly, as was just done with the `borders` dataset, allowing the Earth to rotate.
Remember to update the `frameID` values.
The key difference with this example of fixed data and a changing perspective is that providing `z.range` is crucial to maintaining constant color mapping.


```r
temps1 <- map(1:n, ~mutate(temps[[1]], frameID = .x))
rng1 <- range(temps1[[1]]$z, na.rm = TRUE)
suffix <- "year1_3D_rotating"
walk(temps1, ~save_map(.x, lon = -70, lat = 50, n.period = 30, n.frames = n, 
    col = pal, type = "maptiles", suffix = suffix, z.range = rng1))

```

#### Dynamic data, dynamic map

Finally we have the case of changing data drawn on a map with a changing perspective.
This example plots the full time series list of annual temperature anomalies data frames with globe rotation.


```r
suffix <- "annual_3D_rotating"
walk(temps, ~save_map(.x, lon = -70, lat = 50, n.period = 30, n.frames = n, 
    col = pal, type = "maptiles", suffix = suffix, z.range = rng))

```

Of course, there is no reason to draw static data on a static flat map or static (e.g., non-rotating) globe
because that would yield a still image sequence of a constant image.
The utility of the function is in generating a series of plots where either the data changes, the perspective changes, or both.

### Multiple image sequences

Putting it all together, we can generate three still image sequences to subsequently be layered on top of one another in an animation.
Below is an example using a bathymetry surface map as the base layer, the temperature anomalies that will be the middle layer,
and the nation borders which will stack on top. This also is why the saved png images have a default transparent background.
There is an expectation of eventual layering.


```r
data(bathymetry)
bath <- map(1:n, ~mutate(bathymetry, frameID = .x))
rng_bath <- range(bath[[1]]$z, na.rm = TRUE)
pal_bath <- c("black", "steelblue4")

walk(bath, ~save_map(.x, n.frames = n, col = pal_bath, type = "maptiles", suffix = "background", 
    z.range = rng_bath))
walk(borders, ~save_map(.x, n.frames = n, col = "black", type = "maplines", 
    suffix = "foreground"))
walk(temps, ~save_map(.x, n.frames = n, col = pal, type = "maptiles", suffix = "timeseries", 
    z.range = rng))

```

### Parallel processing

For larger datasets and longer sequences, this is much faster the more it can be parallelized.
`mapmate` is designed to add convenience for making relatively heavy duty animations.
The emphasis is on images which will look sharp and lend themselves to frame smooth transitions.
They may also do so while displaying a large amount of data.
High-resolution images (even larger than 4K) can be used to allow zooming during animations without loss of quality.
For all these reasons, processing large amounts of data and generating still image sequences can take a long time
depending on how large, long, and complex the desired plots sequences are.

The toy examples given earlier are not ideal for serious production.
The code is provided for simple reproduction and tutorial purposes.
The `save_map` function can be used more powerfully on a Linux server with a large number of CPU cores,
and enough RAM to meet the needs of your data.
It uses `mclapply` from the base R `parallel` package.
It will not work on Windows.
The choice was made for convenience.

The example below is like the previous one, but using `mclapply`.
It assumes you have a 32-CPU Linux server node.
If you have multiple nodes, you could even go so far as to explore the `Rmpi` package to link across, say, 10 nodes to yield the power of 320 CPUs.
But if you have access to that kind of computing power, you probably already know it and can explore it on your own.
It is beyond the scope of this tutorial.


```r
mclapply(bath, save_map, n.frames = n, col = pal_bath, type = "maptiles", suffix = "background", 
    z.range = rng_bath, mc.cores = 32)
mclapply(borders, save_map, n.frames = n, col = "orange", type = "maplines", 
    suffix = "foreground", mc.cores = 32)
mclapply(temps, save_map, n.frames = n, col = pal, type = "maptiles", suffix = "timeseries", 
    z.range = rng, mc.cores = 32)

```
