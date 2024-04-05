---
title: "Geospatial risk modeling - Predictive Policing"
author: "Neve/Viva/Yaohan"
date: "2024-04-04"
output: 
  html_document:
    keep_md: yes
    toc: yes
    theme: flatly
    toc_float: yes
    code_folding: hide
    number_sections: no
  pdf_document:
    toc: yes
---

<style>
.kable thead tr th, .table thead tr th {
  text-align: left !important;}
table.kable, table.table {
  width: 100% !important;}
  body {
  line-height: 1.6;
  font-size: 16px
}
</style>



# Introduction

Neve, Viva and Yaohan work together on the scripts, and then finish the write-up separately.


# Data Gathering

## Chicago Neighborhoods, Police Boundaries, and Beats


```r
# Read and process Chicago boundary data
chicagoBoundary <- 
  st_read(file.path(root.dir, "/Chapter5/chicagoBoundary.geojson")) %>%  # Read Chicago boundary data
  st_transform('ESRI:102271')  # Transform coordinate reference system

# Chicago Neighborhoods
neighborhoods <- 
  st_read("https://raw.githubusercontent.com/blackmad/neighborhoods/master/chicago.geojson") %>% 
  st_transform('ESRI:102271')

# Read and process police districts data
policeDistricts <- 
  st_read("https://data.cityofchicago.org/api/geospatial/fthy-xz3r?method=export&format=GeoJSON") %>%
  st_transform('ESRI:102271') %>%  # Transform coordinate reference system
  dplyr::select(District = dist_num)  # Select only the district number, renaming it to 'District'

# Read and process police beats data
policeBeats <- 
  st_read("https://data.cityofchicago.org/api/geospatial/aerh-rz74?method=export&format=GeoJSON") %>%
  st_transform('ESRI:102271') %>%  # Transform coordinate reference system
  dplyr::select(District = beat_num)  # Select only the beat number, renaming it to 'District'

# Combine police districts and beats data into one dataframe
bothPoliceUnits <- rbind(
  mutate(policeDistricts, Legend = "Police Districts"),  # Add a 'Legend' column and label for police districts
  mutate(policeBeats, Legend = "Police Beats")  # Add a 'Legend' column and label for police beats
)
```

## Chicago Crime Data

*Pending Modification + Outcome Selection*


```r
# Read and process damagelaries data
crimes2018 <- 
  read.socrata("https://data.cityofchicago.org/resource/3i3m-jwuy.json")

crimes2018.asdf <- crimes2018 %>% 
  group_by(primary_type, description) %>% 
  tally() %>% 
  arrange(desc(n))

damage2018 <- crimes2018 %>% 
  filter(primary_type == "CRIMINAL DAMAGE" & description == "TO PROPERTY") %>%
  na.omit() %>%  # Remove rows with missing values
  st_as_sf(coords = c("location.longitude", "location.latitude"), crs = 4326, agr = "constant") %>%  # Convert to sf object with specified CRS
  st_transform('ESRI:102271') %>% # Transform coordinate reference system
  st_intersection(chicagoBoundary) %>% # Filter data within Chicago boundary
  distinct()  # Keep only distinct geometries
```

### Map of Outcome in Point Form

*Pending Modification*


```r
ggplot() + 
  geom_sf(data = chicagoBoundary) +  # Add Chicago boundary
  geom_sf(data = damage2018, colour = "red", size = 0.1, show.legend = "point") +
  labs(title = "Criminal Damage, Chicago - 2018") + 
  theme_void()  # Use a blank theme
```

![](Assignment3_files/figure-html/unnamed-chunk-1-1.png)<!-- -->


### Map of Outcome joined to Fishnet

*Pending Modification*


```r
fishnet <- 
st_make_grid(chicagoBoundary,
               cellsize = 500, 
               square = TRUE) %>%
  .[chicagoBoundary] %>%            # fast way to select intersecting polygons
  st_sf() %>%   mutate(uniqueID = 1:n())

crime_net <- 
  dplyr::select(damage2018) %>% 
  mutate(count.damage = 1) %>% 
  aggregate(., fishnet, sum) %>%
  mutate(count.damage = replace_na(count.damage, 0),
         uniqueID = 1:n(),
         cvID = sample(round(nrow(fishnet) / 24), 
                       size=nrow(fishnet), replace = TRUE))

ggplot() +
  geom_sf(data = crime_net, aes(fill = count.damage), color = NA) +
  scale_fill_viridis("Count of Criminal Damage") +
  labs(title = "Count of Criminal Damage for the fishnet") +
  theme_void()
```

![](Assignment3_files/figure-html/unnamed-chunk-2-1.png)<!-- -->

# Additional Data Processing

## Variable 1: Abandoned Cars
updated till 2020


```r
abandonCars <- 
  read.socrata("https://data.cityofchicago.org/Service-Requests/311-Service-Requests-Abandoned-Vehicles/3c9v-pnva") %>%
    # Extract the year from the creation date and filter for the year 2017
    mutate(year = substr(creation_date, 1, 4)) %>% filter(year == "2018") %>%
    # Select latitude and longitude columns and remove rows with missing values
    dplyr::select(Y = latitude, X = longitude) %>%
    na.omit() %>%
    # Convert to simple feature (sf) object with geographic coordinates
    st_as_sf(coords = c("X", "Y"), crs = 4326, agr = "constant") %>%
    # Transform coordinates to match the coordinate reference system (CRS) of the fishnet
    st_transform(st_crs(fishnet)) %>%
    # Add a legend label indicating abandoned cars
    mutate(Legend = "Abandoned_Cars")
```

## Variable 2: Abandoned Buildings
updated till 2018


```r
abandonBuildings <- 
  read.socrata("https://data.cityofchicago.org/Service-Requests/311-Service-Requests-Vacant-and-Abandoned-Building/7nii-7srd") %>%
    mutate(year = substr(date_service_request_was_received,1,4)) %>%  filter(year == "2018") %>%
    dplyr::select(Y = latitude, X = longitude) %>%
    na.omit() %>%
    st_as_sf(coords = c("X", "Y"), crs = 4326, agr = "constant") %>%
    st_transform(st_crs(fishnet)) %>%
    mutate(Legend = "Abandoned_Buildings")
```

## Variable 3: Graffiti
updated till 2018


```r
graffiti <- 
  read.socrata("https://data.cityofchicago.org/Service-Requests/311-Service-Requests-Graffiti-Removal-Historical/hec5-y4x5") %>%
    mutate(year = substr(creation_date,1,4)) %>% filter(year == "2018") %>%
    filter(where_is_the_graffiti_located_ %in% c("Front", "Rear", "Side")) %>%
    dplyr::select(Y = latitude, X = longitude) %>%
    na.omit() %>%
    st_as_sf(coords = c("X", "Y"), crs = 4326, agr = "constant") %>%
    st_transform(st_crs(fishnet)) %>%
    mutate(Legend = "Graffiti")
```


## Variable 4: Disfunctional Street Lights 
updated till 2018


```r
streetLightsOut <- 
  read.socrata("https://data.cityofchicago.org/Service-Requests/311-Service-Requests-Street-Lights-All-Out/zuxi-7xem") %>%
    mutate(year = substr(creation_date,1,4)) %>% filter(year == "2018") %>%
    dplyr::select(Y = latitude, X = longitude) %>%
    na.omit() %>%
    st_as_sf(coords = c("X", "Y"), crs = 4326, agr = "constant") %>%
    st_transform(st_crs(fishnet)) %>%
    mutate(Legend = "Street_Lights_Out")
```

## Variable 5: Sanitation Complaints
updated till 2018


```r
sanitation <-
  read.socrata("https://data.cityofchicago.org/Service-Requests/311-Service-Requests-Sanitation-Code-Complaints-Hi/me59-5fac") %>%
    mutate(year = substr(creation_date,1,4)) %>% filter(year == "2018") %>%
    dplyr::select(Y = latitude, X = longitude) %>%
    na.omit() %>%
    st_as_sf(coords = c("X", "Y"), crs = 4326, agr = "constant") %>%
    st_transform(st_crs(fishnet)) %>%
    mutate(Legend = "Sanitation")
```

## Variable 6: Liquor Retail
updated till 2023


```r
liquorRetail <- 
  read.socrata("https://data.cityofchicago.org/resource/nrmj-3kcf.json") %>%  
    filter(business_activity == "Retail Sales of Packaged Liquor") %>%
    dplyr::select(Y = latitude, X = longitude) %>%
    na.omit() %>%
    st_as_sf(coords = c("X", "Y"), crs = 4326, agr = "constant") %>%
    st_transform(st_crs(fishnet)) %>%
    mutate(Legend = "Liquor_Retail")
```

## Variable 7: Park


```r
park <- 
  read.socrata("https://data.cityofchicago.org/resource/eix4-gf83.json") %>%
  select(X = x_coord, Y = y_coord) %>%
  st_as_sf(coords = c("X", "Y"), crs = 4326, agr = "constant") %>%
    st_transform(st_crs(fishnet)) %>%
    mutate(Legend = "Park")
```

## Variable 8: Environmental Complaints
updated till 2024


```r
environ.complaint <- 
  read.socrata("https://data.cityofchicago.org/resource/fypr-ksnz.json") 

environ.complaint <- environ.complaint %>% 
  mutate(year = substr(complaint_date,1,4)) %>% filter(year == "2018") %>%
    dplyr::select(Y = latitude, X = longitude) %>%
    na.omit() %>%
    st_as_sf(coords = c("X", "Y"), crs = 4326, agr = "constant") %>%
    st_transform(st_crs(fishnet)) %>%
    mutate(Legend = "Environmental_Complaint")
```

## Multiple Map of risk factors in the fishnet


```r
vars_net <- 
  rbind(abandonCars,streetLightsOut,abandonBuildings,
        liquorRetail, graffiti, sanitation, park, environ.complaint) %>% #add on
  st_join(., fishnet, join=st_within) %>%
  st_drop_geometry() %>%
  group_by(uniqueID, Legend) %>%
  summarize(count = n()) %>%
    full_join(fishnet) %>%
    spread(Legend, count, fill=0) %>%
    st_sf() %>%
    dplyr::select(-`<NA>`) %>%
    na.omit() %>%
    ungroup()

fishnet.centroid <- st_centroid(fishnet)

vars_net.long <- vars_net %>%
  dplyr::select(- ends_with(".nn")) %>% 
  gather(Variable, value, -geometry, -uniqueID) %>%
  na.omit()

vars_risk <- unique(vars_net.long$Variable)

mapList <- list()

for(i in vars_risk)
  {mapList[[i]] <- 
    ggplot() +
      geom_sf(data = filter(vars_net.long, Variable == i), aes(fill=value), colour=NA) +
      scale_fill_viridis(name="") +
      labs(title=i) +
      mapTheme() +
      theme(plot.title = element_text(size = 10))}

do.call(grid.arrange,c(mapList, ncol=2, top="Risk Factors by Fishnet"))
```

![](Assignment3_files/figure-html/unnamed-chunk-11-1.png)<!-- -->


## Computing Additional Nearest Neighbor Variables 


```r
vars_net <-
  vars_net %>%
    mutate(
      Abandoned_Buildings.nn =
        nn_function(st_coordinates(st_centroid(vars_net)), st_coordinates(abandonBuildings),3),
      Abandoned_Cars.nn =
        nn_function(st_coordinates(st_centroid(vars_net)), st_coordinates(abandonCars),3),
      Graffiti.nn =
        nn_function(st_coordinates(st_centroid(vars_net)), st_coordinates(graffiti),3),
      Liquor_Retail.nn =
        nn_function(st_coordinates(st_centroid(vars_net)), st_coordinates(liquorRetail),3),
      Street_Lights_Out.nn =
        nn_function(st_coordinates(st_centroid(vars_net)), st_coordinates(streetLightsOut),3),
      Park.nn =
        nn_function(st_coordinates(st_centroid(vars_net)), st_coordinates(park),3),
      Sanitation.nn =
        nn_function(st_coordinates(st_centroid(vars_net)), st_coordinates(sanitation),3),
      Environmental_Complaint.nn =
        nn_function(st_coordinates(st_centroid(vars_net)), st_coordinates(environ.complaint),3))

nn.vars_net.long <- 
  dplyr::select(vars_net, ends_with(".nn")) %>%
  gather(Variable, value, -geometry)

nn.vars <- unique(nn.vars_net.long$Variable)
nn.mapList <- list()

for(i in nn.vars){
  nn.mapList[[i]] <- 
    ggplot() +
      geom_sf(data = filter(nn.vars_net.long, Variable == i), aes(fill=value), colour=NA) +
      scale_fill_viridis(name="") +
      labs(title=i) +
      mapTheme() +
      theme(plot.title = element_text(size = 10))}

do.call(grid.arrange,c(nn.mapList, ncol=2, top="Nearest Neighbor risk Factors by Fishnet"))
```

![](Assignment3_files/figure-html/unnamed-chunk-12-1.png)<!-- -->

# Moran's I-related small multiple map of outcome


```r
final_net <-
  left_join(crime_net, st_drop_geometry(vars_net), by="uniqueID") 

final_net <-
  st_centroid(final_net) %>%
    st_join(dplyr::select(neighborhoods, name)) %>%
    st_join(dplyr::select(policeDistricts, District)) %>%
      st_drop_geometry() %>%
      left_join(dplyr::select(final_net, geometry, uniqueID)) %>%
      st_sf() %>%
  na.omit()
```


```r
final_net.nb <- poly2nb(as_Spatial(final_net), queen=TRUE)
final_net.weights <- nb2listw(final_net.nb, style="W", zero.policy=TRUE)
```


```r
damage.localMorans <- 
  cbind(
    as.data.frame(localmoran(final_net$count.damage, final_net.weights, zero.policy = TRUE)),
    as.data.frame(final_net)) %>% 
    st_sf() 

#modified based on week 10 lecture
mp <- moran.plot(as.vector(scale(damage.localMorans$count.damage)), final_net.weights, zero.policy = TRUE)
```

![](Assignment3_files/figure-html/unnamed-chunk-15-1.png)<!-- -->

```r
damage.localMorans$hotspot <- 0

damage.localMorans[(mp$x >= 0 & mp$wx >= 0) & (damage.localMorans$`Pr(z != E(Ii))` <= 0.01), "hotspot"] <- 1

damage.localMorans.1 <- damage.localMorans %>% 
  dplyr::select(Damage_Count = count.damage, 
                Local_Morans_I = Ii, 
                P_Value = `Pr(z != E(Ii))`,
                hotspot) %>%
      gather(Variable, Value, -geometry)

damage.vars <- unique(damage.localMorans.1$Variable)
damage.varList <- list()

for(i in damage.vars)
  {damage.varList[[i]] <- 
    ggplot() +
      geom_sf(data = filter(damage.localMorans.1, Variable == i), 
              aes(fill = Value), colour=NA) +
      scale_fill_viridis(name="") +
      labs(title=i) +
      mapTheme() + 
      theme(legend.position = "bottom",
          plot.title = element_text(size = 10))}

do.call(grid.arrange,c(damage.varList, ncol = 4, top = "Local Morans I statistics, Criminal Damage"))
```

![](Assignment3_files/figure-html/unnamed-chunk-15-2.png)<!-- -->

# Multiple scatter plot with correlations


```r
final_net <- final_net %>% 
  mutate(damage.isSig = 
           ifelse(damage.localMorans$hotspot == 1, 1, 0)) %>%
  mutate(damage.isSig.dist = 
           nn_function(st_coordinates(st_centroid(final_net)),
                       st_coordinates(st_centroid(filter(final_net, 
                                           damage.isSig == 1))), 
                       k = 1))

correlation.long <-
  st_drop_geometry(final_net) %>%
    dplyr::select(-uniqueID, -cvID, -name, -District) %>%
    gather(Variable, Value, -count.damage)

correlation.cor <-
  correlation.long %>%
    group_by(Variable) %>%
    summarize(correlation = cor(Value, count.damage, use = "complete.obs"))


ggplot(correlation.long, aes(Value, count.damage)) +
  geom_point(size = 0.1) +
  geom_text(data = correlation.cor, aes(label = paste("r =", round(correlation, 2))),
            x = -Inf, y = Inf, vjust = 1.5, hjust = -0.1) +
  geom_smooth(method = "lm", se = FALSE, colour = "red", size = 1) +
  facet_wrap(~Variable, ncol = 2, scales="free") +
  labs(title = "Criminal damage count as a function of risk factors") +
  plotTheme() +
  theme(plot.title = element_text(size = 12),
        strip.text = element_text(size = 8), 
        plot.margin = margin(1, 1, 1.5, 1))
```

![](Assignment3_files/figure-html/unnamed-chunk-16-1.png)<!-- -->

# A Histogram of Dependent Variable

This histogram suggests an OLS regression is not appropriate method of analysis.


```r
ggplot(final_net, aes(x = count.damage)) +
  geom_histogram(binwidth = 3, fill = "grey", color = "black") +
  labs(title = "Distribution of Criminal Damage Counts, Chicago", x = "Damage Count", y = "Frequency") +
  theme_minimal()
```

![](Assignment3_files/figure-html/unnamed-chunk-17-1.png)<!-- -->

# A small multiple map of model errors by random k-fold and spatial cross validation.

## Leave One Group Out CV on spatial features


```r
## define the variables we want
reg.vars <- c(nn.vars)

reg.ss.vars <- c(nn.vars, "damage.isSig", "damage.isSig.dist")
```



```r
## RUN REGRESSIONS
reg.cv <- crossValidate(
  dataset = final_net,
  id = "cvID",
  dependentVariable = "count.damage",
  indVariables = reg.vars) %>%
    dplyr::select(cvID = cvID, count.damage, Prediction, geometry)

reg.ss.cv <- crossValidate(
  dataset = final_net,
  id = "cvID",
  dependentVariable = "count.damage",
  indVariables = reg.ss.vars) %>%
    dplyr::select(cvID = cvID, count.damage, Prediction, geometry)
  
reg.spatialCV <- crossValidate(
  dataset = final_net,
  id = "name",
  dependentVariable = "count.damage",
  indVariables = reg.vars) %>%
    dplyr::select(cvID = name, count.damage, Prediction, geometry)

reg.ss.spatialCV <- crossValidate(
  dataset = final_net,
  id = "name",
  dependentVariable = "count.damage",
  indVariables = reg.ss.vars) %>%
    dplyr::select(cvID = name, count.damage, Prediction, geometry)
```

```r
reg.summary <- 
  rbind(
    mutate(reg.cv, Error = Prediction - count.damage,
                   Regression = "Random k-fold CV: Just Risk Factors"),
                             
    mutate(reg.ss.cv, Error = Prediction - count.damage,
                      Regression = "Random k-fold CV: Spatial Process"),
    
    mutate(reg.spatialCV, Error = Prediction - count.damage,
                          Regression = "Spatial LOGO-CV: Just Risk Factors"),
                             
    mutate(reg.ss.spatialCV, Error = Prediction - count.damage,
                             Regression = "Spatial LOGO-CV: Spatial Process")) %>%
    st_sf() 
```


```r
error_by_reg_and_fold <- 
  reg.summary %>%
    group_by(Regression, cvID) %>% 
    summarize(Mean_Error = mean(Prediction - count.damage, na.rm = T),
              MAE = abs(Mean_Error), na.rm = T) %>%
  ungroup()

#remove afterwards
error_by_reg_and_fold %>%
  ggplot(aes(MAE)) + 
    geom_histogram(bins = 30, colour="black", fill = "#FDE725FF") +
    facet_wrap(~Regression) +  
    geom_vline(xintercept = 0) + scale_x_continuous(breaks = seq(0, 8, by = 1)) + 
    labs(title="Distribution of MAE", subtitle = "k-fold cross validation vs. LOGO-CV",
         x="Mean Absolute Error", y="Count") +
    plotTheme()
```

![](Assignment3_files/figure-html/unnamed-chunk-20-1.png)<!-- -->

#Map of Errors

```r
error_by_reg_and_fold %>%
  filter(str_detect(Regression, "k-fold")) %>%
  ggplot() +
    geom_sf(aes(fill = MAE)) +
    facet_wrap(~Regression) +
    scale_fill_viridis() +
    labs(title = "Damage errors by k-fold Regression") +
    theme_void()
```

![](Assignment3_files/figure-html/unnamed-chunk-21-1.png)<!-- -->

```r
error_by_reg_and_fold %>%
  filter(str_detect(Regression, "LOGO")) %>%
  ggplot() +
    geom_sf(aes(fill = MAE)) +
    facet_wrap(~Regression) +
    scale_fill_viridis() +
    labs(title = "Damage errors by LOGO-CV Regression") +
  theme_void()
```

![](Assignment3_files/figure-html/unnamed-chunk-21-2.png)<!-- -->

# A table of MAE and standard deviation MAE by regression.


```r
st_drop_geometry(error_by_reg_and_fold) %>%
  group_by(Regression) %>% 
    summarize(Mean_MAE = round(mean(MAE), 2),
              SD_MAE = round(sd(MAE), 2)) %>%
  kable() %>%
    kable_styling("striped", "hover")
```

<table class="table table-striped" style="color: black; margin-left: auto; margin-right: auto;">
 <thead>
  <tr>
   <th style="text-align:left;"> Regression </th>
   <th style="text-align:right;"> Mean_MAE </th>
   <th style="text-align:right;"> SD_MAE </th>
  </tr>
 </thead>
<tbody>
  <tr>
   <td style="text-align:left;"> Random k-fold CV: Just Risk Factors </td>
   <td style="text-align:right;"> 0.95 </td>
   <td style="text-align:right;"> 0.80 </td>
  </tr>
  <tr>
   <td style="text-align:left;"> Random k-fold CV: Spatial Process </td>
   <td style="text-align:right;"> 0.86 </td>
   <td style="text-align:right;"> 0.76 </td>
  </tr>
  <tr>
   <td style="text-align:left;"> Spatial LOGO-CV: Just Risk Factors </td>
   <td style="text-align:right;"> 2.07 </td>
   <td style="text-align:right;"> 1.87 </td>
  </tr>
  <tr>
   <td style="text-align:left;"> Spatial LOGO-CV: Spatial Process </td>
   <td style="text-align:right;"> 1.57 </td>
   <td style="text-align:right;"> 1.57 </td>
  </tr>
</tbody>
</table>

# A table of raw errors by race context for a random k-fold vs. spatial cross validation regression.

```r
tracts18 <- 
  get_acs(geography = "tract", variables = c("B01001_001E","B01001A_001E"), 
          year = 2018, state=17, county=031, geometry=T) %>%
  st_transform('ESRI:102271')  %>% 
  dplyr::select(variable, estimate, GEOID) %>%
  spread(variable, estimate) %>%
  rename(TotalPop = B01001_001,
         NumberWhites = B01001A_001) %>%
  mutate(percentWhite = NumberWhites / TotalPop,
         raceContext = ifelse(percentWhite > .5, "Majority_White", "Majority_Non_White")) %>%
  .[neighborhoods,]
```


```r
reg.summary %>% 
  filter(str_detect(Regression, "LOGO")) %>%
    st_centroid() %>%
    st_join(tracts18) %>%
    na.omit() %>%
      st_drop_geometry() %>%
      group_by(Regression, raceContext) %>%
      summarize(mean.Error = mean(Error, na.rm = T)) %>%
      spread(raceContext, mean.Error) %>%
      kable(caption = "Mean Error by neighborhood racial context, 2018") %>%
        kable_styling("striped", "hover") 
```

<table class="table table-striped" style="color: black; margin-left: auto; margin-right: auto;">
<caption>Mean Error by neighborhood racial context, 2018</caption>
 <thead>
  <tr>
   <th style="text-align:left;"> Regression </th>
   <th style="text-align:right;"> Majority_Non_White </th>
   <th style="text-align:right;"> Majority_White </th>
  </tr>
 </thead>
<tbody>
  <tr>
   <td style="text-align:left;"> Spatial LOGO-CV: Just Risk Factors </td>
   <td style="text-align:right;"> -0.9661727 </td>
   <td style="text-align:right;"> 1.0010598 </td>
  </tr>
  <tr>
   <td style="text-align:left;"> Spatial LOGO-CV: Spatial Process </td>
   <td style="text-align:right;"> -0.4408509 </td>
   <td style="text-align:right;"> 0.4422612 </td>
  </tr>
</tbody>
</table>

# Comparing Model to Traditional Methods

```r
damage_ppp <- as.ppp(st_coordinates(damage2018), W = st_bbox(final_net))
damage_KD.1000 <- density.ppp(damage_ppp, 1000)
damage_KD.1500 <- density.ppp(damage_ppp, 1500)
damage_KD.2000 <- density.ppp(damage_ppp, 2000)
damage_KD.df <- rbind(
  mutate(data.frame(rasterToPoints(mask(raster(damage_KD.1000), as(neighborhoods, 'Spatial')))), Legend = "1000 Ft."),
  mutate(data.frame(rasterToPoints(mask(raster(damage_KD.1500), as(neighborhoods, 'Spatial')))), Legend = "1500 Ft."),
  mutate(data.frame(rasterToPoints(mask(raster(damage_KD.2000), as(neighborhoods, 'Spatial')))), Legend = "2000 Ft.")) 

damage_KD.df$Legend <- factor(damage_KD.df$Legend, levels = c("1000 Ft.", "1500 Ft.", "2000 Ft."))

ggplot(data=damage_KD.df, aes(x=x, y=y)) +
  geom_raster(aes(fill=layer)) + 
  facet_wrap(~Legend) +
  coord_sf(crs=st_crs(final_net)) + 
  scale_fill_viridis(name="Density") +
  labs(title = "Kernel density with 3 different search radii") +
  mapTheme(title_size = 14)
```

![](Assignment3_files/figure-html/unnamed-chunk-24-1.png)<!-- -->

```r
as.data.frame(damage_KD.1000) %>%
  st_as_sf(coords = c("x", "y"), crs = st_crs(final_net)) %>%
  aggregate(., final_net, mean) %>%
   ggplot() +
     geom_sf(aes(fill=value)) +
     geom_sf(data = sample_n(damage2018, 1500), size = .5) +
     scale_fill_viridis(name = "Density") +
     labs(title = "Kernel density of 2017 thefts") +
     mapTheme()
```

![](Assignment3_files/figure-html/unnamed-chunk-25-1.png)<!-- -->
# Retrieving 2019 crime data

```r
damage19 <- 
  read.socrata("https://data.cityofchicago.org/resource/w98m-zvie.json")

damage19 <- damage19 %>% 
filter(primary_type == "CRIMINAL DAMAGE" & description == "TO PROPERTY") %>% 
  na.omit() %>%  # Remove rows with missing values
  st_as_sf(coords = c("location.longitude", "location.latitude"), crs = 4326, agr = "constant") %>%  # Convert to sf object with specified CRS
  st_transform('ESRI:102271') %>%  # Transform coordinate reference system
  distinct() %>%  # Keep only distinct geometries
  .[fishnet,]
```


```r
damage_KDE_sf <- as.data.frame(damage_KD.1000) %>%
  st_as_sf(coords = c("x", "y"), crs = st_crs(final_net)) %>%
  aggregate(., final_net, mean) %>%
  mutate(label = "Kernel Density",
         Risk_Category = ntile(value, 100),
         Risk_Category = case_when(
           Risk_Category >= 90 ~ "90% to 100%",
           Risk_Category >= 70 & Risk_Category <= 89 ~ "70% to 89%",
           Risk_Category >= 50 & Risk_Category <= 69 ~ "50% to 69%",
           Risk_Category >= 30 & Risk_Category <= 49 ~ "30% to 49%",
           Risk_Category >= 1 & Risk_Category <= 29 ~ "1% to 29%")) %>%
  cbind(
    aggregate(
      dplyr::select(damage19) %>% mutate(damageCount = 1), ., sum) %>%
    mutate(damageCount = replace_na(damageCount, 0))) %>%
  dplyr::select(label, Risk_Category, damageCount)

damage_risk_sf <-
  filter(reg.summary, Regression == "Spatial LOGO-CV: Spatial Process") %>%
  mutate(label = "Risk Predictions",
         Risk_Category = ntile(Prediction, 100),
         Risk_Category = case_when(
           Risk_Category >= 90 ~ "90% to 100%",
           Risk_Category >= 70 & Risk_Category <= 89 ~ "70% to 89%",
           Risk_Category >= 50 & Risk_Category <= 69 ~ "50% to 69%",
           Risk_Category >= 30 & Risk_Category <= 49 ~ "30% to 49%",
           Risk_Category >= 1 & Risk_Category <= 29 ~ "1% to 29%")) %>%
  cbind(
    aggregate(
      dplyr::select(damage19) %>% mutate(damageCount = 1), ., sum) %>%
      mutate(damageCount = replace_na(damageCount, 0))) %>%
  dplyr::select(label,Risk_Category, damageCount)

rbind(damage_KDE_sf, damage_risk_sf) %>%
  na.omit() %>%
  gather(Variable, Value, -label, -Risk_Category, -geometry) %>%
  ggplot() +
    geom_sf(aes(fill = Risk_Category), colour = NA) +
    geom_sf(data = sample_n(damage19, 3000), size = .1, colour = "black") +
    facet_wrap(~label, ) +
    scale_fill_viridis(discrete = TRUE) +
    labs(title="Comparison of Kernel Density and Risk Predictions",
         subtitle="2018 criminal risk predictions; 2019 criminal damage") +
    mapTheme()
```

![](Assignment3_files/figure-html/unnamed-chunk-27-1.png)<!-- -->


```r
rbind(damage_KDE_sf, damage_risk_sf) %>%
  st_set_geometry(NULL) %>% na.omit() %>%
  gather(Variable, Value, -label, -Risk_Category) %>%
  group_by(label, Risk_Category) %>%
  summarize(count.theft = sum(Value)) %>%
  ungroup() %>%
  group_by(label) %>%
  mutate(Rate_of_test_set_crimes = count.theft / sum(count.theft)) %>%
    ggplot(aes(Risk_Category,Rate_of_test_set_crimes)) +
      geom_bar(aes(fill=label), position="dodge", stat="identity") +
      scale_fill_viridis(discrete = TRUE) +
      labs(title = "Risk prediction vs. Kernel density, 2019 thefts") +
      plotTheme() + theme(axis.text.x = element_text(angle = 45, vjust = 0.5))
```

![](Assignment3_files/figure-html/unnamed-chunk-28-1.png)<!-- -->
