---
title: "Cleaning polygonised raster data"
author: "@obrl_soil"
date: "2017-09-22"
layout: post
permalink: /cleaning-polygons-internal/
categories: 
  - spatial
tags: 
  - R
  - vector
  - polygon
---



In my last post, I put together a workflow for [polygonising categorical raster data](https://obrl-soil.github.io/raster_to_polygons/) that gives nice-looking outputs. As mentioned though, they still need a bit of tidying-up before they can be used.

There are actually two problems with the data I generated. Most importantly, they're incorrectly defined once exported from GRASS - while not corrupt, they aren't valid under the Simple Features standard. This won't mess with data display, but can potentially affect further analyses. The other issue is that the `r.to.vect` algorithm doesn't give the most efficient outputs - it creates a lot of redundant vertices, by which I mean vertices that don't significantly change the bearing of a polygon ring segment. These can slow down subsequent analysis, but more importantly they're taking up disk space where I could be storing hilarious cat gifs. The internet can be...somewhat unhelpful about these issues, so let's work through them in in turn.

<!--more-->


```r
library(sp)
library(sf)
library(tidyverse)
library(ggspatial)
library(gridExtra)
library(emo)

options(stringsAsFactors = FALSE)
```

### Correcting polygons

Google 'fix polygon topology' or similar, and you'll get a whole lot about managing inter-polygon issues (e.g. unwanted gaps and overlaps) but not much about intra-polygon. Its a tricky topic given that not all file formats follow the same standard, so maybe that's fair enough. A great place to start is Edzer Pebesma's blog post [Tidying feature geometries with sf](http://r-spatial.org/r/2017/03/19/invalid.html) from earlier this year, plus all the linked materials. Unfortunately the `lwgeom` topology library doesn't accompany `sf` on Windows yet, so I have to look elsewhere for help.

Lucky for me, some very clever folks from the [3D geoinformation research group at TU Delft](https://3d.bk.tudelft.nl/) have come up with a little command-line app that'll correctly rebuild just about any polygon, no matter how badly borked. Its called ['prepair'](https://github.com/tudelft3d/prepair). Click through to github and go through the readme and linked documents to find out how it works.

Installation is easy, just grab the appropriate binary off github and unzip it somewhere sensible. I'm going to make a really horrible corrupt polygon and use it to demo the app and why its important. R is a really easy place to make terrible spatial data:


```r
bad_poly <- matrix(c(0.5, 0.5,
                     0.4, 5.2,
                     9.8, 1.1,
                     7.6, 8.5,
                     9.5, 8.0,
                     9.4, 6.1,
                     7.4, 0.7,
                     0.5, 0.5), ncol = 2, byrow = TRUE) %>%
  st_polygon(x = list(.)) %>%
  st_sfc() %>%
  st_sf()

ggplot() + 
  geom_spatial(data = as(bad_poly, 'Spatial'), alpha = 0.5) + 
  ggtitle("Seems legit", subtitle = '...but is it?') +
  theme(axis.title.x=element_blank(),axis.title.y=element_blank()) +
  coord_equal()
```

<img src="{{ site.url }}/images/cleaning-polygons-internalbadpoly-1.png" title="plot of chunk badpoly" alt="plot of chunk badpoly" style="display: block; margin: auto;" />

\*shrugs\* One polygon, looks like three triangles. What could go wrong? Let's buffer it and see what happens.


```r
ggplot() + 
  geom_spatial(data = as(bad_poly, 'Spatial'), alpha = 0.5) +
  geom_spatial(data = as(st_buffer(bad_poly, dist = 0.5), 'Spatial'),
               fill = '#277d8e', alpha = 0.3) +
  ggtitle('Oh heck', subtitle = 'oh dang') +
  theme(axis.title.x=element_blank(), axis.title.y=element_blank()) +
  coord_equal()
```

<img src="{{ site.url }}/images/cleaning-polygons-internalbpbuff-1.png" title="plot of chunk bpbuff" alt="plot of chunk bpbuff" style="display: block; margin: auto;" />

Well that didn't work. Only part of the polygon got buffered! The middle bit is being ignored. Why is that... let's use a little ggplot magic to break it down:


```r
bps <- st_geometry(bad_poly) %>%
  unlist(., recursive = FALSE) %>%
  as.data.frame() %>%
  mutate(X1END = lead(X1), X2END = lead(X2))

ggplot() +
  geom_segment(data = bps, aes(x = X1, y = X2, xend = X1END, yend = X2END),
               arrow =  arrow(angle = 15, length = (unit(0.15, "inches")), type = 'closed'),
               na.rm = TRUE, col = 'grey35') + 
  geom_point(data = bps, aes(x = X1, y = X2), col = 'darkred') +
  geom_point(data = bps[1, ], aes(x = X1, y = X2), col = 'red', size = 2) +
  ggtitle('Bad Polygon', subtitle = '...exposed!!') +
  theme(axis.title.x = element_blank(), axis.title.y = element_blank()) +
  coord_equal()
```

<img src="{{ site.url }}/images/cleaning-polygons-internalbp_ggseg-1.png" title="plot of chunk bp_ggseg" alt="plot of chunk bp_ggseg" style="display: block; margin: auto;" />

See the missing dots where the lines cross? A valid geometry would (at least) have extra vertices there. The buffer algo doesn't quite know what to do without them.

Funnily enough the first cleanup trick used by many spatial data practitioners is to buffer a polygon by distance = 0. The buffering algorithm breaks down and rebuilds polygons as a side effect, solving a lot of minor issues without much computational overhead. In this case though, it can't catch what's wrong with this polygon. If we look at the geometry before buffering: 


```r
st_as_text(st_geometry(bad_poly))
```

```
## [1] "POLYGON ((0.5 0.5, 0.4 5.2, 9.8 1.1, 7.6 8.5, 9.5 8, 9.4 6.1, 7.4 0.7, 0.5 0.5))"
```
and after: 

```r
st_as_text(st_geometry(st_buffer(bad_poly, dist = 0L)))
```

```
## [1] "MULTIPOLYGON (((0.5 0.5, 0.4 5.2, 7.86132971506106 1.94559023066486, 7.4 0.7, 0.5 0.5)), ((8.79730134932534 4.47271364317841, 7.6 8.5, 9.5 8, 9.4 6.1, 8.79730134932534 4.47271364317841)))"
```

we can see that `st_buffer` found the coordinates of those two 'self-intersections' and used them to rebuild the polygon in the best way it can (also by casting it as a multipolygon, which is the other problem with our Bad Polygon - its the wrong geometry type for a shape that complex). This means ignoring that middle triangle though, which is not what we want. 

Incidentally, the area of the unbuffered polygon is 22.795, which is smaller than the 0-buffered one (25.6403538). This means both area calculations are wrong. The latter area must be from both of the multipolygon sub_areas seen above, so the former area must be only the bottom-left sub-area. Double dang!

Prepair does a much better job:


```r
bad_wkt <- st_as_text(st_geometry(bad_poly))

good_wkt <- system2('C:/prepair_win64/prepair.exe', 
                     args = c('--wkt', paste0('"', bad_wkt, '"')), 
                     stdout = TRUE)
```
What does Bad Polygon look like now?

```r
good_wkt
```

```
## [1] "MULTIPOLYGON (((0.4 5.2,0.5 0.5,7.4 0.7,7.861329715061059 1.945590230664858,0.4 5.2)),((7.6 8.5,8.797301349325338 4.472713643178411,9.4 6.1,9.5 8.0,7.6 8.5)),((7.861329715061059 1.945590230664858,9.8 1.1,8.797301349325338 4.472713643178411,7.861329715061059 1.945590230664858)))"
```
Nice, this time the output is a multipolygon with the three components I expect. 
<img src="{{ site.url }}/images/cleaning-polygons-internalresf-1.png" title="plot of chunk resf" alt="plot of chunk resf" style="display: block; margin: auto;" />

Last check, the area is now 28.4857075, as it should be.

So that's cool! Looking back at the dataset I created in my last post though, QGIS says I have over 70 invalid geometries to repair, and prepair is a one-poly-at-a-time app. There is a version called [pprepair](https://github.com/tudelft3d/pprepair) that operates across whole datasets, but... its not available for Windows ;_;

On the bright side, the Bad Polygons produced by polygonising a raster per my last post are actually fixable using buffer-by-0:


```r
cat_v_pretty <- read_sf(file.path(getwd(), 'cat_v_pretty.gpkg')) %>%
  mutate(ID   = 1:nrow(.))

cvp_b0 <- cat_v_pretty %>%
  # buffer expects projected coords
  st_transform(., 28355) %>% 
  st_buffer(., dist = 0L) %>% 
  st_union(., by_feature = TRUE) %>% 
  st_transform(., 4326)
```

I can demo this with the smallest error, `cat_v_pretty[1716, ]`:

<img src="{{ site.url }}/images/cleaning-polygons-internalp1716-1.png" title="plot of chunk p1716" alt="plot of chunk p1716" style="display: block; margin: auto;" />

Buffer-0 works here because the errors are relatively simple - the geometry may be invalid, but it isn't corrupt the way my toy example above was. The [OGC simple features polygon spec](http://www.opengeospatial.org/standards/sfa) says that the exterior ring is supposed to be defined first, then any subsequent rings are interpreted as holes. The rings should also be stored as a list of matrices, one per ring. My invalid polygons are instead stored as single matrices, and the inner ring vertices may come first. My data plots well only because the sf plot method for polygons uses `graphics::polypath`, which has rules for figuring out fill areas that don't consider the above. There's no guarantee other functions in R (or other GIS software) will interpret these polygons the same way.

The 0-buffered dataset passes QGIS' topology checks. There are still the redundant vertices to deal with, though - you can see them on the diagonals above.

### 'Lossless' vertex removal

Lossless, or shape-preserving, simplification means different things to different people, weirdly enough. Its not always clear what 'redundant' vertices are, for one thing, or how much shape change is allowable in a given dataset. 

I'm operating in a pretty specific context: my polygons are generated by an algorithm that creates line segments in only 8 possible directions - or it should, but it struggled on the diagonals. The source data is also fine-scaled, so all my polygons are made of very short lines. This means that if I can get the bearing of each polygon segment, I can use it as a filter - and I don't have to be very precise about that bearing.

Most simplification algorithms use a distance-based method of some kind, and while they're efficient, they aren't really capable of distinguishing vertex signal from noise in the way I need. I'm going to have to go it alone. 

Below is a function that will work on sfc_POLYGON objects, even the mildly invalid ones shown above:


```r
library(geosphere)

clean_verts <- function(x = NULL, dir_tol = 0) {
  g <- st_geometry(x)
  
  message(paste0(Sys.time(), ': removing redundant vertices...'))
  clean_g <- lapply(g, function(geometry) {
    # get a list of polygon subgeometries, if present (they're x,y matrices)
    lapply(geometry, function(subgeometry) {
      # for each, convert to table of points
      data.frame(subgeometry) %>%
        # turn that into a table of line segments 
        mutate(LX1 = lead(X1), LX2 = lead(X2)) %>%
        filter(!is.na(LX1)) %>% 
        # remove duplicates (0-length segments)
        filter(!(X1 == LX1 & X2 == LX2)) %>%
        rowwise() %>%
        # get the direction of each segment (to the nearest degree)
        mutate(DIR = round(geosphere::bearingRhumb(c(X1, X2), c(LX1, LX2)))) %>%
        ungroup() %>%
        # tag segments that run in the same dir as the one previous
        mutate(DEL = ifelse(DIR == lag(DIR), 1, 0),  
               DEL = ifelse(is.na(DEL), 0, DEL)) %>%
        # and chuck them out
        filter(DEL == 0) %>% 
        # get original points
        select(X1, X2) %>%
        # repeat point 1 at end of table to close ring
        add_row(X1 = head(.$X1, 1), X2 = head(.$X2, 1)) %>%
        as.matrix(., ncol = 2, byrow = TRUE)
    })}) 
  message(paste0(Sys.time(), ': ...complete')) 
  # spatialise cleaned data
  clean_g_sf <- lapply(clean_g, function(geom) {
     st_multipolygon(list(geom)) 
  }) %>%
    st_sfc() %>%
    st_sf()
}

# test!
p1716_cv <- clean_verts(cat_v_pretty[1716, ])
```

Great success, this looks much better:

<img src="{{ site.url }}/images/cleaning-polygons-internalcv_plot-1.png" title="plot of chunk cv_plot" alt="plot of chunk cv_plot" style="display: block; margin: auto;" />

### Wrapping it all up

So I've got the components of my cleaning routine sorted, now I just have to plug them together and make them work over a whole dataset. I'm going to run prepair across all my polygons too, just to be thorough:


```r
clean_verts_n <- function(x = NULL, dir_tol = 0) {
  g <- st_geometry(x)
  
  # remove redundant vertices
  message(paste0(Sys.time(), ': removing redundant vertices...'))
  clean_g <- lapply(g, function(geometry) {
    lapply(geometry, function(subgeometry) {
      data.frame(subgeometry) %>%
        mutate(LX1 = lead(X1), LX2 = lead(X2)) %>%
        filter(!is.na(LX1)) %>%
        filter(!(X1 == LX1 & X2 == LX2)) %>%
        rowwise() %>%
        mutate(DIR = round(geosphere::bearingRhumb(c(X1, X2), c(LX1, LX2)))) %>%
        ungroup() %>%
        mutate(DEL = ifelse(DIR == lag(DIR), 1, 0),  
               DEL = ifelse(is.na(DEL), 0, DEL)) %>%
        filter(DEL == 0) %>% 
        select(X1, X2) %>%
        add_row(X1 = head(.$X1, 1), X2 = head(.$X2, 1)) %>%
        as.matrix(., ncol = 2, byrow = TRUE)
    })}) 
  message(paste0(Sys.time(), ': ...complete')) 
  
  # spatialise cleaned data
  clean_g_sf <- lapply(clean_g, function(geom) {
     st_multipolygon(list(geom)) 
  }) %>% 
    st_sfc()
  
  message(paste0(Sys.time(), ': Running prepair...'))
  
  # now fix internal topology with prepair - this returns list of wkts
  pr_g <- lapply(seq_along(clean_g_sf), function(row) {
    gt  <- st_as_text(clean_g_sf[[row]])
    
    # some wkts are too long to send to the command line >.>
    gpr <- if(nchar(gt) < 7500) {
      system2('C:/prepair_win64/prepair.exe', 
              args = c('--wkt', paste0('"', gt, '"')),
              stdout = TRUE)
    } else {
      write(gt, file.path(getwd(), 'temp.txt'))
      out <- system2('C:/prepair_win64/prepair.exe', 
                     args = c('-f', file.path(getwd(), 'temp.txt')),
                     stdout = TRUE)
      do.call('paste0', as.list(out))
    }
    #message(paste0("fixed ", row)) # for testing
    gpr
  })
  message(paste0(Sys.time(), ': ...complete'))
  # turn list of wkts back into a geometry column and replace the input
  prsf <- st_as_sfc(pr_g)
  st_geometry(x) <- prsf
  st_crs(x) <- st_crs(g)
  x
}

cvp_cvpr <- clean_verts_n(cat_v_pretty)
```


This is pretty kludgey and slow, but it gets the job done. It handles around 200 polygons/minute on my test machines. I wanted to be able to run prepair selectively to speed things up, but it does sometimes return vertices with higher precision than the input data, and that can cause topology check failures later.

<img src="{{ site.url }}/images/cleaning-polygons-internalcvp_pr-1.png" title="plot of chunk cvp_pr" alt="plot of chunk cvp_pr" style="display: block; margin: auto;" />

My final dataset looks identical to the input on-screen, passes its topology checks and is about a third smaller on disk (in GPKG format). I can now be confident that the data I'm generating is as robust and portable as possible.

***
