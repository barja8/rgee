
<img src="https://raw.githubusercontent.com/ryali93/rgee/master/man/figures/logo.png" align="right" width = 15%/>

# rgee

[![Build
Status](https://travis-ci.org/ryali93/rgee.svg?branch=master)](https://travis-ci.org/ryali93/rgee)
[![AppVeyor build
status](https://ci.appveyor.com/api/projects/status/github/ryali93/rgee?branch=master&svg=true)](https://ci.appveyor.com/project/ryali93/rgee)
[![License](https://img.shields.io/badge/License-Apache%202.0-blue.svg)](https://opensource.org/licenses/Apache-2.0)

### Buildings for the Python Earth Engine API

**rgee** provides a quick-witted way to interact with Google Earth
Engine since R. The idea is not interact directly with the Earth Engine
Web REST API but, through
[`reticulate`](https://rstudio.github.io/reticulate/) call the Python
library `ee` from R.

The main user relevant functions are:

  - `ee_initialize`: Authenticate and Initialize the Earth Engine ang
    Google Drive API.
  - `ee_map`: Display a given ee\_Image, ee\_Feature,
    ee\_FeatureCollection or ee\_ImageCollection using the
    [`mapview`](https://github.com/r-spatial/mapview) style.
  - `ee_download_*`: Move the results of an Earth Engine task to export
    to Hard disk.
  - `ee_manage_*`: Google Earth Engine Batch Asset Manager.
  - `ee_as_eegeom`: Convert foreign R geometry object to an Spatial
    Earth Engine object.
  - `ee_print`: Fetch and return metadata about Earth Engine Objects.
  - `ee_upload`: Batch uploading to the asset using RSelenium.

## Installation (Not available yet)

For CRAN release version of **rgee** use

``` r
install.packages("rgee")
```

To install the development version install the
[devtools](https://cran.r-project.org/package=devtools)
package.

``` r
devtools::install_github("ryali93/rgee")
```

## How does it works?

![workflow](https://raw.githubusercontent.com/ryali93/rgee/master/man/figures/rgee.png)

## Example

``` r
library(rgee)
library(readr)
library(stars)
library(sf)
library(mapview)

ee_initialize()

# Communal Reserve Amarakaeri - Peru
xmin <- -71.132591318
xmax <- -70.953664315
ymin <- -12.892451233
ymax <- -12.731116372
x_mean <- (xmin + xmax) / 2
y_mean <- (ymin + ymax) / 2

ROI <- c(xmin, ymin, xmin, ymax, xmax, ymax, xmax, ymin, xmin, ymin)
ROI_polygon <- matrix(ROI, ncol = 2, byrow = TRUE) %>%
  list() %>%
  st_polygon() %>%
  st_sfc() %>%
  st_set_crs(4326)
ee_geom <- ee_as_eegeom(ROI_polygon)


# Get the mean annual NDVI for 2011
cloudMaskL457 <- function(image) {
  qa <- image$select("pixel_qa")
  cloud <- qa$bitwiseAnd(32L)$
    And(qa$bitwiseAnd(128L))$
    Or(qa$bitwiseAnd(8L))
  mask2 <- image$mask()$reduce(ee_Reducer()$min())
  image <- image$updateMask(cloud$Not())$updateMask(mask2)
  image$normalizedDifference(list("B4", "B3"))
}

ic_l5 <- ee_ImageCollection("LANDSAT/LT05/C01/T1_SR")$
  filterBounds(ee_geom)$
  filterDate("2011-01-01", "2011-12-31")$
  map(cloudMaskL457)
mean_l5 <- ic_l5$mean()$rename("NDVI")
mean_l5 <- mean_l5$reproject(crs = "EPSG:4326", scale = 500)
mean_l5_Amarakaeri <- mean_l5$clip(ee_geom)
```

``` r
# Download -> ee$Image
task_img <- ee$batch$Export$image$toDrive(
  image = mean_l5_Amarakaeri,
  description = "Amarakaeri_task_1",
  folder = "Amarakaeri",
  fileFormat = "GEOTIFF",
  fileNamePrefix = "NDVI_l5_2011_Amarakaeri"
)
task_img$start()
ee_download_monitoring(task_img, eeTaskList = TRUE) # optional
img <- ee_download_drive(
  task = task_img,
  filename = "amarakaeri.tif",
  overwrite = T
)
plot(img)

# Download -> ee$FeatureCollection
amk_f <- ee$FeatureCollection(list(ee$Feature(ee_geom, list(name = "Amarakaeri"))))
amk_fc <- ee$FeatureCollection(amk_f)
task_vector <- ee$batch$Export$table$toDrive(
  collection = amk_fc,
  description = "Amarakaeri_task_2",
  folder = "Amarakaeri",
  fileFormat = "GEOJSON",
  fileNamePrefix = "geometry_Amarakaeri"
)

task_vector$start()
ee_download_monitoring(task_vector, eeTaskList = TRUE) # optional
amk_geom <- ee_download_drive(
  task = task_vector,
  filename = "amarakaeri.geojson",
  overwrite = TRUE
)

plot(amk_geom$geometry, add = TRUE, border = "red", lwd = 3)
py_help(ee$batch$Export$table$toDrive)
```

## Contact

Please file bug reports and feature requests at
<https://github.com/ryali93/rgee/issues>

In case of Pull Requests, please make sure to submit them to the develop
branch of this repository.
