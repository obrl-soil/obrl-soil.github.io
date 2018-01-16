---
title: "dsmartr: My first R package, and some thoughts on learning to code"
author: "@obrl_soil"
date: "2018-01-16"
layout: post
permalink: /dsmartr-announcement/
categories: 
  - digital soils mapping
  - soil science
tags: 
  - R
  - dsmart
  - meta
  - ramblings
---




Welp, I've written my first R package so I guess I oughta blog about it. [`dsmartr`](https://github.com/obrl-soil/dsmartr) is a digital soils mapping package which implements the DSMART algorithm of [Odgers et al (2014)](http://dx.doi.org/10.1016/j.geoderma.2013.09.024). The idea is to take existing polygon-based soils mapping and correlate it with various dense gridded environmental datasets (covariates), allowing production of a more detailed soils map. 'Soil map disaggregation' is a common phrase used to describe the process. I'm going to demonstrate the package's use in a series of future blog posts, but today I want to stay meta and talk about how I got to this point in the first place.

In mid-2016 I was handed a work project that had two goals: 1) Use DSMART to disaggregate a whole lot of 1:100,000 scale soils mapping along the Queensland coast, and 2) use the disaggregated outputs as a basis for mapping soils attributes and constraints to agriculture at a fine scale (30m pixels). If the process worked well enough, it could put property-scale soils mapping in the hands of local landowners without the need for a lot of extra field and lab-work. If the process didn't work that well, it would be a clear demonstration that we can only push legacy soils data so far.

I came into this with fairly minimal Python and R skills - I'd done some 101-level courses and mucked around with basic scripting, and I had a strong background in point-and-click GIS software (oh, and soil science, of course!). That alone might have got me through to some acceptable products without too much fuss, but I'd been reading a lot about reproducible workflows and had a bee in my bonnet about making the project fully open-source and replicable. Point-and-click was out of the question, I had to up my scripting game.

My learning from this point on was frustration-driven and somewhat chaotic, but since it gave me a lot of small, clear goals to work on, I think it was actually a good way to learn deeply. A lot of my intial upskilling was centered around input data preparation, which gave me a solid grounding in the R packages responsible for spatial data and database interaction. Tasks included:

  * Connecting to our Oracle-based soils database to extract both spatial and attribute data (skills: prying non-standard connection details out of database admin staff ('...Why can't you just use [Discoverer](https://gohubble.com/hubbleology/213/support-for-oracle-discoverer-has-ended-)?' 'SO. MANY. REASONS.'); using `DBI`; writing SQL in R scripts)
  * Cleaning the input map data and arranging it in a format suitable for disaggregation (skills: cleaning spatial data; combining multiple adjacent soils map projects into one coherent dataset; rearranging and reformatting attributes)
  * Finding and downloading environmental covariates from internal and external repositories (skills: finding out about covariates from the literature, conversations with colleagues, strategic googling; using GDAL to extract portions of large (e.g. Australia-sized) rasters stored in online repositories; appeasing my &*%^ing workplace firewall)
  * Creating new covariates using some of the downloaded data and a wide range of existing algorithms (skills: Using open source GIS tools like GRASS and SAGA via R with `raster`, `rgrass7`, and good ol' `base::system2()`; picking up enough OS knowledge to get all those programs to talk to each other).
  
Other skills I picked up in this stage related to using RStudio to properly document my work. I started using R projects to manage each sub-area of my project, as the data had to be processed in geographically distinct sections. I moved from .R scripts to .Rmds, generating low-level reporting for each processing step I took from raw data to final product. I also started adding tables and graphs to my Rmd reports with packages like `DT`, `ggplot2`, and `mapview`. The process of learning to write these reports was incredibly valuable as it forced me to work out a solid structure for my data analysis process - one that worked both for me and for the less-technical people who were funding my project and supervising my work.

As I was developing these processes, I started to look more closely at the DSMART code I was using. The DSMART algorithm was originally implemented in Python, and later [ported to R](https://bitbucket.org/brendo1001/dsmart) (non-CRAN). I had started my project using the Python version because 'Everybody Says Python Is Better'. In this case... nope. I switched to R after realising the Py code couldn't handle the data volume I was throwing at it. Also, I'd spent a couple of weeks battling obscure dependencies and compatibility issues that had arisen as the package aged, and I was heartily sick of it. I could have persisted trying to learn Py for this work, but R was just far more accessible. Better docs, more tutorials, and easier setup. CRAN really is amazing.

Anyway, the existing R version was more stable, if slightly slower, but still didn't quite meet my needs...so I started tinkering, and things kind of snowballed from there. The R package I was using had some bugs and RAM-consumption issues that needed attention. I started small, fixing the bugs and tweaking some output files, and before long I had [alternate versions of the main package functions](https://github.com/obrl-soil/disaggregation) that could handle much bigger input datasets without falling over. This was a huge confidence booster, so when the sf package was released as the successor to sp, I felt capable of modifying the code to take advantage of the newer package. This led to substantial speed gains and more readable code, although unfortunately I couldn't drop sp completely as raster is not sf-compatible. At around this point (late 2016), I was achieving good enough outputs with my pilot study area to present my work at the joint [Aus/NZ soils conference](https://nzsss.science.org.nz/nzasss-conference/) in Christchurch.

After that I decided I should go ahead and package my functions rather than relying on standalone scripts. Colleagues in my office wanted to use my scripts, and getting them to run on multiple machines without my supervision was difficult. After having spent all that time learning to code and applying my fancy new skills, I was having to rush through disaggregating the rest of the soils maps on my list, and I still had to get a good process for attribute mapping off the ground. I didn't need distractions! I also wanted a packaged version of my code to make it easy to demonstrate exactly which code I'd used for my project, and so I could cite it clearly.

Starting the packaging process was very easy with the help of github and [R packages](http://r-pkgs.had.co.nz/). Quiet shout-out to the GitHub for Windows desktop app, incidentally. Getting version control set up was the easy part, though - going from scripts to packaged functions involved a lot more code modification than I thought. 

My functions used a lot of tidyverse-style piped code which doesn't really work inside packages without substantial modification, and I'd gotten a bit caught up using fancy new functions where base R was perfectly fine. I love tidyverse for data prep and display, but I don't fully grok programming tidyverse-style yet and I confess I'm not in a huge hurry to learn. 

I then learnt how to document my R code properly using `roxygen2` and actually spent a lot of time on that - I've often been frustrated by other packages' docs and couldn't bear to be a hypocrite on that front. I also added quite a few functions to handle data preparation and post-processing evaluation. [Draft version 0.0.0.9013](https://github.com/obrl-soil/dsmartr/releases/tag/v0.0.0.9013) wound up being the version I used to finalise my disaggregation project in early November 2017, as I was over deadline and also quite burnt out after 18 months of very steep learning curve.

That's probably the downside of learning the way I did - I got into this spiral of learning some new trick, immediately having to try it out, and then if it worked, having to re-run my code for half a dozen sub-project-areas so that all the outputs were consistent. This was time-consuming and stressful even though it did lead to stronger products, and it was difficult for me to draw a line and say 'enough'. Eventually I admitted how exhausted I was getting and forced myself to wrap things up. At that point my outputs weren't going to get any better, and I was basically running on Berocca and spite. D-minus work/life balance management, would not reccommend.

I've since recharged enough to progress the package to something I'm content to publicise - the last task was to unit test as much as possible. The main functions are now just wrappers that handle things like on-disk file creation and parallel processing, whereas before they contained a lot of sub-functions that did the actual work of disaggregation. The wrappers still make life a lot easier for the end user, but the important parts of the process like polygon sampling are now separated, documented and covered by unit tests so its clear to everyone that they do what they ought to. I'm not actually sure how best to extend unit test coverage beyond where it is now (ideas welcome!), but I'm confident that the package works as it should.

### Learning resources

Throughout this project I've been very reliant on free online learning resources, as not many people in my workplace do any kind of programming, and there was no funding for formal training. Resources I found invaluable include:

  * Twitter: Twitter is amazing for #rstats. Following the right people and hashtags kept me up to date with everything from Big Stuff like new R packages to useful little tips like '[use `file.path()` instead of `paste0()`](https://twitter.com/bhaskar_vk/status/852180763101581312)'. The only downside is managing the onslaught of new ideas; this is a very fast-moving space.
  * [rweekly.org](https://rweekly.org/) has become a fantastic digest of all things #rstats.
  * [GIS-SE](https://gis.stackexchange.com/) and [Stack Overflow](https://stackoverflow.com/) - I know SE sites have an intimidating reputation, but its still worth wading in, you just have to pay attention to the social norms there. They do exist for a reason, which others have mentioned but bears repeating: I solved at least half a dozen major problems I was having without even asking a question. The act of trying to formulate an acceptable question that wouldn't get locked led me straight to the solution. Better than a rubber duck. I'd like to think I've given back a bit, too.
  * Edzer Pebesma's vignettes for sf, and his blog posts on [r-spatial](http://r-spatial.org/) were invaluable for the move from sp to sf, and to gaining a deeper understanding of how spatial data stuff really works. This is something that a lot of point-and-click GIS users don't even realise they don't understand - I've had some uncomfortable [Dunning-Kruger](https://en.wikipedia.org/wiki/Dunning%E2%80%93Kruger_effect) moments this past year, but I'm better off for it.
  * GitHub issues pages - if your problem is not on SE, its probably here.
  * Hadley Wickham's online books '[R for Data Science](http://r4ds.had.co.nz/)' and '[R packages](http://r-pkgs.had.co.nz/)' (especially the latter). I know, me and everyone else, but they're popular for a reason.
  *  Yihui Xie's knitr documentation: https://yihui.name/knitr/options/#chunk_options may as well have been my homepage for while there.
  * Kieran Healy's '[Data Visualisation: A practical introduction](http://socviz.co/)'. This is just so well-written, it doesn't matter that its not aimed at my field. It works well as a general intro to R and to best-practice data visualisation, as well as providing specific coding instruction.
  
Honourable mention for the upcoming ['Geocomputation with R'](https://geocompr.robinlovelace.net/) ebook, which came along a little too late for me, but is worth a read for anyone new to this space. It'll save you a lot of time. The [RStudio community forums](https://community.rstudio.com/) also launched recently and they're pretty cool.

### Where to from here?

  * At some point I should consider a CRAN submission, but that can wait until the various versions of R-based DSMART code have been reconciled. I don't think anyone wants multiple versions of the same idea on CRAN, and I've had a few idle chats with Nathan Odgers and Brendan Malone about combining our efforts. Herding soil scientists is worse than herding cats though, so don't hold your breath :P
  * Oh gosh [`stars`](https://github.com/r-spatial/stars) is coming! I might be able to move away from sp/raster later this year, which will be nice as dsmartr's dependency list is quite long. Not going to bother until `stars` is on CRAN, though.
  * Train myself to type 'dsmartr' correctly the first time instead of having to constantly change it from dsamrtr *\*sigh\**
  
Ok, I've rambled enough. Next time, how to DSMART.

