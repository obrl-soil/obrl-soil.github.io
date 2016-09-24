---
title: Arcpy bounding boxes
date: 2016-06-14T20:02:26+00:00
author: obrl_soil
layout: post
permalink: /arcpy-bounding-boxes/
dsq_thread_id:
  - 4909206425
categories:
  - Spatial
tags:
  - arcpy
  - ESRI
  - spatial data
---
Ok, so if you're an ESRI user and you want a bounding box for a single dataset, or a feature/set of features within a single dataset, there's the [Minimum Bounding Geometry](http://desktop.arcgis.com/en/arcmap/10.3/tools/data-management-toolbox/minimum-bounding-geometry.htm) tool for vector data (assuming you have 10.3 or above). Raster bounding boxes require more work, but not for any good reason.

There's not much around for generating bounding boxes over multiple datasets, unfortunately, which is stupid because extent parameters are so easily extracted from any spatial dataset. Sure, you could use MBG after mucking around for an hour creating individual bounding polygons for each of your datasets and then running [Dissolve](http://desktop.arcgis.com/en/arcmap/10.3/tools/data-management-toolbox/dissolve.htm) or something, but ugh. _Ugh_.

My problem today was that I had four (roughly) adjacent soil survey boundaries, and I wanted to run extracts from other overlapping datasets for Science Reasons. I wanted a simple clipping box that encompassed all of the survey areas, and I didn't need to be picky about restricting the clip to the actual, irregular, outlines. Below is the quick-and-dirty method I came up with, starting with [this guy's code](http://anothergisblog.blogspot.com.au/2011/07/creating-extent-polygons-using-arcpy.html) (hat tip!) and branching out.

The idea is that it will draw a box around anything you've added to the map window. Caveats: I haven't accounted for meridian crosses, haven't tested it on multiple data types, and haven't tested it on northern hemisphere data, because I don't need to care about any of that ¯\\\_(ツ)\_/¯

```python  

import arcpy  
from arcpy import env  
from arcpy import mapping  

# list layers  
mxd = mapping.MapDocument("CURRENT") # will work even if your project is unsaved  
layer_list = mapping.ListLayers(mxd) # lists everything in the ToC window  
sr = arcpy.SpatialReference(4283) # GDA_94  

# coordinates from each layer's bounding box will be placed in these lists  
N_list = []  
E_list = []  
S_list = []  
W_list = []  

for layer in layer_list:  
    extent = arcpy.Describe(layer).extent
    N_list.append(extent.YMax)  
    E_list.append(extent.XMax)  
    S_list.append(extent.YMin)  
    W_list.append(extent.XMin)  

# get the biggest/smallest (...extentiest?) values for each direction  
bound_N = max(N_list)  
bound_E = max(E_list)  
bound_S = min(S_list)  
bound_W = min(W_list)  
# create point objects for new BB  
point_UR = arcpy.Point(bound_E, bound_N)  
point_LR = arcpy.Point(bound_E, bound_S)  
point_LL = arcpy.Point(bound_W, bound_S)  
point_UL = arcpy.Point(bound_W, bound_N)  

# Create the bounding box, it will be saved in your default gdb  
boundPoly = "{}\\extent".format(env.scratchWorkspace)  
array = arcpy.Array()  
array.add(point_UR)  
array.add(point_LR)  
array.add(point_LL)  
array.add(point_UL)  
# ensure the polygon is closed  
array.add(point_UR)  
# Create the polygon object  
polygon = arcpy.Polygon(array, sr)  
array.removeAll()  
# save to disk  
arcpy.CopyFeatures_management(polygon, boundPoly)  
del polygon
```

Usage instructions for noobs: Open everything you want to box up, click Geoprocessing > Python Console, paste in the above, hit enter, profit.

After I was done I buffered the bounding polygon by 250m or so because I'll be running some raster analyses that have wacky edge effects. You should probably do something similar.
