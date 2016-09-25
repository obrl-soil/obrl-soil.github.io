---
title: "Mapping soil texture class from particle size rasters"
author: "obrl_soil"
date: "2016-09-25"
layout: post
comments: true
permalink: /soil-texture-rasters/
categories: 
  - spatial
tags: 
  - R
  - raster
  - soiltexture
---




A lot of new soil mapping projects are being delivered as a stack of attribute surfaces rather than an intepreted chloropleth map. This gives the end user freedom to mix and match components, generating custom products that suit their needs. That's great if you know how, but I haven't seen all that much documentation around, so. Here's a quick workflow for one of the simplest possible products, a soil texture class map from particle size distribution rasters.

I'm demo-ing this using some modelled data I pinched from the [Soil and Landscape Grid of Australia](http://www.clw.csiro.au/aclep/soilandlandscapegrid), which is a pretty boss product. To be specific, I used the gdal_translate method [I blogged about here](https://obrl-soil.github.io/fun-with-gdal-part-1-of-x/) to extract a ~1 x 1 degree area from each of the 0-5cm clay, silt and sand datasets (ulx 148, uly -22).

<img src="{{ site.url }}/images/soil-texture-rastersinputdata-1.png" title="_but what does it mean?!_" alt="_but what does it mean?!_" width="80%" style="display: block; margin: auto;" />

I am also using the `soiltexture` package to do most of the heavy lifting, and boy am I ever glad I didn't have to code all that myself.

Required packages and data sources:


```r
library(sp)
library(rgdal)
library(raster)
library(soiltexture)
library(rasterVis)
library(RColorBrewer)
library(dplyr)

working_dir <- 'C:/Users/obrl_soil/Downloads'
clay_src <- paste0(working_dir, '/Clay_05_1dd.tif')
silt_src <- paste0(working_dir, '/Silt_05_1dd.tif')
sand_src <- paste0(working_dir, '/Sand_05_1dd.tif')
inputs <- c(clay_src, silt_src, sand_src)
```

Note the order of entry for `inputs`. One must be consistent from this point on. Next thing to do is read the three rasters into a `RasterStack` object, and promote that to `SpatialPixelsDataFrame`.


```r
SSC <- stack()

for (i in 1:length(inputs)) {
  rn  <- inputs[i]  
  r   <- raster(rn)
  SSC <- addLayer(SSC, r)
}

SSCP <- as(SSC, 'SpatialPixelsDataFrame')
```

Yeah, I used a loop in R code. _Fight me_.

I chose SpatialPixelsDataFrame (SPDF) over SpatialGridDataFrame (SGDF) for a couple of reasons. Firstly, the workflow fails for SGDF if there are any NA values in the input data, so coastal datasets would be a problem. I'm too much of a n00b to pin down why SPDF can handle it and SGDF can't beyond 'SPDF has @grid.index' but there it is. The other issue is that SGDF is intended for strictly regular grids, and anything projected in spherical coordinates fails that test. All this data is in WGS84 (and no, it doesn't matter if the extent is small!).

Now, the soiltexture package often looks for default column headings so some renaming at this point can make life easier. Also, I'm going to round off the PSD data, since it wanders out to 6+ decimal places. Laboratories don't measure PSD that precisely - anything past &plusmn;0.01 is a fantasy, and the rounded data is just easier to read.


```r
names(SSCP@data) <- c('CLAY', 'SILT', 'SAND')
SSCP@data <- round(SSCP@data, 2)
SSCP@data$raw_totals <- rowSums(SSCP@data[, 1:3])
```

Particle size data should add to about 100%. There's some legitimate variation in lab results, partly due to the presence of organic matter, dissolvable materials etc, and partly due to inherent method imprecision, but a good result range is 100% &plusmn;5%. Maybe &plusmn;10% in some surface soils, thanks to elevated OM (OM% by volume). Probably not in central Queensland, though.

Anyway modelled data can fail this acceptable-range test, because each fraction is predicted independent of the others. This sample isn't too bad - the mean of total PSD is 97.11 and the standard deviation is 3.54 but the range is still 77.6 to 118.04 and the histogram looks like

<img src="{{ site.url }}/images/soil-texture-rastershisto-1.png" title="plot of chunk histo" alt="plot of chunk histo" width="80%" style="display: block; margin: auto;" />

Orange = too low, blue = too high. There's also a spatial pattern to the out of range values which bothers me a bit:

<img src="{{ site.url }}/images/soil-texture-rastersfailmap-1.png" title="plot of chunk failmap" alt="plot of chunk failmap" width="80%" style="display: block; margin: auto;" />

Glancing at the histogram and the satellite imagery, there's particular underprediction on alluvial flats, idk why - those are usually the more heavily sampled areas, for accessibility reasons if nothing else. Whatever, all up, about a third of the pixels are out of range. So a) be cautious about modelled PSD data applications and interpretation and b) proportional normalisation of your PSD data should be a routine step unless you have a good reason not to do so. The following is the simplest kind of normalisation: `fraction/total of fractions * 100`.


```r
SSCP_norm <- TT.normalise.sum(tri.data = SSCP@data, residuals = T)
colnames(SSCP_norm)[1:3] <- paste0(colnames(SSCP_norm)[1:3], "_n")
SSCP@data <- cbind(SSCP@data, round(SSCP_norm, 2)) 
rm(SSCP_norm)

str(SSCP@data)
```

```
## 'data.frame':	1440000 obs. of  9 variables:
##  $ CLAY      : num  10.5 12.3 15.1 12.9 10.6 ...
##  $ SILT      : num  7.84 8.85 9.36 8.63 9.29 8.99 8.73 9.36 9.56 8.64 ...
##  $ SAND      : num  86.3 80.8 75.2 79.5 83 ...
##  $ raw_totals: num  104.7 102 99.6 101 102.9 ...
##  $ QA        : Factor w/ 3 levels "Too low","Just right!",..: 2 2 2 2 2 2 2 2 2 2 ...
##  $ CLAY_n    : num  10 12.1 15.1 12.8 10.3 ...
##  $ SILT_n    : num  7.49 8.68 9.39 8.55 9.03 9.03 8.81 9.63 9.31 8.9 ...
##  $ SAND_n    : num  82.5 79.2 75.5 78.7 80.7 ...
##  $ residuals : num  4.69 2.01 -0.35 0.97 2.89 -0.46 -0.89 -2.76 2.74 -2.96 ...
```

Now the normalised data can be classified. 'soiltexture' comes with a bunch of predefined texture triangles. I'm going to use the Australian one to match the source data. Check the documentation for `TT.points.in.classes` for details but the upshot is that I'm pulling out a text vector of classes with length = _n_ pixels and adding it as a new @data$column:


```r
SSCP@data <- cbind(SSCP@data, 
                  "TEXCLASS" = TT.points.in.classes(tri.data  = SSCP@data[, c('CLAY_n', 'SILT_n',  'SAND_n')],
                                                    css.names = c('CLAY_n', 'SILT_n', 'SAND_n'),
                                                    class.sys = "AU2.TT", 
                                                    PiC.type  = "t",
                                                    collapse  = ', ')
                  )
```

For the speed-freaks, everything from data import to this point (excluding plots) took just over a minute on my little ol' Surface Pro (Intel Core i5 1.7GHz processor, 4GB RAM, Win 10).

At this point some stuff has to be dealt with: 

 - Some PSDs will fall on texture class boundaries and will be named e.g. "class1, class2". I just want a quick map so I'm happy to accept class1 in all cases. Note: rounding the input data actually made this issue slightly worse, but I'm still not willing to leave what amounts to imaginary variation in place.
 - A small number of cells have failed to classify for unknown reasons (probably the fault of the AU2.TT triangle rules, there's a comment to that effect in the [vignette](https://cran.r-project.org/web/packages/soiltexture/vignettes/soiltexture_vignette.pdf)). They all seem to be ~80% sand and ~9% each of silt and clay.
 - An unintended consequence of this is that one of the texture classes is `""` - and since its character data in a data frame, its been coerced to factor. R hates zero-length factors, and its not very informative anyway.


```r
levels(SSCP@data$TEXCLASS)
```

 [1] ""        "C"       "C, CL"   "CL"      "L"       "L, CL"   "LS"     
 [8] "SCL"     "SL"      "SL, L"   "SL, SCL"

```r
SSCP@data$TEXCLASS_SIMPLE <- as.character(SSCP@data$TEXCLASS)
SSCP@data$TEXCLASS_SIMPLE[SSCP@data$TEXCLASS_SIMPLE == ""] <- "Not classified"
SSCP@data$TEXCLASS_SIMPLE <- as.factor(SSCP@data$TEXCLASS_SIMPLE)
SSCP@data$TEXCLASS_SIMPLE <- recode(SSCP@data$TEXCLASS_SIMPLE,
                                   "C, CL" = "C", "L, CL" = "L", "SL, SCL" = "SL", "SL, L" = "SL")
levels(SSCP@data$TEXCLASS_SIMPLE)
```

[1] "C"              "CL"             "L"              "LS"            
[5] "Not classified" "SCL"            "SL"            

Lastly, I want my final factor list in a sensible order so my map looks pretty. 


```r
SSCP@data$TEXCLASS_SIMPLE <- factor(SSCP@data$TEXCLASS_SIMPLE, 
                                   levels = c('C', 'CL', 'SCL', 'L', 'SL', 'LS', 'Not classified'))
```

Lets take a look:

<img src="{{ site.url }}/images/soil-texture-rastersmappymap-1.png" title="plot of chunk mappymap" alt="plot of chunk mappymap" width="80%" style="display: block; margin: auto;" />

_Nooooooiice_ \*pats self on back\*

To export this as GeoTiff you still have to make it numeric, but its easy enough to write a lookup table to accompany the image


```r
SSCP@data$TEXCLASS_SN <- as.numeric(SSCP@data$TEXCLASS_SIMPLE)
writeGDAL(SSCP['TEXCLASS_SN'], 
          paste0(working_dir, '/tcmap_148-22.tif'), 
          drivername = 'GTiff')

LUT <- data.frame(unique(SSCP@data[, c('TEXCLASS_SN', 'TEXCLASS_SIMPLE')]))        
write.table(LUT, 
            file = paste0(working_dir, '/tcmap_148-22_LUT.txt'), 
            sep = ", ",
            quote = FALSE,
            row.names = FALSE,
            col.names = TRUE)  
```

And that's that. Next time, if I feel so inclined: what to do after you've classified all your depth slices.
