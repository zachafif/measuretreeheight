# Tree Height Measurement using LiDAR
Trial on how to measure tree height from LiDAR data

![gifmaker_me](https://github.com/zachafif/measuretreeheight/assets/103014717/d5cd5595-6d83-4dfe-8b51-0e19126ac44a)


## Background

One of the most common remote sensing applications in agriculture and forestry field is tree height measurement. Several methods and tools can be tried out for tree height measurement. In this case, the Individual Tree Detection method from LiDAR data will be carried out. 

## Objective

Conduct a tree height measurement on the LiDAR dataset using Individual Tree Method (lidR packages).


## Dataset

NEON (National Ecological Observatory Network). Discrete return LiDAR point cloud (DP1.30003.001), RELEASE-2024. https://doi.org/10.48443/hj77-kf64. Dataset accessed from https://data.neonscience.org/data-products/DP1.30003.001/RELEASE-2024 on June 17, 2024.


## Method

The LiDAR point cloud first need to be classified as ground and non-ground point. Then the data will be normalized, this process is the same as the canopy height model (raster) for those who are more familiar with the term.
After that, the local maximum filter (LMF) algorithm applied to the dataset. The algorithm works by looking on the highest (maxima) on neighboring points in a circle of a certain radius (window size). 

The workflow the will be carried out shown in this picture,

![diagram case 1](https://github.com/zachafif/measuretreeheight/assets/103014717/1f1cf7a2-ac83-4d20-986f-679f94f06c50)



## Result:

First, import the library needed.

```
library(lidR)
library(terra)
library(sf)
library(raster)

```
Next, load the LiDAR point cloud.

```
#Load Data

input<-"inputdirectory/NEON_D16_ABBY_DP1_553000_5062000_classified_point_cloud_colorized.laz"

output<-"outputdirectory"

las<-readALSLAS(input)

```

Next, Do an initial checking,

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
Here it is stated that the CRS is in WGS 84 / UTM zone 10N  with a point density of 9.67 points/m² which is quite good.

Then more advanced checking were done to see if the data is valid,

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

From above console message, The data is already classified as ground and non-ground (done by the data provider) which will skip one step. There are some data duplicates but for now it will be kept.

Next is data normalization. The process that being done to normalize LiDAR elevation data relative to ground points.

```
#normalized Data

nlas <- normalize_height(class_las, knnidw())
nlas<- nlas[nlas$Z<40]

```

The last line was added because the result of normalization has an outlier (which is because no noise filter were done), therefore the data will be filtered with a limit of no more than 40 meters above the ground.

Next the Individual tree detection can be done both using point cloud based processes and raster based processes. For point cloud-based process, it is as simple as the following code,

```

#Individual Tree Detection (point based)

tree_pts <- locate_trees(nlas, lmf(ws = 5))

```

As for raster-based process, It was needed to generate a canopy height model first which came from differencing a digital surface model and a digital terrain model,

```

#Individual Tree Detection (raster based)

dtm<-rasterize_terrain(class_las,algorithm = knnidw(k = 6L, p = 2))

dsm <- rasterize_canopy(class_las)

chm<-dsm-dtm

writeRaster(chm,paste0(output,"/chm.Tif",overwrite=TRUE)) #line to export the CHM

tree_pts <- locate_trees(chm, lmf(ws = 5))

```

After the process were done, the tree points can be plotted.

```
#Plot the results

x <- plot(nlas, color="RGB",bg = "white", size = 1)

add_treetops3d(x, tree_pts)

```
Here is the result on overview,
![NEON_D16_ABBY_DP1_553000_5062000_classified_point_cloud_colorized](https://github.com/zachafif/measuretreeheight/assets/103014717/aa9ed6ae-3bc2-463b-b118-824ef32b2c2d)


The tree points can be exported into a shapefile. ST_SF() is added to convert tree points into an SF object and ST_ZM is added to convert 3D points into 2D points.

```
#Export to SHP

tree_pts_sf<-st_sf(tree_pts)

tree_pts_sf<-st_zm(tree_pts_sf)

st_write(tree_pts_sf, paste0(output,"/tree_pts.shp")) #line to export the SHP

```

Here is the tree point result loaded in QGIS overlayed on top of the CHM,
![image](https://github.com/zachafif/measuretreeheight/assets/103014717/7a6a8742-b340-442b-a5ae-b998fc950a59)

Also,  tree points can be exported into a CSV file with coordinates. The coordinates need to be extracted first before then being added to the table.

```
#Export to CSV

tree_pts_df<-as.data.frame(tree_pts_sf)

coordinates<-as.data.frame(st_coordinates(tree_pts$geometry))

tree_pts_df$X<-coordinates$X

tree_pts_df$Y<-coordinates$Y

tree_pts_df<-tree_pts_df[,c(1,4,5,2)]

write.csv(tree_pts_df,paste0(output,"/tree_pts.csv")) #line to export the CSV

```

The result will be as follows,

![image](https://github.com/zachafif/measuretreeheight/assets/103014717/024efc71-c673-454a-93f5-2d2df249161f)

The Z column in the table both in SHP and CSV is the tree height information. The information is available per tree points with a treeID as it identifier.

## Conclusion:

Tree height measurement can be done using the combination of LiDAR sensor and individual tree detection method. The process also become very convenient using lidR package with only a few lines of code and can result in detailed information as the height values are available for all tree points. However, the tree points results are not perfect, thus it need manual checking and editing in order to have a good result.

# Reference:

#> Roussel, J.R., Auty, D., Coops, N. C., Tompalski, P., Goodbody, T. R. H., Sánchez Meador, A., Bourdon, J.F., De Boissieu, F., Achim, A. (2021). lidR : An R package for analysis of Airborne Laser Scanning (ALS) data. Remote Sensing of Environment, 251 (August), 112061. <doi:10.1016/j.rse.2020.112061>.


#> Jean-Romain Roussel and David Auty (2023). Airborne LiDAR Data Manipulation and Visualization for Forestry Applications. R package version 3.1.0. https://cran.r-project.org/package=lidR
