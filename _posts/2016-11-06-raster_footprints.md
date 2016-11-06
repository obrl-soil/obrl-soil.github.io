---
title: "Raster Footprints with R and GRASS"
author: "@obrl_soil"
date: "2016-11-06"
layout: post
permalink: /raster-footprints/
categories: 
  - spatial
tags: 
  - R
  - raster
  - GRASS
  - polygon
---



In response to [this stackexchange question](http://gis.stackexchange.com/questions/216624/creating-a-shapefile-for-footprint-of-rasters-valid-data-areas-with-gdal), here's a workflow for getting vector outlines of non-NULL areas in rasters, using GRASS 7 to do the hard work and R to handle batching. This whole thing is really hacky, but there don't seem to be any other solutions around at present.

## Setup

I'm working with the OSGeo4W64 package, R 3.3.1 and RStudio in a Windows 10 environment.

### Data prep

Get all your target rasters into one folder. Make sure that:

* files are named in a logical sequence that does not start with a number.
* files are all in the same projection
* files should be spatially aligned; i.e. the pixels should be all on the same overarching grid system.

Files can be in any GDAL/OGR compatible format, although I've only tested this with .tif and .asc. I'm using a few chunks of [Geoscience Australia's 1" SRTM DEM-H](http://www.ga.gov.au/metadata-gateway/metadata/record/gcat_72759) today. 

### R/GRASS interface

GRASS and R need a little help to speak to each other. Go to System > Advanced System Settings > Environment Variables and set the following:

* GDAL_DATA: C:\\OSGeo4W64\\share\\gdal
* GISBASE: C:\\OSGeo4W64\\apps\\grass\\grass-7.0.5
* GISRC: C:\\Users\\obrl_soil\\AppData\\Roaming\\GRASS7\\rc
* GRASS_PYTHON: C:\\OSGeo4W64\\bin\\pythonw.exe
* PROJ_LIB: C:\\OSGeo4W64\\share\\proj
* PYTHONPATH: C:\\OSGeo4W64\\apps\\grass\\grass-7.0.5\\etc\\python
* PATH: add the following paths, in the same order as below
    + C:\\OSGeo4W64\\bin
    + C:\\OSGeo4W64\\apps\\grass\\grass-7.0.5\\bin
    + C:\\OSGeo4W64\\apps\\grass\\grass-7.0.5\\lib
    + C:\\OSGeo4W64\\apps\\qgis
    + C:\\OSGeo4W64\\apps\\Python27

If GRASS\_ADDON\_BASE is set on your machine, remove it for now. Also, if you have standalone versions of anything in the OSGeo4W installer, e.g. another version of Python or GDAL, consider carefully where you place the OSGeo4w paths. The path variable is scanned in order of entry. Just quietly, it was bit of a nightmare getting all that correct. I have suffered so you don't have to!

### GRASS setup

Open GRASS and [establish a new location](https://grasswiki.osgeo.org/wiki/GRASS_Location_Wizard) with the same CRS as your target rasters. Don't worry about region settings for now, and there's no need to create any extra mapsets beyond 'PERMANENT'. Open the PERMANENT mapset in your new location, and then just exit GRASS. This will update the file that `GISRC` points to.

### R setup

Open Rstudio and make sure the packages 'sp', 'raster', 'rgdal', 'XML', and 'rgrass7' are installed. I have them, and am ready to process.

### Processing


```r
library(sp)
library(raster)
library(rgdal)
library(rasterVis)
library(ggplot2)
library(XML)
library(rgrass7)
```

The console output will indicate a connection to the GRASS location you just created. Now, set your R working directory to your raster folder, and list the files available:


```r
setwd('C:/footprint_test')
rasters <- list.files(getwd(), pattern = '\\.tif$', full.names = T)
```
They look like this:

<img src="{{ site.url }}/images/raster-footprintsrplots-1.png" title="plot of chunk rplots" alt="plot of chunk rplots" style="display: block; margin: auto;" />

Props to [Andrew Tredennick](https://nrelscience.org/2013/05/30/this-is-how-i-did-it-mapping-in-r-with-ggplot2/) for the ggplot method used above, although I'd be careful with very large rasters.

Next, import all the rasters you've listed into your GRASS location using [r.in.gdal](https://grass.osgeo.org/grass73/manuals/r.in.gdal.html); they will be stored in GRASS' native data format under the mapset PERMANENT.


```r
grassnames_list <- list() 
for (item in rasters) {
  grass_name <- substr(basename(item), 1, nchar(basename(item)) - 4)
  execGRASS('r.in.gdal', 
            flags = 'overwrite', 
            parameters = list(input  = item,
                              output = grass_name))
  grassnames_list[[item]] <- grass_name
}
```

Now use [g.region](https://grass.osgeo.org/grass73/manuals/g.region.html) to set region extents to cover all the files you just imported. 


```r
execGRASS('g.region', parameters= list(raster=paste(grassnames_list, collapse=",")))
```

I'm using [r.mapcalc](https://grass.osgeo.org/grass73/manuals/r.mapcalc.html) to make data/nodata versions of each raster; its very powerful but not accessible via rgrass7 for some reason, so we're just going to make a direct system call. This'll only work if you provide instructions in a text file, so that needs to be generated first, using some string manipulation shenanigans.


```r
exp_list <- list()
fpname_list <- list()
for (item in rasters) {
  grassname <- substr(basename(item), 1, nchar(basename(item)) -4)
  fp_name   <- paste0(grassname, '_foot')
  exp       <- paste0(fp_name, ' = if(isnull(', grassname, '), null() , 1)')
  
  exp_list[[item]]    <- exp
  fpname_list[[item]] <- fp_name
}

write(c(paste(exp_list, collapse = '\n'), 'end'),
      file = paste0(getwd(), '/rmc_instructions.txt'),
      ncolumns = 1)
```

Open the output *.txt if you want to see an example of r.mapcalc's map algebra format. Run that:


```r
rmc   <- 'C:/OSGeo4W64/apps/grass/grass-7.0.5/bin/r.mapcalc.exe'
calcs <- 'C:/footprint_test/rmc_instructions.txt'
system2(rmc, paste0('file=', calcs, ' --overwrite'))
```

Time to vectorise those footprints with [r.to.vect](https://grass.osgeo.org/grass72/manuals/r.to.vect.html):


```r
vname_list <- list()
for (fp in fpname_list) {
  v_name <- paste0(fp, '_v')
  execGRASS('r.to.vect', 
            flags = 'overwrite',
            parameters = list(input  = fp,
                              output = v_name,
                              type   = 'area'))
  vname_list[[fp]] <- v_name
}
```

If you open your mapset now (or look at it in the QGIS browser, assuming you have the GRASS plugin activated), you'll see your imported rasters, the footprints for each, and a vectorised version of the latter. Nothing left to do now but export with [v.out.ogr](https://grass.osgeo.org/grass72/manuals/v.out.ogr.html). There's a wide choice of export formats available, just demo-ing with shp:


```r
outname_list <- list()
for (vn in vname_list) {
  out_name <- paste0(getwd(), '/', vn, '.shp')
  execGRASS('v.out.ogr', 
            flags = 'overwrite', 
            parameters = list(input  = vn,
                              output = out_name,
                              format = 'ESRI_Shapefile'))
  outname_list[[vn]] <- out_name
}
```

Output files will have multiple polygons if there was more than one 'island' of data in the input. If you want one multipolygon, use [v.dissolve](https://grass.osgeo.org/grass72/manuals/v.dissolve.html) (this is why I had to set all those python environment variables). Note that you'll see a lot of windows flashing up as v.dissolve runs, don't panic. After that, use flag -m with `v.out.ogr`:


```r
dissname_list <- list()
for (vn in vname_list) {
  diss_name <- paste0(vn, '_sp')
  execGRASS('v.dissolve', 
            flags = 'overwrite',
            parameters = list(input  = vn,
                              column = 'value',
                              output = diss_name))
  dissname_list[[vn]] <- diss_name
}

outname_list2 <- list()
for (dn in dissname_list) {
  out_name2 <- paste0(getwd(), '/', dn, '.shp')
  execGRASS('v.out.ogr', 
            flags = c('m', 'overwrite'), 
            parameters = list(input  = dn,
                              output = out_name2,
                              format = 'ESRI_Shapefile'))
  outname_list2[[dn]] <- out_name2
}
```

End result:
<img src="{{ site.url }}/images/raster-footprintsoutput-1.png" title="plot of chunk output" alt="plot of chunk output" style="display: block; margin: auto;" />

You can find all the files related to this post in [https://github.com/obrl-soil/bits-n-pieces/footprint_test](https://github.com/obrl-soil/bits-n-pieces/tree/master/footprint_test).
