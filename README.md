# measuretreeheight
Trial on how to measure tree height from LiDAR data using R

## Background

One of the most common remote sensing application in agriculture and forestry field is tree height measurement. Several methods and tools can be tried out for tree height measurement. In this case, we will try out the Individual Tree Detection method from LiDAR data. 

## Objective

Conduct a tree height measurement on the LiDAR dataset using R (lidR packages).


## Dataset

NEON (National Ecological Observatory Network). Discrete return LiDAR point cloud (DP1.30003.001), RELEASE-2024. https://doi.org/10.48443/hj77-kf64. Dataset accessed from https://data.neonscience.org/data-products/DP1.30003.001/RELEASE-2024 on June 17, 2024.


## Method

The LiDAR point cloud first need to be classified as ground and non-ground point. Then the data will be normalized, this process is the same as the canopy height model (raster) for those more familiar with the term.
After that, we can apply the local maximum filter (LMF) algorithm to the dataset. The algorithm works by looking on the highest (maxima) on neighboring points in a circle of a certain radius (window size). 

The method I will be using is shown in this picture,

![diagram case 1](https://github.com/zachafif/measuretreeheight/assets/103014717/1f1cf7a2-ac83-4d20-986f-679f94f06c50)



## Result:

First we will import the library needed,

```
library(lidR)
library(terra)
library(sf)
library(raster)

```
Next we will load the LiDAR point cloud,

```
#Load Data

input<-"H:/AfifBelajar/R/NEON_lidar-point-cloud-line/NEON_lidar-point-cloud-line/NEON.D16.ABBY.DP1.30003.001.2017-06.basic.20240612T130227Z.RELEASE-2024/NEON_D16_ABBY_DP1_553000_5062000_classified_point_cloud_colorized.laz"

las<-readALSLAS(input)

```

Next, we do initial checking,

```
> print(las)
class        : LAS (v1.3 format 3)
memory       : 848.3 Mb 
extent       : 553000, 554000, 5062000, 5063000 (xmin, xmax, ymin, ymax)
coord. ref.  : WGS 84 / UTM zone 10N 
area         : 1 km²
points       : 9.67 million points
density      : 9.67 points/m²
density      : 5.05 pulses/m

```
Here we got that the CRS are in WGS 84 / UTM zone 10N  with a point density of 9.67 points/m² which is quite good. Additionaly we have a 4 return in this data.

Then we do more advanced checking to see if the data is valid,

```
> las_check(las)

 Checking the data
  - Checking coordinates... ✓
  - Checking coordinates type... ✓
  - Checking coordinates range... ✓
  - Checking coordinates quantization... ✓
  - Checking attributes type... ✓
  - Checking ReturnNumber validity... ✓
  - Checking NumberOfReturns validity... ✓
  - Checking ReturnNumber vs. NumberOfReturns... ✓
  - Checking RGB validity... ✓
  - Checking absence of NAs... ✓
  - Checking duplicated points...
    ⚠ 525 points are duplicated and share XYZ coordinates with other points
  - Checking degenerated ground points...
    ⚠ There were 66 degenerated ground points. Some X Y Z coordinates were repeated
    ⚠ There were 990 degenerated ground points. Some X Y coordinates were repeated but with different Z coordinates
  - Checking attribute population... ✓
  - Checking gpstime incoherances ✓
  - Checking flag attributes... ✓
  - Checking user data attribute...
    🛈 8653550 points have a non 0 UserData attribute. This probably has a meaning
 Checking the header
  - Checking header completeness... ✓
  - Checking scale factor validity... ✓
  - Checking point data format ID validity... ✓
  - Checking extra bytes attributes validity... ✓
  - Checking the bounding box validity... ✓
  - Checking coordinate reference system... ✓
 Checking header vs data adequacy
  - Checking attributes vs. point format... ✓
  - Checking header bbox vs. actual content... ✓
  - Checking header number of points vs. actual content... ✓
  - Checking header return number vs. actual content... ✓
 Checking coordinate reference system...
  - Checking if the CRS was understood by R... ✓
 Checking preprocessing already done 
  - Checking ground classification... yes
  - Checking normalization... no
  - Checking negative outliers...
    ⚠ 5 points below 0
  - Checking flightline classification... yes
 Checking compression
  - Checking attribute compression...
   -  Synthetic_flag is compressed
   -  Keypoint_flag is compressed
   -  Withheld_flag is compressed

```

We can see that the data is already classified as ground and non-ground (done by the data provider) which will skip one step. There are some data duplicates but we will keep it for now.

Next we will do data normalization,

```
#normalized Data

nlas <- normalize_height(class_las, knnidw())
nlas<- nlas[nlas$Z<40]

```

I add the last line because the result of normalization has an outlier (which is because we do not do any noise filter), therefore we try to filter the data with a limit no more than 40 meter above ground.

Next we do the Individual tree detection both using point cloud based processed and raster based processed. For point cloud based processed it is as simple as the following code,

```

#Individual Tree Detection (point based)

ttops <- locate_trees(nlas, lmf(ws = 5))

```

As for raster based processed, we need to generate canopy height model first which come from differencing a digital surface model and digital terrain model,

```

#Individual Tree Detection (raster based)

dtm<-rasterize_terrain(class_las,algorithm = knnidw(k = 6L, p = 2))

dsm <- rasterize_canopy(class_las)

chm<-dsm-dtm

writeRaster(chm,"H:/AfifBelajar/R/NEON_lidar-point-cloud-line/NEON_lidar-point-cloud-line/chm.Tif",overwrite=TRUE) #export the CHM

ttops <- locate_trees(chm, lmf(ws = 5))

```

After the process is done, we can plot the tree points results using this following code,

```
#Plot the results

x <- plot(nlas, color="RGB",bg = "white", size = 1)

add_treetops3d(x, ttops)

```
Here is the result on overview,
![NEON_D16_ABBY_DP1_553000_5062000_classified_point_cloud_colorized](https://github.com/zachafif/measuretreeheight/assets/103014717/aa9ed6ae-3bc2-463b-b118-824ef32b2c2d)


We can export the tree points into a shapefile,

```
#Export to SHP

ttops_sf<-st_sf(ttops)

ttops_sf<-st_zm(ttops_sf)

st_write(ttops_sf, "H:/AfifBelajar/R/NEON_lidar-point-cloud-line/NEON_lidar-point-cloud-line/ttops.shp")

```

Here is the tree point result loaded in QGIS overlayed on top of the CHM,
![image](https://github.com/zachafif/measuretreeheight/assets/103014717/7a6a8742-b340-442b-a5ae-b998fc950a59)

Also We can export the tree points into a CSV file with coordinates,

```
#Export to CSV

ttops_df<-as.data.frame(ttops_sf)

coordinates<-as.data.frame(st_coordinates(ttops$geometry))

ttops_df$X<-coordinates$X

ttops_df$Y<-coordinates$Y

ttops_df<-ttops_df[,c(1,4,5,2)]

write.csv(ttops_df,"H:/AfifBelajar/R/NEON_lidar-point-cloud-line/NEON_lidar-point-cloud-line/ttops.csv")

```

The result will be as follows,

![image](https://github.com/zachafif/measuretreeheight/assets/103014717/024efc71-c673-454a-93f5-2d2df249161f)
