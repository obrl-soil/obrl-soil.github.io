---
id: 15
title: Arcpy bounding boxes
date: 2016-06-14T20:02:26+00:00
author: ob rl
layout: post
guid: http://www.leobrien.net/?p=15
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
Ok, so if you&#8217;re an ESRI user and you want a bounding box for a single dataset, or a feature/set of features within a single dataset, there&#8217;s the <a href="http://desktop.arcgis.com/en/arcmap/10.3/tools/data-management-toolbox/minimum-bounding-geometry.htm" target="_blank">Minimum Bounding Geometry</a> tool for vector data (assuming you have 10.3 or above). Raster bounding boxes require more work, but not for any good reason.

There&#8217;s not much around for generating bounding boxes over multiple datasets, unfortunately, which is stupid because extent parameters are so easily extracted from any spatial dataset. Sure, you could use MBG after mucking around for an hour creating individual bounding polygons for each of your datasets and then running <a href="http://desktop.arcgis.com/en/arcmap/10.3/tools/data-management-toolbox/dissolve.htm" target="_blank">Dissolve</a> or something, but ugh. _Ugh_.

My problem today was that I had four (roughly) adjacent soil survey boundaries, and I wanted to run extracts from other overlapping datasets for Science Reasons. I wanted a simple clipping box that encompassed all of the survey areas, and I didn&#8217;t need to be picky about restricting the clip to the actual, irregular, outlines. Below is the quick-and-dirty method I came up with, starting with <a href="http://anothergisblog.blogspot.com.au/2011/07/creating-extent-polygons-using-arcpy.html" target="_blank">this guy&#8217;s code</a> (hat tip!) and branching out.

The idea is that it will draw a box around anything you&#8217;ve added to the map window. Caveats: I haven&#8217;t accounted for meridian crosses, haven&#8217;t tested it on multiple data types, and haven&#8217;t tested it on northern hemisphere data, because I don&#8217;t need to care about any of that ¯\_(ツ)_/¯

<!--more-->

`import arcpy<br />
from arcpy import env<br />
from arcpy import mapping</p>
<p># list layers<br />
mxd = mapping.MapDocument("CURRENT") # will work even if your project is unsaved<br />
layer_list = mapping.ListLayers(mxd) # lists everything in the ToC window<br />
sr = arcpy.SpatialReference(4283) # GDA_94</p>
<p># coordinates from each layer's bounding box will be placed in these lists<br />
N_list = []<br />
E_list = []<br />
S_list = []<br />
W_list = []</p>
<p>for layer in layer_list:<br />
&nbsp;&nbsp;&nbsp;&nbsp;extent = arcpy.Describe(layer).extent<br />
&nbsp;&nbsp;&nbsp;&nbsp;N_list.append(extent.YMax)<br />
&nbsp;&nbsp;&nbsp;&nbsp;E_list.append(extent.XMax)<br />
&nbsp;&nbsp;&nbsp;&nbsp;S_list.append(extent.YMin)<br />
&nbsp;&nbsp;&nbsp;&nbsp;W_list.append(extent.XMin)</p>
<p># get the biggest/smallest (...extentiest?) values for each direction<br />
bound_N = max(N_list)<br />
bound_E = max(E_list)<br />
bound_S = min(S_list)<br />
bound_W = min(W_list)</p>
<p># create point objects for new BB<br />
point_UR = arcpy.Point(bound_E, bound_N)<br />
point_LR = arcpy.Point(bound_E, bound_S)<br />
point_LL = arcpy.Point(bound_W, bound_S)<br />
point_UL = arcpy.Point(bound_W, bound_N)</p>
<p># Create the bounding box, it will be saved in your default gdb<br />
boundPoly = "{}\\extent".format(env.scratchWorkspace)<br />
array = arcpy.Array()<br />
array.add(point_UR)<br />
array.add(point_LR)<br />
array.add(point_LL)<br />
array.add(point_UL)<br />
# ensure the polygon is closed<br />
array.add(point_UR)<br />
# Create the polygon object<br />
polygon = arcpy.Polygon(array, sr)<br />
array.removeAll()<br />
# save to disk<br />
arcpy.CopyFeatures_management(polygon, boundPoly)<br />
del polygon`

Usage instructions for noobs: Open everything you want to box up, click Geoprocessing > Python Console, paste in the above, hit enter, profit.

After I was done I buffered the bounding polygon by 250m or so because I&#8217;ll be running some raster analyses that have wacky edge effects. You should probably do something similar.