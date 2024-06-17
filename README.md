# measuretreeheight
Trial on how to measure tree height from LiDAR data using R

## Background

Nowadays, remote sensing technology is used widely in the agriculture and forestry fields. 
One of the most common applications is tree height measurement. Some several methods and tools can be tried out for tree height measurement. In this case i will try out the Individual Tree Detection method from LiDAR data. 

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

First we will check the LiDAR point cloud, etc.

```
R CODES
```

Here is the result on overview,
![NEON_D16_ABBY_DP1_553000_5062000_classified_point_cloud_colorized](https://github.com/zachafif/measuretreeheight/assets/103014717/aa9ed6ae-3bc2-463b-b118-824ef32b2c2d)

Here is the result loaded in QGIS,
![image](https://github.com/zachafif/measuretreeheight/assets/103014717/7a6a8742-b340-442b-a5ae-b998fc950a59)

