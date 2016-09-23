---
title: Fun with GDAL, Part 1 of x
date: 2016-07-12T22:49:16+00:00
author: obrl_soil
layout: post
permalink: /fun-with-gdal-part-1-of-x/
dsq_thread_id:
  - 4979649715
categories:
  - Spatial
tags:
  - GDAL
  - open data
  - "we don't need no stinkin' GUI"
---
If you ever find yourself needing part of a large (>2GB) raster dataset from the [CSIRO Data Access Portal](https://data.csiro.au/dap/home?execution=e3s1), you may need to use their [WebDAV environment](https://confluence.csiro.au/public/daphelp/data-access-portal-users-guide/find-data/data-descriptions/access-or-download-data/large-collection-access-non-csiro-users) to access it. This generally means downloading the entire raster (many of which are >50GB) in order to clip out a small chunk of it. This is a bit crap if you're not on AARNet, tbh. 

Here's an alternative - you can clip out portions of the target raster without downloading the whole thing first, using [gdal_translate](http://www.gdal.org/gdal_translate.html).

The following assumes you are working in Windows and have GDAL 1.11 - get it from [GIS Internals](http://www.gisinternals.com/). See the end of the post for an explanation of why I'm not using the latest version.

I got a little help nailing this workflow down from the good folks at [gis.stackexchange.com](http://gis.stackexchange.com/questions/201493/gdal-translate-and-remote-file-location).

1. In the CSIRO DAP, pick the dataset you want and click on the 'Files' tab. Fill out the data request stuff on the bottom-right of he page (email address, then pick 'Download via WebDAV' and the nearest location to you. Since your options are Canberra or elbourne, you may as well flip a coin if you're more than 500km from either).

2. When you receive your confirmation email, click the link and log in with the credentials provided.

3. Navigate to the exact file you want to extract. Most of the big datasets in the DAP contain .tif mosaics, so let&#8217;s focus on hose. Find the .tif you want, right click on it, and copy the full path to an open text file.

4. Check the metadata, then pick a set of bounding box coordinates in the dataset's CRS (almost always WGS84, EPSG:4326). Write them ut in decimal degrees, ulx, uly, rlx, rly

5. in your text file, create a gdal_translate command like:  
        `gdal_translate --config CPL_TMPDIR C:\path\to\local\folder -projwin_srs EPSG:4283 -projwin ulx uly lrx lry /vsicurl/http://USER:PASSWORD@webdav-bm.data.csiro.au/path/to/tif/sweet_sweet_data.tif C:\path\to\local\folder\sweet_sweet_data_clip.tif`  

Where it says USER:PASSWORD above, substitute in the email address and password you received in your access email. Replace the @ in your email address with its UTF-8 code, %40.

Once you've worked out the whole command, open the OSGeo4W shell and paste it in, then hit enter. If all goes well, you should see a line reporting the number of rows and columns in the target dataset, and then a progress bar. Go make a coffee.

The output .tif will contain any cells whose lower-right corners fell within the bounding box. No alignment shifts or resampling will occur.

**Update 01-Aug-2016:** Turns out there's a bit of a bug in GDAL 2.1 - a change was made that meant the -projwin tag did some unexpected pixel reshaping and realignment. [The behaviour should be reverted in v.2.1.2](https://trac.osgeo.org/gdal/ticket/6610), so stick to using v1.x for this kind of operation until then. I've tested the method successfully with GDAL 1.9 and 1.11. Note also that the QGIS >= 2.14 Clipper tool relies on GDAL 2.1.0, so it may not perform as you might expect.

NB: If you're behind a firewall, you'l have to set up your proxy stuff first. In the OSGeo4w shell, the commands are:  
* `set GDAL_HTTP_PROXY=http://proxy_url:port`  
* `set GDAL_HTTP_PROXYUSRPASS=USER:PASSWORD` (in this case, sub in your proxy logon stuff, not the WebDAV credentials. For Win users, this is usually your network logon and password. Don't save those in any text files, of course)  

This has been working pretty well for me over the past few days, although transfer times got really slow once the work week started; about 2 hours for 140MB, ugh. Not sure whether to blame general usage levels or the PokemonGo launch...
