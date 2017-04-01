---
title: "R and the OSGeo4W package"
author: "@obrl_soil"
date: "2017-04-01"
layout: post
comments: true
permalink: /r-osgeo4w-windows/
categories: 
  - spatial
  - computer whispering
tags: 
  - R
  - OSGeo4W
  - QGIS
  - SAGA
  - GRASS
---



This is a setup post about getting R and the OSGeo4W package to talk to each other on Windows by setting [environment variables](https://en.wikipedia.org/wiki/Environment_variable) appropriately. R's built-in spatial capabilities have been drastically enhanced the last few years, but a lot of really useful tools remain external to R itself, and this is an easy way to get at lots of them. I've also been using R as a workflow recording and reporting tool as much as an actual data processing environment, so I find this setup extremely useful for reproducibility. 

This post can't be all things to all people, but it should work for most. Its been tested on standard Win10 and 10.1, plus a few workplace-specific builds of Win7 that have had god-knows-what done to them by my IT people, bless their uncommunicative, control-freaky hearts. I'll be updating this fairly regularly as requirements evolve (and as I learn more tricks!). I won't be helping debug your setup in the comments, though, post any problems on stackexchange where they belong.

***

My usual practice for a new setup is

  * Install R to C:\\R\\\*version\*
  * Install RStudio
  * Install RTools (not essential, but handy for compiling some packages from source code)
  
then make sure both RTools and R are on the system Path, fairly early in the list. If they don't appear there as a result of the install process, just paste in the following at the very start of the path, without altering what's already there, e.g.

    C:\RTools\bin;C:\RTools\mingw_32\bin;C:\R\R-3.3.3\bin;
    
Adjust R version number as needed.

Next, install the [OSGeo4W package](https://trac.osgeo.org/osgeo4w/wiki/WikiStart#QuickStartforOSGeo4WUsers) to its default location (mine is C:\\OSGeo4W64). Its easiest to run the automatic install first and then go back into setup under advanced mode to select any extras you want. After that, set all of the following environment variables:

In User variables:

  * GISRC = %USERPROFILE%\\AppData\\Roaming\\GRASS7\\rc
  * GRASS_ADDON_PATH = %USERPROFILE%\\AppData\\Roaming\\GRASS7\\addons
  
the first one tells the system where the GRASS rc file is. The rc file contains local settings for some GRASS startup variables, [which you can read about here](https://grass.osgeo.org/grass72/manuals/variables.html). This file won't exist until you open GRASS for the first time and set up a database location, but you can still set GISRC ahead of time. The second one tells the system where any [GRASS addon modules](https://grass.osgeo.org/grass72/manuals/addons/) you choose to install are located.

In System variables:

  * OSGEO4W_ROOT     = C:\\OSGeo4W64
  * GDAL_DATA        = %OSGEO4W_ROOT%\\share\\gdal
  * GDAL_DRIVER_PATH = %OSGEO4W_ROOT%\\bin\\gdalplugins
  * GEOTIFF_CSV      = %OSGEO4W_ROOT%\\share\\gdal
  * GISBASE          = %OSGEO4W_ROOT%\\apps\\grass\\grass-7.2.0
  * GRASS_PYTHON     = %OSGEO4W_ROOT%\\bin\\pythonw.exe 
  * PROJ_LIB         = %OSGEO4W_ROOT%\\share\\proj
  * PYTHONHOME       = %OSGEO4W_ROOT%\\apps\\Python27
  * PYTHONPATH       = %GISBASE%\\etc\\python
  * SAGA             = C:\\OSGeo4W64\\apps\\saga
  
These make your system aware of the versions of GDAL/OGR, GRASS, Python and SAGA that come with OSGeo4W. Adjust the version number in GISBASE as neccessary. You may also need to use -ltr for some of the apps if you installed those instead (e.g. saga-ltr). Note that setting SAGA to %OSGEO4W_ROOT%\\apps\\saga won't work.

Now edit the Path again - among the rest of what's already there, add the following, in this order:

  * %SAGA%
  * %SAGA%\\modules
  * %PYTHONHOME%
  * %OSGEO4W_ROOT%\\bin
  * %OSGEO4W_ROOT%\\apps\\qgis
  * %GISBASE%\\bin
  * %GISBASE%\\scripts
  * %GISBASE%\\lib
  
appending to the end of the Path should work, but you may have to make some adjustments depending on what-all else you have installed on your machine. 

You can test functionality in R with a few system2 calls, e.g.


```r
system2('gdal_translate')
system2('saga_gui')
system2('qgis-browser')
```

You should be able to access any of the command-line options that are listed when you open the OSGeo4W shell with the above method. The rgrass7 package should also initialise properly when loaded with `library()`. There is a wrapper package for SAGA (`RSAGA`)  as well, but tbh I find it easier to just use `system2('saga_cmd', args = c(...))`. IMO its not much of a wrapper package if you can't feed an R spatial object in directly...

You can also get at the QGIS processing toolbox via the `RQGIS` package, although its generally faster and simpler to access the underlying software directly, where possible. For example, a system call to gdalwarp is around 400 times faster than `RQGIS::run_qgis('gdalogr:warpreproject')` on my machine (and a lot simpler to write), but QGIS-only tools like 'Distance to nearest Hub' are probably worth a look.
