1. raw data zip: download data from https://zenodo.org/record/6398971#.Y_w6H3ZBy3A

2. raw data (nc files): uncompress the zip files

3. daily tiff: transform nc files to tiff files

   - tips: cannot install gdal package
     
     - package error: Microsoft Visual C++ 14.0 or greater is required. Get it with "Microsoft C++ Build Tools": https://visualstudio.microsoft.com/visual-cpp-build-tools/
     - solved by using anoconda installation
       - according to youtube [Install GDAL for Python with Anaconda - YouTube](https://www.youtube.com/watch?v=q5WzabB-5Z0)
       - in Pycharm settings: add the conda interpreter to build the environment

   - Python code
     
     ```python
     #;+
     #; :Author: Dr. Jing Wei (Email: weijing_rs@163.com)
     #;-
     import os
     from osgeo import gdal
     import netCDF4 as nc
     import numpy as np  
     from glob import glob
     from osgeo import osr
     
     #Define work and output paths
     WorkPath = r'F:\PM2.5\chinadaily\rawdata_nc\2008'
     OutPath  = r'F:\PM2.5\chinadaily\dailytiff\2008'
     
     #Define air pollutant type 
     #e.g., PM1, PM2.5, PM10, O3, NO2, SO2, and CO, et al.
     AP = 'PM2.5'
     
     #Define spatial resolution 
     #e.g., 1 km ≈ 0.01 Degree
     SP = 0.01 #Degrees
     
     if not os.path.exists(OutPath):
         os.makedirs(OutPath)
     path = glob(os.path.join(WorkPath, '*.nc'))
     
     for file in path:
         f = nc.Dataset(file)   
         #Read SDS data
         data = np.array(f[AP][:]) 
         #Define missing value: NaN or -999
         data[data==65535] = np.nan #-999 
         #Read longitude and latitude information
         lon = np.array(f['lon'][:])
         lat = np.array(f['lat'][:])        
         LonMin,LatMax,LonMax,LatMin = lon.min(),lat.max(),lon.max(),lat.min()    
         N_Lat = len(lat) 
         N_Lon = len(lon)
         Lon_Res = SP #round((LonMax-LonMin)/(float(N_Lon)-1),2)
         Lat_Res = SP #round((LatMax-LatMin)/(float(N_Lat)-1),2)
         #Define Define output file
         fname = os.path.basename(file).split('.nc')[0]
         outfile = OutPath + '/{}.tif' .format(fname)        
         #Write GeoTIFF
         driver = gdal.GetDriverByName('GTiff')    
         outRaster = driver.Create(outfile,N_Lon,N_Lat,1,gdal.GDT_Float32)
         outRaster.SetGeoTransform([LonMin-Lon_Res/2,Lon_Res,0,LatMax+Lat_Res/2,0,-Lat_Res])
         sr = osr.SpatialReference()
         sr.SetWellKnownGeogCS('WGS84')
         outRaster.SetProjection(sr.ExportToWkt())
         outRaster.GetRasterBand(1).WriteArray(data)
         print(fname+'.tif',' Finished')     
         #release memory
         del outRaster
         f.close()
     ```

4. zonal statistics: tiff to csv
   
   - arcgis is unavailable
     - both the iterator and zonal instrument report errors
   
   - write R code to get the results
   
     ```R
     library(terra)
     
     # Load the shapefile with polygon IDs
     shapefile_path <- "D:/PM2.5/2000china_city_map/map/city_dingel_2000.shp"
     polygons <- vect(shapefile_path)
     
     
     # Get a list of all rasters in a folder
     raster_folder <- "D:/PM2.5/chinadaily/dailytiff/2000"
     raster_files <- list.files(raster_folder, pattern = ".tif$", full.names = TRUE)
     
     
     # Loop over all raster files and extract zonal statistics
     
     for (i in seq_along(raster_files)) {
       cat("Processing raster file ", i, " of ", length(raster_files), "\n")
       raster_file <- rast(raster_files[i])
     
       # Create data frames for storing the zonal statistics
       zonal_stats1 <- zonal(raster_file, polygons, 
                                     fun = sum, na.rm = TRUE, 
                                     id = polygons$city_id)
       colnames(zonal_stats1)[1] <- "sumPM"
       zonal_stats1$city_id <- polygons$city_id
     
       zonal_stats2 <- data.frame(city_id = polygons$city_id)
       zonal_stats2 <- zonal(raster_file, polygons, 
                                     fun = mean, na.rm = TRUE, 
                                     id = polygons$city_id)
       colnames(zonal_stats2)[1] <- "meanPM"
       zonal_stats2$city_id <- polygons$city_id
     
       # Combine the two data frames and save to disk
       zonal_stats <- merge(zonal_stats1, zonal_stats2, by = "city_id")
     
       # modify the name of csv
       date <- sub(".*_(\\d{8})_.*", "\\1", basename(raster_files[i]))
       output_path <- paste0("D:/PM2.5/chinadaily/zonal/2000/", date, ".csv")
       write.csv(zonal_stats, output_path, row.names = FALSE)
     }
     ```
   
   - split the calculation to 8 computers to fasten the speed

5. modify the format (from daily_split files to a comprehensive dataset)

```R
setwd("D:/PM2.5/chinadaily/zonal")

# get a list of all the subfolders
folders <- list.dirs(full.names = TRUE)

file_list <- list()
for (folder in folders) {
  cat("Processing folder ", folder, " of ", length(folders), "\n")
  files <- list.files(path = folder, pattern = "*.csv", full.names = TRUE)
  for (file in files) {
    df <- read.csv(file)
    df$year <- as.numeric(substr(basename(file), 1, 4))
    df$month <- as.numeric(substr(basename(file), 5, 6))
    df$day <- as.numeric(substr(basename(file), 7, 8))
    
    new_order <- c("year", "month", "day", "city_id", "sumPM", "meanPM")
    df <- df[, new_order]
    
    file_list[[file]] <- df
  }
}

merged_df <- do.call(rbind, file_list)
write.csv(merged_df, "daily_summary_2000_2021.csv", row.names = FALSE)
```
