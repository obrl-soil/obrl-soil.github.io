---
title: "Vectorising Rasters with R and GRASS"
author: "@obrl_soil"
date: "2017-06-04"
layout: post
permalink: /raster_to_polygons/
categories: 
  - spatial
tags: 
  - R
  - raster
  - vector
  - GRASS
  - polygon
---



In response to [this R-sig-geo email](https://stat.ethz.ch/pipermail/r-sig-geo/2017-June/025756.html), here's a workflow for turning a categorical raster into a polygon layer, with appropriate smoothing. This is one of those tasks I'm surprised isn't talked about more.

This requires a software environment set up according to [my post on getting R and GRASS talking](https://obrl-soil.github.io/r-osgeo4w-windows/), so do that stuff first. 


```r
library(sp)
library(sf)
library(raster)
library(rgdal)
library(rgrass7)
library(tidyverse)
library(rosm)
library(ggspatial)
library(ggsn)
library(viridis)

options(stringsAsFactors = FALSE)
```

I'm going to use a random chunk of model output from a project I'm currently working on. The raster categories are predicted soil classes, and there's a nodata area with an island in the file, so the data should be messy enough to provide a robust test of this method. 


```r
system2('gdalwarp', 
        args = c('-s_srs', 'EPSG:3577',
                 '-t_srs', 'EPSG:4326',
                 '-r', 'near', 
                 '-te', '1407921 -1884176 1436633 -1861473', 
                 '-te_srs', 'EPSG:3577',
                 paste0('"', srcfile, '"'),
                 '"C:/DATA/r2p/cat_clip.tif"'))
```


```r
cr <- raster("C:/DATA/r2p/cat_clip.tif") 
```

<img src="{{ site.url }}/images/raster-to_polygonsplot1-1.png" title="plot of chunk plot1" alt="plot of chunk plot1" style="display: block; margin: auto;" />

Set up a new GRASS location:


```r
initGRASS(gisBase  = "C:/OSGeo4W64/apps/grass/grass-7.2.1",
          gisDbase = getwd(),
          location = 'grassdata',
          mapset   = "PERMANENT", 
          override = TRUE)
```

```
## gisdbase    C:/DATA/r2p 
## location    grassdata 
## mapset      PERMANENT 
## rows        1 
## columns     1 
## north       1 
## south       0 
## west        0 
## east        1 
## nsres       1 
## ewres       1 
## projection  NA
```

The location's crs must be matched to the input dataset before import. Use [`g.proj`](https://grass.osgeo.org/grass72/manuals/g.proj.html):

```r
execGRASS("g.proj", 
          georef = file.path(getwd(), 'cat_clip.tif'),
          flags = c('t', 'c'))
```

And then the file can be imported with [`r.in.gdal`](https://grass.osgeo.org/grass72/manuals/r.in.gdal.html). Lastly, the region settings need to be updated with [`g.region`](https://grass.osgeo.org/grass72/manuals/g.region.html) to match the raster (nrows, ncols, origin etc).

```r
execGRASS('r.in.gdal',
          input  = file.path(getwd(), 'cat_clip.tif'),
          output = 'cat_clip',
          flags  = 'overwrite')

# to import a raster object already in R:
#writeRAST(cat_clip 'cat_clip')

execGRASS('g.region', raster = 'cat_clip')

# lets see what we've done:
gmeta()
```

```
## gisdbase    C:/DATA/r2p 
## location    grassdata 
## mapset      PERMANENT 
## rows        919 
## columns     971 
## north       -16.76522 
## south       -16.99697 
## west        145.2097 
## east        145.4547 
## nsres       0.0002521752 
## ewres       0.000252385 
## projection  +proj=longlat +no_defs +a=6378137 +rf=298.257223563
## +towgs84=0.000,0.000,0.000
```

So far, so good. I'll do a strict polygonisation with [`r.to.vect`](https://grass.osgeo.org/grass72/manuals/r.to.vect.html) first. 


```r
execGRASS('r.to.vect',
          input  = 'cat_clip',
          output = 'cat_v_raw',
          type   = 'area',
          flags  = c('v', 'overwrite'))

# pull the results into R (rgrass7 only supports sp)
cat_v_raw <- readVECT('cat_v_raw') %>% 
  st_as_sf() %>%
  st_cast('POLYGON', warn = FALSE)

# backup
st_write(cat_v_raw, file.path(getwd(), 'cat_v_raw.gpkg'), quiet = TRUE)
```

Here's a zoomed-in look:

<img src="{{ site.url }}/images/raster-to_polygonszoom1-1.png" title="plot of chunk zoom1" alt="plot of chunk zoom1" style="display: block; margin: auto;" />

This is, of course, the only way to produce a vector dataset that fully represents the source data, but a perfect analog is not always desirable. Small pixels and large extents can result in massive polygon counts. Products like this perform poorly and are difficult to work with, particularly on tablets without a pointer or stylus. A better alternative is to simplify the output map somewhat before converting to vector.

### Raster simplification

Before simplifying a raster, an appropriate minimum zone size must be chosen - this becomes the minimum acceptable polygon size. Choice of zone size is specific to whatever data you're working with. Since I'm playing with a soils map, I'm going be a good little soil scientist and pay attention to the Guidelines for Surveying Soil and Land Resources ([McKenzie et al, 2008, the 'Blue Book'](http://www.publish.csiro.au/book/5650/)).

According to these guidelines, soil map scale is determined from sample point density, using some well-established rules of thumb e.g. 8-16 points per sq km = ~1:50,000. This map was produced not by sampling in the real world, but by using a model algorithm that generates what are essentially 'virtual sites' at a rate (for this project) of ~15 sites per square kilometre. The algorithm has a floor of 15 sites per polygon for Reasons, so polygons smaller than 1 sq km are sampled at a higher density. I have examined some model sample point files and found a median sample rate of about 50 sites per polygon.

Referring to Table 3.1 of the Blue Book (p. 32), that median sample rate translates to a scale of ~1:10,000, and a minimum delineable area of 0.4 hectares. One pixel in this raster is approximately 0.075 hectares, so this means that I should remove any continuous areas of less than 6 pixels before vectorising.

There are plenty of other ways to pick a target zone size, do as you will. Now, to the actual smoothing:

For data in a projected crs, you can use the convenience wrapper [`r.reclass.area`](https://grass.osgeo.org/grass72/manuals/r.reclass.area.html), but since I'm using lat-long I have to take the long way. This is just a chain of [`r.to.vect`](https://grass.osgeo.org/grass72/manuals/r.to.vect.html), [`v.clean`](https://grass.osgeo.org/grass72/manuals/v.clean.html) and [`v.to.rast`](https://grass.osgeo.org/grass72/manuals/v.to.rast.html) with particular settings locked in.

Note that many people seem to think applying a moving window filter to a raster (modal, in this case) is the only way to smooth it out. I disagree, that tends to blur out too much detail. It's especially harsh on small linear features. The process below respects the source data better.


```r
# the first step (r.to.vect) was completed above
execGRASS('v.clean',
          input     = 'cat_v_raw',
          type      = 'area',
          output    = 'cat_v_smoothed',
          tool      = 'rmarea',
          threshold = 4000, # square meters
          flags     = c('overwrite'))

execGRASS( 'v.to.rast',
           input  = 'cat_v_smoothed',
           type   = 'area',
           output = 'cat_r_smoothed',
           use    = 'cat',
           flags  = 'overwrite')

## the above is equivalent to the below
#execGRASS('r.reclass.area',
#          input  = 'cat_clip',
#          output = 'cat_r_smoothed',
#          value  = 0.4,             
#          mode   = 'lesser',
#          method = 'rmarea',
#          flags  = c('overwrite'))

cat_v_smoothed <- readVECT('cat_v_smoothed') %>% 
  st_as_sf() %>%
  st_cast('POLYGON', warn = FALSE)

# backup
st_write(cat_v_smoothed, file.path(getwd(), 'cat_v_smoothed.gpkg'), quiet = TRUE)
```

The simplified vector map produced with v.clean above is usable, and we've dropped the number of polygons from 16225 to 3880, which is quite respectable (original raster data underneath for comparison):

<img src="{{ site.url }}/images/raster-to_polygonszoom2-1.png" title="plot of chunk zoom2" alt="plot of chunk zoom2" style="display: block; margin: auto;" />

That said, you can get a much nicer looking output from re-vectorising the smoothed raster, using the 's' flag.


```r
execGRASS('r.to.vect',
          input  = 'cat_r_smoothed',
          output = 'cat_v_pretty',
          type   = 'area',
          flags  = c('v', 's', 'overwrite'))

cat_v_pretty <- readVECT('cat_v_pretty') %>% 
  st_as_sf() %>%
  st_cast('POLYGON', warn = FALSE)

# backup
st_write(cat_v_pretty, file.path(getwd(), 'cat_v_pretty.gpkg'), quiet = TRUE)
```


<img src="{{ site.url }}/images/raster-to_polygonszoom3-1.png" title="plot of chunk zoom3" alt="plot of chunk zoom3" style="display: block; margin: auto;" />

The final product has 3600 features and looks like :

<img src="{{ site.url }}/images/raster-to_polygonsplot2-1.png" title="plot of chunk plot2" alt="plot of chunk plot2" style="display: block; margin: auto;" />

### Topology issues

Outside GRASS, the results of this workflow can have self-intersection problems, for instance:

<p align="center"><img src="{{ site.url }}/images/topofail.PNG" alt = "screencap of topological problem"  style="display: block; margin: auto;" /></p>

I feel like if this were easily fixed, a tool would exist already. A challenge for someone not-me!
