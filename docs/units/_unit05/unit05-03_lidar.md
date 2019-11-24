---
title: "Deriving a CHM from LiDAR & more"
toc: true
toc_label: In this example
---

Light detection and ranging (LiDAR) observations are point clouds representing the returns of laser pulses reflected from objects, e.g. a tree canopy. Processing LiDAR (or optical point cloud) data generally  requires more computational resources than 2D optical observations. The spotlit provides both technical handling of LiDAR data as well as some helpful hints how to optimize the technical and conceptual workflow of prgramming, data handling and information gathering.

<!--more-->

## Introduction


For this example, the lidR package will be used. Extensive documentation and workflow examples can be found in the Wiki of the respective [GitHub repository](https://github.com/Jean-Romain/lidR). Other software options include e.g. [GRASS GIS](http://grasswiki.osgeo.org/wiki/LIDAR) or [ArcGIS](https://desktop.arcgis.com/en/arcmap/10.3/manage-data/las-dataset/a-quick-tour-of-lidar-in-arcgis.htm).

For the following, make sure these libraries are part of your setup (should be the case if you follow [extended setup spotlight]({{ site.baseurl }}{% link _unit05/unit05-02_extended_setup.md %}){:target="_blank"}.

```r
libs = c("lidR", "link2GI", "mapview", "raster", "rgdal", "rlas", "sp", "sf")
```

## Checking environment and capabilities to deal with  LAS data sets for further processing
Technically spoken the LiDAR data comes in the LAS file format (for a format definition i.e. have a look at the [American Society for Photogrammetry & Remote Sensing LAS documentation file](https://www.asprs.org/a/society/committees/standards/LAS_1_4_r13.pdf)). One LAS data set typically but not necessarily covers an area of 1 km by 1 km. Since the point clouds in the LAS data sets are large, a spatial index file (LAX) considerably reduces search and select operations in the data.

Now we make a short general check that we can start over i.e. if everything is ready to use. First load the tutorial data to the [temp]({{ site.baseurl }}{% link _unit05/unit05-02_extended_setup.md %}){:target="_blank"} data folder of your project.

```r
# NOTE file size is about 12MB
utils::download.file(url="https://github.com/gisma/gismaData/raw/master/uavRst/data/lidR_data.zip",
                     destfile=paste0(envrmt$path_tmp,"chm.zip"))
unzip(paste0(envrmt$path_tmp,"chm.zip"),
      exdir = envrmt$path_tmp,  
      overwrite = TRUE)

las_files = list.files(envrmt$path_tmp,
                       pattern = glob2rx("*.las"),
                       full.names = TRUE)

```

If you want to read a single LAS file, you can use the readLAS function. Plotting the data set results in a 3D interactive screen which opens from within R.

```r
lidar_file = readLAS(las_files[1])
plot(lidar_file, bg = "green", color = "Z",colorPalette = mvTop(256),backend="pcv")
```

{% include figure image_path="/assets/videos/las_mof_plot.gif" alt="LiDAR 3D animation." %}



If this works, you're good to go.

## Creating a Canopy Height Model (CHM) Part 1

For training purposes (i.e. installing of missing packages, adapting of external scripts to our needs etc.) please switch over to the online tutorial [Building a CHM from LiDAR Data](https://github.com/gisma/uavRst/wiki/Building-a-Canopy-Height-Model-(CHM)-using-lidR){:target="_blank"}.

Please follow the tutorial risk a click on the linked content and consider the following points:

- just try to run the tutorial as it is. Do not integrate it in your project structure (best for training would be to set up a new structure according to your needs)
- if you are not able to install necessary packages consider if the package is mandatory for success (a viewer may not be necessary etc.).
- Try to understand what is happening. Do the tutorial **step by step!**


## Creating a Canopy Height Model (CHM) Part 2

After successful application of the tutorial we will transfer it into a suitable script for our workflow. Please check the comments for better understanding and do it **step by step!** again. Please note that the following script is meant to be an basic example how: 
- to organize scripts in common 
- with a hands on example for creating a canopy height model.

For a deeper understanding please look at the [Spotlight Best practices in scripting with R]({{ site.baseurl }}{% link _unit05/unit05-05_best_scripting.md %}){:target="_blank"}.  



After revisiting the tutorial is seems to be a good choice to follow the tutorial of the `lidR` developer that is [linked](https://github.com/Jean-Romain/lidR/wiki/Rasterizing-perfect-canopy-height-models){:target="_blank"} in the above tutorial. Why? because for the first Jean-Romain explains 6 ways how to create in a very simple approach a CHM, second to show up that it makes sense to read and third to loop back because it does not work with bigger files. Let us start with the new script structure.


```r
#------------------------------------------------------------------------------
# Name: make_CHM_Tiles.R
# Type: control script 
# Author: Chris Reudenbach, creuden@gmail.com
# Description:  script creates a canopy height model from generic Lidar 
#              las data using the lidR package
# Data: regular las LiDAR data sets 
# Copyright: Chris Reudenbach, Thomas Nauss 2017,2019, GPL (>= 3)
#------------------------------------------------------------------------------


# 0 - load packages
#-----------------------------
library("lidR")
library("future")


# 1 - source files
#-----------------

#---- source setup file
source(file.path(envimaR::alternativeEnvi(root_folder = "~/edu/mpg-envinsys-plygrnd",
                                          alt_env_id = "COMPUTERNAME",
                                          alt_env_value = "PCRZP",
                                          alt_env_root_folder = "F:/BEN/edu"),
                 "src/mpg_course_basic_setup.R"))


# 2 - define variables
#---------------------

## define current projection (It is not magic you need to check the meta data or ask your instructor) 
## ETRS89 / UTM zone 32N
proj4 = "+proj=utm +zone=32 +datum=WGS84 +units=m +no_defs +ellps=WGS84 +towgs84=0,0,0"


## get viridris color palette
pal<-mapview::mapviewPalette("mapviewTopoColors")

# 3 - start code 
#-----------------

  
 #---- NOTE file size is about 12MB
 utils::download.file(url="https://github.com/gisma/gismaData/raw/master/uavRst/data/lidR_data.zip",
                     destfile=paste0(envrmt$path_tmp,"chm.zip"))
 unzip(paste0(envrmt$path_tmp,"chm.zip"),
      exdir = envrmt$path_tmp,  
      overwrite = TRUE)
 
 #---- Get all *.las files of a folder into a list
 las_files = list.files(envrmt$path_tmp,
                       pattern = glob2rx("*.las"),
                       full.names = TRUE)
 
 #---- create CHM as provided by
 # https://github.com/Jean-Romain/lidR/wiki/Rasterizing-perfect-canopy-height-models
 las = readLAS(las_files[1])
 las = lidR::lasnormalize(las, knnidw())
 
 # reassign the projection
 sp::proj4string(las) <- sp::CRS(proj4)
 
 # calculate the chm with the pitfree algorithm
 chm = lidR::grid_canopy(las, 0.25, pitfree(c(0,2,5,10,15), c(0,1), subcircle = 0.2))
 
 # write it to tif
 raster::writeRaster(chm,file.path(envrmt$path_data_mof,"mof_chm_one_tile.tif"),overwrite=TRUE) 
 

# 4 - visualize 
-------------------

# call mapview with some additional arguments
mapview(raster::raster(file.path(envrmt$path_data_mof,"mof_chm_one_tile.tif")),
         legend=TRUE, 
         layer.name = "canopy height model",
         col = pal(256),
         alpha.regions = 0.7)

```

{% include  media url="/assets/misc/chm_one_tile.html" alt="Canopy Height Model map." %}
[Full screen version of the map]({{ site.baseurl }}/assets/misc/chm_one_tile.html){:target="_blank"}
### Coping with computer memory resources

The above example is based on a las-tile of 250 by 250 meter. That means a fairly small tile size. If you did not run in memory or CPU problems you can deal with these tiles by simple looping.  

**But** you have to keep in mind that even if a tile based processing can easily be handled with loops but has some pitfalls. E.g. if you compute some focal operations, you have to make sure that if the filter is close to the boundary of the tile that data from the neighboring tile is loaded in addition. Also the tile size may be a problem for your memory availability. 

The lidR package comes with a feature called catalog and a set of catalog functions which make tile-based life easier. A catalog meta object also holds information on e.g. the cartographic reference system of the data or the cores used for parallel processing. So we start over again - this time with a **real** data set.

If not already done [download](http://gofile.me/3Z8AJ/c6m5CfvWZ){:target="_blank"} the data for the target area to the `envrmt$path_lidar_org` folder. 

Please note that dealing with `lidR` catalogs is pretty stressful for the memory adminstration of your rsession. So best practicese is to:
1 clean the environment 
1 restart your rsession

This will help you to avoid frustrating situation like restarting your PC after hours of waiting...

```r
#------------------------------------------------------------------------------
# Type: control script 
# Name: make_CHM_Catalog.R
# Author: Chris Reudenbach, creuden@gmail.com
# Description:  script creates a canopy height model from generic Lidar 
#              las data using the lidR package. In addition the script cuts the area to 
#              a defined extent using the lidR catalog concept.
#              Furthermore the data is tiled into a handy format even for poor memory 
#              and a canopy height model is calculated
# Data: regular las LiDAR data sets 
# Output: Canopy heightmodel RDS file of the resulting catalog
# Copyright: Chris Reudenbach, Thomas Nauss 2017,2019, GPL (>= 3)
#------------------------------------------------------------------------------


# 0 - load packages
#-----------------------------
library("future")

# 1 - source files
#-----------------
source(file.path(envimaR::alternativeEnvi(root_folder = "~/edu/mpg-envinsys-plygrnd",
                                          alt_env_id = "COMPUTERNAME",
                                          alt_env_value = "PCRZP",
                                          alt_env_root_folder = "F:/BEN/edu"),
                 "src/mpg_course_basic_setup.R"))


# 2 - define variables
#---------------------


##  area of interest (central MOF)
cxl<-476174.
cyb<-5631386.
cxr<-478217. 
cyt<-5632894.

## define variables for the lidR catalog
chunksize = 100
overlap= 10

## define current projection (It is not magic you need to check the meta data or ask your instructor) 
## ETRS89 / UTM zone 32N
proj4 = "+proj=utm +zone=32 +datum=WGS84 +units=m +no_defs +ellps=WGS84 +towgs84=0,0,0"


# get viridris color palette
pal<-mapview::mapviewPalette("mapviewTopoColors")

# 3 - start code 
#-----------------

# Get all *.las files of a folder into a list
las_files = list.files(envrmt$path_lidar_org, pattern = glob2rx("*.las"), full.names = TRUE)

#------- set up the a FIRST lidR catalog 
# we need it for better handling poor available memory 
# check on https://rdrr.io/cran/lidR/man/catalog.html
mof_ctg = lidR::readLAScatalog(envrmt$path_lidar_org)
sp::proj4string(mof_ctg) <- sp::CRS(proj4) 
future::plan(multisession, workers = 2L)
lidR::set_lidr_threads(4)
lidR::opt_chunk_size(mof_ctg) = chunksize
lidR::opt_chunk_buffer(mof_ctg) <- overlap


#------- cut the original data to new extent 
# check on https://rdrr.io/cran/lidR/man/lasclip.html
las_core_aoimof<- lidR::lasclipRectangle(mof_ctg, xleft = cxl, ybottom = cyb, xright = cxr, ytop = cyt)

#---- write the new dataset to the level0 folder and create a corresponding index file (lax)
lidR::writeLAS(las_core_aoimof,file.path(envrmt$path_level0,"las_mof.las"))
rlas::writelax(file.path(envrmt$path_level0, "las_mof.las"))

#---- setting up a *NEW* lidR catalog for further processing 
mof100_ctg <- lidR::readLAScatalog(envrmt$path_level0)
sp::proj4string(mof100_ctg) <- sp::CRS(proj4)
future::plan(multisession, workers = 2L)
lidR::set_lidr_threads(4)
lidR::opt_chunk_size(mof100_ctg) = chunksize
lidR::opt_chunk_buffer(mof100_ctg) <- overlap
lidR::opt_output_files(mof100_ctg) <- paste0(envrmt$path_ID,"/{ID}")

#---- retile it
# check on https://rdrr.io/cran/lidR/man/catalog_retile.html
mof100_ctg = lidR::catalog_retile(mof100_ctg)

# add correct projection (it is sometimes lost)
sp::proj4string(mof100_ctg) <- sp::CRS(proj4)

# save both catalogs to RDS FILES
saveRDS(mof_ctg,file= file.path(envrmt$path_level1,"mof_ctg.rds"))
saveRDS(mof100_ctg,file= file.path(envrmt$path_level1,"mof100_ctg.rds"))

# re-read the RDS file
mof<-readRDS(file.path(envrmt$path_level1,"mof100_ctg.rds"))

#---- calculating chm
# add output filname template
lidR::opt_output_files(mof) <- paste0(envrmt$path_normalized,"/{ORIGINALFILENAME}_tinnormalized")

# Remove the topography from a point cloud using knnidw()
mof_norm_tin = lidR::lasnormalize(las = mof, algorithm = tin())

saveRDS(mof_norm_tin,file= file.path(envrmt$path_level1,"mof_norm_tin.rds"))
mof_norm<-readRDS(file.path(envrmt$path_level1,"mof_norm_tin.rds"))


#---- Creates a digital surface model (DSM) using pitfree()
# if the error  "filename exists; use overwrite=TRUE" occures navigate to the 
# paste0(envrmt$path_normalized,"/{ORIGINALFILENAME}_normalized") fo
chm_all = grid_canopy(mof_norm, 1.0, pitfree(c(0,2,5,10,15), c(0,1), subcircle = 0.2))

# write it to tiff
raster::writeRaster(chm_all,file.path(envrmt$path_data_mof,"mof_chm_all.tif"),overwrite=TRUE) 


# 4 - visualize 
-------------------

## set mapview raster options to full resolution
## check on https://r-spatial.github.io/mapview/
mapview::mapviewOptions(mapview.maxpixels = 1000*1000)

## visualize the catalogs
mapview(mof_ctg ) + mapview(mof100_ctg, zcol= "Max.Z" )

## call mapview with some additional arguments
mapshot(mapview(raster::raster(file.path(envrmt$path_data_mof,"mof_chm_all.tif")),
         legend=TRUE, 
         layer.name = "canopy height model",
         col = pal(256),
         alpha.regions = 0.65),url="chm_all_map.html")



```
### The visualisation of an operating lidR catalog action.

The `mof_ctg` catalog is shows the extent of the original las file as provided by the the data server. The vecor that is visualized is the resulting `lidR` catalog containing all extracted parameters. Same with the `mof100_ctg` herre you see the extracted maximum Altitude of each tile used for visualisation. Feel free to investigate the catalog parameters by clicking on the tiles.  

{% include media url="/assets/misc/mof_lidr_catalog.html" alt="lidR catalog map." %}

### The whole clipped MOF area rendered to a canopy height model. 

{% include  media url="/assets/misc/chm_all.html" alt="Canopy Height Model map." %}
[Full screen version of the map]({{ site.baseurl }}/assets/misc/chm_all.html){:target="_blank"}

Assuming this script (or the script you have adapted) runs without any technical errors, the exciting part of the data preprocessing comes now:
* which algorithms did I use and why? 
* do the values make sense? 
* which variables and whats the range of Values? 
* why not? What correction options do I have? 
* is the error due to the data or the algorithm or the script?

## Where to go?

To answer the above questions we need a two folded approach. First we need to strip down the script to a *real* control script all functionality that is used more than once will be moved into functions. Second we need to find out what algorithm is most suitable and reliable to answer our questions. 

The spotlight [simple functions]({{ site.baseurl }}{% link _unit05/unit05-05_best_scripting.md %}){:target="_blank"} deals with the first task. It shows to strip this control file into useful functions. Remember all functions are stored in the folder `envrmt$path_src`. 

The spotlight [validation strategies]({{ site.baseurl }}{% link _unit05/unit05-09_validation_strategies.md %}){:target="_blank"} deals with the second task.

Now everything is prepared to dive into  [tree segmentation ]({{ site.baseurl }}{% link _unit05/unit05-07_segmentation_strategies.md %}){:target="_blank"}strategies.

