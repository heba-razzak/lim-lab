Data Preprocessing
================

## Load required libraries

``` r
library(dplyr) # For data manipulation
library(data.table) # Faster than dataframes (for big files)
library(ggplot2) # For visualizing data
library(rjson) # For working with JSON data
library(httr) # For making HTTP requests
library(sf) # For working with spatial data
library(osmdata) # Open Street Map
library(mapview) # For interactive maps
library(tidycensus) # Census data
library(lubridate) # Dates
```

### Coordinate reference system (CRS) that will be used for all shapefiles

### CRS: 4326 in degrees (sphere)

### CRS: 3857 in meters (flat map) - didnt work with getbb

``` r
crs = 4326
```

# Input dates for the month you want

``` r
year = 2019
month = 9
```

# format dates for filenames

``` r
# Create a date object for the first & last day of the month
first_day <- make_date(year, month, 1)

# Get the last day of the month
last_day <- ceiling_date(make_date(year, month, 1), "month") - days(1)

# Format the dates as strings in the desired format
purpleairformat <- paste0(format(first_day, "%Y-%m-01"), "_", format(last_day, "%Y-%m-%d"))

uberformat <- sprintf("%d-%d", year, month)
```

# read csv files

``` r
purpleair <- fread(paste0(
  "/Users/heba/Desktop/Uni/Lim Lab/Purple Air/purple_air_sanfran_",purpleairformat,".csv"))
uber_data <- fread(paste0(
  "/Users/heba/Desktop/Uni/Lim Lab/Uber/Speeds/movement-speeds-hourly-san-francisco-",uberformat,".csv"))
```

``` r
# Convert the OSM way IDs in the Uber speeds data to character type
uber_data$osm_way_id <- as.character(uber_data$osm_way_id)
```

#################################################### 

# Get sensor ids in San Francisco with Lat and Lon

#################################################### 

``` r
# Store the URL of the API endpoint to request data from for PurpleAir air quality sensors
all <- "https://api.purpleair.com/v1/sensors?fields=latitude%2C%20longitude%2C%20date_created%2C%20last_seen"

# Define the API key used to authenticate the user's request to the PurpleAir API
auth_key  <- "2C4E0A86-014A-11ED-8561-42010A800005"

# Define the header for the HTTP request to the API, including the API key and Accept content type
header = c('X-API-Key' = auth_key,'Accept' = "application/json")

# Get Purple Air data using the following steps
# Make the HTTP request to the PurpleAir API using the GET function from the httr library
# Convert the raw content returned by the API into a character string
# Convert the character string into a JSON object
# Extract the "data" element from the JSON object and convert it to a data frame
result <- GET(all, add_headers(header))
raw <- rawToChar(result$content)
json <- jsonlite::fromJSON(raw)
pa <- as.data.frame(json$data)

# Rename the columns of the PurpleAir data frame
colnames(pa) <- c("sensor_id","date_created", "last_seen", "lat", "lon")

pa <- pa %>% select(sensor_id,lat,lon)

pa <- pa %>% na.omit()

pa <- pa %>% filter(sensor_id %in% unique(purpleair$sensor_id))

dt <- st_as_sf(pa, coords=c("lon", "lat"), crs = crs)

rm(result)
rm(json)
rm(pa)
```

``` r
# Change highway column to factor (to have ordered levels)
# https://wiki.openstreetmap.org/wiki/Map_features#Highway

uber_data$highway <- factor(uber_data$highway,
                               levels=c("motorway", "trunk", "primary", "secondary", "tertiary",
                                        "construction", "unclassified", "residential",
                                        "motorway_link", "trunk_link", "primary_link", "secondary_link",
                                        "tertiary_link", "service", "pedestrian", "cycleway"))

uber_data <- uber_data %>% select(utc_timestamp, osm_way_id, speed_mph_mean, highway)
```

    path <- "/Users/heba/Desktop/Uni/Lim Lab/OSM"

    # read roads file
    filename <- file.path(path, "sanfrangrid_roads_osm.shp")
    sanfrangrid_roads <- st_read(filename)

``` r
uber_data <- uber_data %>%
  filter(utc_timestamp >= as.POSIXct("2019-09-01 00:00:00") &
         utc_timestamp < as.POSIXct("2019-09-02 00:00:00"))

purpleair <- purpleair %>%
  filter(time_stamp >= as.POSIXct("2019-09-01 00:00:00") &
         time_stamp < as.POSIXct("2019-09-02 00:00:00"))
```

    osm_uber_sf <- sanfrangrid_roads[sanfrangrid_roads$osm_id %in% unique(uber_data$osm_way_id), ]
    rm(sanfrangrid_roads)
    osm_uber_roads <- merge(osm_uber_sf, uber_data, by.x = "osm_id", by.y = "osm_way_id", all.x = TRUE)
    rm(uber_data)

``` r
purpleair_sf <- dt[dt$sensor_id %in% unique(purpleair$sensor_id), ]
rm(dt)
# purpleair_points <- merge(purpleair_sf, purpleair, by.x = "sensor_id", by.y = "sensor_id", all.x = TRUE)
# rm(purpleair)
```

``` r
# Plot purple airs map
map_sf <- mapview(purpleair_sf, col.regions = "purple", col = "purple", cex = 0.1, legend = FALSE)

# Add uber layer
map_sf <- map_sf + mapview(osm_uber_sf, col.regions="lightblue", col="lightblue")

# View map
map_sf

# Save map as html
mapshot(map_sf, url = '/Users/heba/Desktop/Uni/Lim Lab/uber_purpleair/map_sf.html')
```

############################################ 

# Create Buffers around Purple Air Sensors

############################################ 

``` r
# buffer radius in meters
buffer = 500
purpleairs_buffers <- st_buffer(purpleair_sf, dist=buffer)
```

``` r
# Save shapefile
# st_write(purpleairs_buffers, paste0(path,'purpleairs_buffers.shp'))

# purpleairs_buffers <- st_read(paste0(path,'purpleairs_buffers.shp'))

# Plot purple airs map
# alpha: line opacity
# alpha.regions: fill opacity
map_uber_pa <- mapview(purpleairs_buffers, col.regions="purple", col="purple", alpha=1, alpha.regions=0.2)

# Add uber layer (showing highway type)
map_uber_pa <- map_uber_pa + mapview(osm_uber_sf,zcol="highway", legend=TRUE)

# View map
map_uber_pa

# Save map as html
url2 = paste0(path,'map_uber_pa.html')

mapshot(map_uber_pa, url = url2)
```

``` r
map_uber_pa
```

``` r
# absolute difference threshold
threshold <- 50

# Filter out rows where absolute difference is greater than threshold
purpleair_points <- purpleair_points %>%
  filter(abs(pm2.5_atm_a - pm2.5_atm_b) <= threshold) %>%
  filter(pm2.5_atm_a<2000) %>%
  filter(pm2.5_atm_b<2000)

avg_pm25 <- purpleair_points %>%
  group_by(sensor_id) %>%
  summarize(avg_pm25 = mean(pm2.5_atm))

# rm(purpleair_points)
# Define the intervals and corresponding colors
intervals <- c(0, 12, 35.4, 55.4, Inf)
AQI <- c("Good", "Moderate", "Unhealthy for Sensitive Groups", "BAD")

# Create a new column with color intervals
avg_pm25$AQI <- cut(avg_pm25$avg_pm25, breaks = intervals, labels = AQI, include.lowest = TRUE)

# Plot the average PM2.5 for each sensor
map <- mapview(avg_pm25, zcol = "AQI", legend = TRUE)
map

custom_colors <- c("green", "yellow", "orange", "red")

# Create a new column with color intervals
avg_pm25$AQI <- cut(avg_pm25$avg_pm25, breaks = intervals, labels = AQI, include.lowest = TRUE)

# Plot the average PM2.5 for each sensor with custom colors
map <- mapview(avg_pm25, zcol = "AQI", col.regions = custom_colors, legend = TRUE)
map
```

# Get intersections of Roads & PurpleAir

``` r
# # use downloaded osm data
# path = '/Users/heba/Desktop/Uni/Lim Lab/uber_purpleair/shape/'
# roads <- st_read(paste0(path,"roads.shp"))
#
# # keep roads that are in uber data
# uber_roads_osm <- roads[roads$osm_id %in% unique(uber_data$osm_way_id), ]
#
# # keep relevent columns
# uber_roads_osm <- uber_roads_osm %>% select(osm_id, name, type)
#
# # number of osm ways: 13709
# length(unique(uber_roads_osm$osm_id))
# # number of uber ways: 71603 (larger than SF area)
# length(unique(uber_data$osm_way_id))
#
# # convert selected_ways to sf object
# uber_roads_osm <- st_transform(uber_roads_osm, crs = crs)
#
# path = '/Users/heba/Desktop/Uni/Lim Lab/uber_purpleair/'
# purpleairs_buffers <- st_read(paste0(path,'purpleairs_buffers.shp'))


# intersection between purpleair buffers and uber roads
# purpleair_uber_roads <- st_intersection(osm_uber_roads, purpleairs_buffers)
purpleair_uber_roads <- st_intersection(osm_uber_sf, purpleairs_buffers)

# save shapefile
st_write(purpleair_uber_roads, "/Users/heba/Desktop/Uni/Lim Lab/uber_purpleair/purpleair_uber_roads.shp", append=FALSE)
```

# Map of PurpleAir and Roads intersections

``` r
path = '/Users/heba/Desktop/Uni/Lim Lab/uber_purpleair/'
purpleair_uber_roads <- st_read(paste0(path,'purpleair_uber_roads.shp'))
```

``` r
mapview(purpleair_uber_roads)
```

# Length of Roads around PurpleAirs

``` r
path = '/Users/heba/Desktop/Uni/Lim Lab/uber_purpleair/'
purpleair_uber_roads <- st_read(paste0(path,'purpleair_uber_roads.shp'))

purpleair_road_length <- purpleair_uber_roads %>%
  group_by(sensr_d, type) %>%
  summarize(road_length = sum(st_length(geometry)))

st_write(purpleair_road_length, "/Users/heba/Desktop/Uni/Lim Lab/uber_purpleair/purpleair_road_length.shp", append=FALSE)
```

######################################### 

# Buildings and PurpleAir Intersections

######################################### 

    path = '/Users/heba/Desktop/Uni/Lim Lab/uber_purpleair/'
    purpleairs_buffers <- st_read(paste0(path,'purpleairs_buffers.shp'))
    path = '/Users/heba/Desktop/Uni/Lim Lab/uber_purpleair/shape/'
    buildings <- st_read(paste0(path,"buildings.shp"))

    # buildings & purple air intersections
    is_valid_buildings <- st_is_valid(buildings)
    valid_buildings <- buildings[is_valid_buildings, ]
    valid_buildings <- st_make_valid(valid_buildings)
    valid_buildings <- st_transform(valid_buildings, crs=crs)


    n <- 10000
    purpleair_buildings <- list()

    for (i in 1:ceiling(nrow(valid_buildings) / n)) {
      cat("\nIteration", i, "of", ceiling(nrow(valid_buildings) / n), "\n")
      start_idx <- (i - 1) * n + 1
      end_idx <- min(i * n, nrow(valid_buildings))

      purpleair_buildings_i <- st_intersection(valid_buildings[start_idx:end_idx, ], purpleairs_buffers)

      if (nrow(purpleair_buildings_i) > 0)  {
        # Add the results to the list
        purpleair_buildings[[i]] <- purpleair_buildings_i
      }

    }

    purpleair_buildings <- do.call(rbind, purpleair_buildings)
    # cant save it to shapefile if osm_id is numeric (??)
    purpleair_buildings$osm_id <- as.character(purpleair_buildings$osm_id)

    st_write(purpleair_buildings, "/Users/heba/Desktop/Uni/Lim Lab/uber_purpleair/purpleair_buildings.shp", append=FALSE)

# Map of PurpleAir and Buildings intersections

``` r
path = '/Users/heba/Desktop/Uni/Lim Lab/uber_purpleair/'
purpleair_buildings <- st_read(paste0(path,"purpleair_buildings.shp"))

mapview(purpleair_buildings)
```

# Get area of buildings for each PA sensor

    path = '/Users/heba/Desktop/Uni/Lim Lab/uber_purpleair/'
    purpleair_buildings <- st_read(paste0(path,"purpleair_buildings.shp"))

    purpleair_building_areas <- purpleair_buildings %>%
      group_by(sensr_d, type) %>%
      summarize(total_area = sum(st_area(geometry)))

    st_write(purpleair_building_areas, "/Users/heba/Desktop/Uni/Lim Lab/uber_purpleair/purpleair_building_areas.shp", append=FALSE)

############################################# 

# Use downloaded OSM data for San Fransisco

############################################# 

    # https://download.bbbike.org/osm/bbbike/SanFrancisco/

    path = '/Users/heba/Desktop/Uni/Lim Lab/uber_purpleair/shape/'


    # read shapefiles
    buildings <- st_read(paste0(path,"buildings.shp"))
    landuse <- st_read(paste0(path,"landuse.shp"))
    natural <- st_read(paste0(path,"natural.shp"))
    places <- st_read(paste0(path,"places.shp"))
    points <- st_read(paste0(path,"points.shp"))
    railways <- st_read(paste0(path,"railways.shp"))
    roads <- st_read(paste0(path,"roads.shp"))
    waterways <- st_read(paste0(path,"waterways.shp"))

    mapview(buildings) # buildings to get area from
    mapview(landuse)
    mapview(natural) # parks/water...
    mapview(places)
    mapview(points)
    mapview(railways) # get railways and length
    mapview(roads) # exlude footways and paths? by road type?
    mapview(waterways)

# Get Buildings & PurpleAir intersections

``` r
# use downloaded osm data
path = '/Users/heba/Desktop/Uni/Lim Lab/uber_purpleair/shape/'
buildings <- st_read(paste0(path,"buildings.shp"))

# read purple air buffers
path = '/Users/heba/Desktop/Uni/Lim Lab/uber_purpleair/'
purpleairs_buffers <- st_read(paste0(path,'purpleairs_buffers.shp'))

# intersection between purpleair buffers and san fran buildings
is_valid_buildings <- st_is_valid(buildings)
valid_buildings <- buildings[is_valid_buildings, ]
valid_buildings <- st_make_valid(valid_buildings)
purpleair_buildings <- st_intersection(valid_buildings, purpleairs_buffers)
mapview(purpleair_buildings)
purpleair_buildings <- st_intersection(buildings, purpleairs_buffers)

# save shapefile
st_write(purpleair_uber_roads, "/Users/heba/Desktop/Uni/Lim Lab/uber_purpleair/purpleair_uber_roads.shp", append=FALSE)
```

``` r
path <- "/Users/heba/Desktop/Uni/Lim Lab/OSM"

# read roads file
output_file <- file.path(path, "grid238_roads_osm.shp")
sanfrancell_roads <- st_read(output_file)

# read buildings file
output_file <- file.path(path, "grid238_buildings_osm.gpkg")
sanfrancell_buildings <- st_read(output_file)

# read trees file
output_file <- file.path(path, "grid238_trees_osm.gpkg")
sanfrancell_trees <- st_read(output_file)
```

``` r
osm_uber_sf <- sanfrancell_roads[sanfrancell_roads$osm_id %in% unique(uber_data$osm_way_id), ]
rm(sanfrancell_roads)
# osm_uber_roads <- merge(osm_uber_sf, uber_data, by.x = "osm_id", by.y = "osm_way_id", all.x = TRUE)
# rm(uber_data)

osm_uber_roads <- merge(sanfrancell_roads, uber_data, by.x = "osm_id", by.y = "osm_way_id", all.x = TRUE)
```

``` r
# tiny san fran area
# -122.4 37.742 -122.463 37.78
bbox <- c(left = -122.463, bottom = 37.742, right = -122.4, top = 37.78)
# bbox <- c(left = -122.495663, bottom = 37.714958, right = -122.399274, top = 37.787837)
x = list(rbind(c(bbox["left"],bbox["bottom"]),
               c(bbox["left"],bbox["top"]),
               c(bbox["right"],bbox["top"]),
               c(bbox["right"],bbox["bottom"]),
               c(bbox["left"],bbox["bottom"])))

# Create a polygon for san fran area
bbox_sf <- st_polygon(x)

# convert to sf object
crs = 4326
bbox_sf <- st_sfc(bbox_sf, crs=crs)
```

``` r
osm_uber_sf2 <- st_intersection(osm_uber_sf, bbox_sf)
purpleairs_buffers2 <- st_intersection(purpleairs_buffers, bbox_sf)
sanfrancell_trees2 <- st_intersection(sanfrancell_trees, bbox_sf)
purpleairs <- st_intersection(purpleair_sf, bbox_sf)

purpleair_roads <- st_intersection(osm_uber_sf2, purpleairs_buffers2)
purpleair_trees <- st_intersection(sanfrancell_trees2, purpleairs_buffers2)

sanfrancell_buildings2 <- st_intersection(sanfrancell_buildings, bbox_sf)
purpleair_buildings <- st_intersection(sanfrancell_buildings2, purpleairs_buffers2)
purpleair_buildingsss <- purpleair_buildings %>% select(building)
```

``` r
mapview(purpleairs_buffers2)
```

``` r
uber_data <- uber_data %>%
  group_by(osm_way_id) %>%
  mutate(free_flow_speed = quantile(speed_mph_mean, 0.95)) %>%
  ungroup()
uber_data$congestion_ratio <- uber_data$speed_mph_mean / uber_data$free_flow_speed

speed_map <- purpleair_roads %>% left_join(uber_data, by = c("osm_id" = "osm_way_id"))
palette3 <- colorRampPalette(c("red", "green"))

map <- mapview(purpleairs, col.regions = "purple", legend = FALSE, layer.name = "PurpleAir Sensors")
# (1 = free flow speed, <1 = congestion, >1 = faster speed)
# map1 <- map + mapview(speed_map, zcol = "congestion_ratio",
#                       col.regions = palette3, col = palette3, legend.lab = "Congestion Ratio")
map1 <- map + mapview(speed_map, zcol = "free_flow_speed",
                      col.regions = palette3, col = palette3,
                      legend.lab = "Free Flow Speed", layer.name = "Free Flow Speed")
map1
```

``` r
# map1 <- map + mapview(speed_map, zcol = "congestion_ratio",
#                       col.regions = palette3, col = palette3,
#                       legend.lab = "Congestion Ratio", layer.name = "Congestion Ratio")
# map1
```

``` r
map2 <- mapview(purpleair_buildingsss, layer.name = "Buildings", col.regions = "grey", legend = FALSE)
map2
```

``` r
ggplot(purpleair_buildingsss)+geom_sf()
```

``` r
map3 <- mapview(purpleair_trees, col.regions = "green",cex=3, legend = FALSE, layer.name = "Trees")
map1 + map2 + map3
```

``` r
map5 <- mapview(speed_map, zcol = "congestion_ratio", col.regions = palette3, col = palette3)
map5
```

``` r
# Group by sensor_id and calculate the average pm2.5_atm for each sensor
avg_pm25 <- purpleair_points %>%
  group_by(sensor_id) %>%
  summarize(avg_pm25 = mean(pm2.5_atm, na.rm = TRUE))

# Define the intervals and corresponding colors
intervals <- c(0, 12, 35.4, 55.4, Inf)
AQI <- c("Good", "Moderate", "Unhealthy for Sensitive Groups", "BAD")

# Create a new column with color intervals
avg_pm25$AQI <- cut(avg_pm25$avg_pm25, breaks = intervals, labels = AQI, include.lowest = TRUE)

# Plot the average PM2.5 for each sensor
map <- mapview(avg_pm25, zcol = "AQI", legend = TRUE)
map

custom_colors <- c("green", "yellow", "orange", "red")

# Create a new column with color intervals
avg_pm25$AQI <- cut(avg_pm25$avg_pm25, breaks = intervals, labels = AQI, include.lowest = TRUE)

# Plot the average PM2.5 for each sensor with custom colors
map <- mapview(avg_pm25, zcol = "AQI", col.regions = custom_colors, legend = TRUE)
map
# Plot the average pm2.5_atm for each sensor on a map
# mapview(avg_pm25_by_sensor, zcol = "avg_pm25", legend = TRUE)
```

``` r
sensor_with_hourly_readings <- purpleair_points %>%
  group_by(sensor_id) %>%
  filter(n_distinct(hour(time_stamp)) == 24) %>%
  slice(1)  # Select the first sensor that meets the condition

# Plot the daily trend for the selected sensor
ggplot(sensor_with_hourly_readings, aes(x = hour(time_stamp), y = pm2.5_atm)) +
  geom_line() +
  labs(x = "Hour of the Day", y = "PM2.5 ATM", title = "Daily Trend of PM2.5 for Selected Sensor") +
  theme_minimal()
```