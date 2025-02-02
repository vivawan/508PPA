---
title: "Property Value Prediction Model for Seattle"
author: "Viva/Yaohan"
date: "2024-04-01"
output: 
  html_document:
    keep_md: yes
    toc: yes
    theme: flatly
    toc_float: yes
    code_folding: hide
    number_sections: yes
    font: 12pt
  pdf_document:
    toc: yes
---

<style>
.kable thead tr th, .table thead tr th {
  text-align: left !important;}
table.kable, table.table {
  width: 100% !important;}
</style>




# Introduction

**Viva and Yaohan work together on the scripts, and then finish the write-up separately.**

- Analysis goal:

- Methods for data collection:

- Data source:

- Study area:  
Housing unit in Seattle, exclude one housing unit with an extremely high price over 7 million.

- Import all datasets

```r
# read Seattle boundary
seattle <- st_read(here::here("data/raw/Boundary/Seattle_City.geojson")) %>%
  st_union() %>%
  st_as_sf() 

# import house data
kc_hh<- read_csv(here::here("data/raw/kc_house_data.csv"))

# load census tract data
census_api_key("3ec2bee8c227ff3f9df970d0ffbb11ee1384076e", install = TRUE, overwrite = TRUE)

acs_variable_list.2015 <- load_variables(2015, # year
                                         "acs5", # five year ACS estimates
                                         cache = TRUE)

acs_vars <- c("B01003_001E", # total population
              "B01001A_001E", # white alone
              "B01001_003E", # male under 5
              "B01001_004E", # male 5-9
              "B01001_005E", # male 10-14
              "B01001_020E", # male 65-66
              "B01001_021E", # male 67-69
              "B01001_022E", # male 70-74
              "B01001_023E", # male 75-79
              "B01001_024E", # male 80-84
              "B01001_025E", # male over 85
              "B01001_027E", # female under 5
              "B01001_028E", # female 5-9
              "B01001_029E", # female 10-14
              "B01001_044E", # female 65-66
              "B01001_045E", # female 67-69
              "B01001_046E", # female 70-74
              "B01001_047E", # female 75-79
              "B01001_048E", # female 80-84
              "B01001_049E", # female over 85
              "B15003_001E", # educational attainment over 25
              "B15003_022E", # bachelor's degree
              "B19013_001E", # median household income
              "B23025_004E", # employed labor force
              "B23025_003E", # total labor force
              "B17020_002E") # income below poverty level

# read amenities data
sub <- st_read(here::here("data/raw/Amenities/Metro_Sub_Stations_in_King_County___sub_stations_point.geojson"))

sch <- st_read(here::here("data/raw/Amenities/Seattle_School_Board_Director_Districts___dirdst_area.geojson"))

park<-st_read(here::here("data/raw/Amenities/Parks_in_King_County___park_area.geojson"))

tree_canopy_2016 <- st_read(here::here("data/raw/Amenities/Tree_Canopy.geojson"))

med <- st_read(here::here("data/raw/Amenities/Medical_Facilities_including_Hospitals___medical_facilities_point.geojson"))

mark<- st_read(here::here("data/raw/Amenities/King_County_Landmarks___landmark_point.geojson"))

# read spatial data
neigh_large <- st_read(here::here("data/raw/Boundary/Neighborhood_Map_Atlas_Districts.geojson")) 

neigh_small <- st_read(here::here("data/raw/Boundary/Neighborhood_Map_Atlas_Neighborhoods.geojson"))
```

- Plot house locations in Seattle

```r
# create house location
kc_hh <- kc_hh %>%
  st_as_sf(coords = c("long", "lat"), crs = 4326, agr = "constant") %>%  # convert to sf object with specified CRS
  st_transform(st_crs(seattle)) %>%  # transform coordinate reference system to "Washington State Plane North"
  distinct()  # keep only distinct geometries

hh <- kc_hh %>%
  st_intersection(seattle)

# examine house id
hh.id<-length(unique(hh$id))#6691 < 6740

# add an unique key
hh$key <- 1:nrow(hh)

# exclude outliers with extremely high price
hh <- hh %>%
  filter(!price > 5000000)

ggplot() + 
  geom_sf(data = seattle, aes(fill = "Seattle Boundary"), color = NA) +  # Use fill for polygon, label "Legend"
  geom_sf(data = hh, aes(color = "Housing Units", shape = "Housing Units"), size = 0.5) +  # Use color and shape for points, label "Legend"
  labs(title = "Housing Unit Locations in Seattle", 
       color = "Legend",  # This now acts as the legend title for both fill and color
       fill = NULL,  # Hide separate fill legend
       shape = NULL) +  # Hide separate shape legend, if unnecessary
  scale_fill_manual(values = c("Seattle Boundary" = "grey90")) +  # Set polygon fill color
  scale_color_manual(values = c("Housing Units" = "#2166ac"), name = "Legend") +  # Set point color and unified legend title
  scale_shape_manual(values = c("Housing Units" = 16), guide = FALSE) +  # Set point shape, hide shape guide if it's redundant
  theme_void() +
  theme(plot.title = element_text(hjust = 0.5, face = "bold"),
        legend.position = "right")  # Customize legend position
```

![](Midterm_files/figure-html/house_location-1.png)<!-- -->


# Data Description

## Variable Selection
- Four categories
- First selection based on literature theory
- Category reclassification based on plot

**1. Internal Characteristics**  
- Year used (continuous)  
- Renovation (dummy)  
- Bedroom (continuous + category)  
- Bathroom (continuous + category)  
- Floors (continuous + category)  
- Living area (continuous)  
- Lot area (continuous)  
- Waterfront (dummy)  
- View (category)  
- Condition (category)  
- Grade (category)


```r
# add year_used and renovation and ensure categorical data are factor
house <- hh %>% 
  mutate(year_used = 2015 - yr_built, # used year
         reno_dum = as.factor(if_else(yr_renovated>0, 1, 0)),#renovation yes or no
         water_dum = as.factor(waterfront),
         view_cat = as.factor(view),
         condition_cat = as.factor(condition),
         grade_cat = as.factor(grade))

# create categorical data by the mean of price
## bed categories
house$bed.factor <- factor(house$bedrooms, levels =sort(unique(house$bedrooms)))

plotMean.bedrooms <- house %>%
  st_drop_geometry() %>%
  group_by(bed.factor)%>%
  summarize(price_m = mean(price))%>%
  ggplot(aes(x = bed.factor, y = price_m)) +
  geom_col(position = "dodge")+
  plotTheme() + theme(axis.text.x = element_text(angle = 45, hjust = 1)) #0-3,4-7,8+

## bathroom category
house$bath.factor <- factor(house$bathrooms, levels =sort(unique(house$bathrooms)))

plotMean.bathrooms<-house %>%
  st_drop_geometry() %>%
  group_by(bath.factor)%>%
  summarize(price_m = mean(price))%>%
  ggplot(aes(x = bath.factor, y = price_m)) +
  geom_col(position = "dodge")+
  plotTheme() + theme(axis.text.x = element_text(angle = 45, hjust = 1)) #0-4, 4+

## floor category
house$floor.factor <- factor(house$floors, levels =sort(unique(house$floors)))

plotMean.floors <- house %>%
  st_drop_geometry() %>%
  group_by(floor.factor)%>%
  summarize(price_m = mean(price))%>%
  ggplot(aes(x = floor.factor, y = price_m)) +
  geom_col(position = "dodge")+
  plotTheme() + theme(axis.text.x = element_text(angle = 45, hjust = 1)) #1-2+3, 2.5+3.5

## reclassify grade
plotMean.grade <- house %>%
  st_drop_geometry() %>%
  group_by(grade)%>%
  summarize(price_m = mean(price))%>%
  ggplot(aes(x = grade, y = price_m)) +
  geom_col(position = "dodge")+
  plotTheme() + theme(axis.text.x = element_text(angle = 45, hjust = 1))#4-9, 10-14

# add categories
house <- house %>%
  mutate(
    bed_cat = factor(case_when(
    bedrooms <=3 ~ "few",
    bedrooms >3 & bedrooms <= 7 ~ "medium",
    bedrooms >=8 ~ "many"
    )),
    bath_dum = factor(case_when(
    bathrooms <= 4 ~ "few",
    bathrooms > 4 ~ "many"
    )),
    floor_cat = factor(case_when(
    floors <= 2 | floors == 3 ~ "regular",
    floors %in% c(2.5, 3.5) ~ "irregular"
    )),
    grade_dum = factor(ifelse(grade <= 9, "low","high"), 
                       levels = c("low","high")))

# select variables
house <- house %>%
  select("key", "price","year_used","reno_dum",
         "bedrooms", "bed_cat", "bathrooms", "bath_dum",
         "sqft_living", "sqft_lot", "floor_cat",
         "water_dum", "view_cat", "condition_cat", "grade_dum")
```

**2. Socio-economic Characteristics**  
- Population density (continuous)  
- White population share (continuous)  
- Age structure (continuous)  
- Education level (continuous + dummy)  
- Median household income (continuous + dummy)  
- Employment rate (continuous + dummy)  
- Poverty rate (continuous + dummy)  


```r
acsTractsSeattle.2015 <- get_acs(geography = "tract",
                             year = 2015, 
                             variables = acs_vars,
                             geometry = TRUE,
                             state = "Washington", 
                             county = "King",
                             output = "wide") %>%
  st_transform(st_crs(seattle)) %>%
  select(GEOID, NAME, all_of(acs_vars)) %>%
  rename(total_pop = B01003_001E,
          white_pop = B01001A_001E,
          edu_bach = B15003_022E,
          edu_attain = B15003_001E,
          median_hh_income = B19013_001E,
          total_labor = B23025_003E,
          employ_labor = B23025_004E,
          poverty = B17020_002E) %>%
  mutate(area = st_area(.)) %>%
  mutate(area = set_units(x = area, value = "acres"))%>%
  mutate(pop_den = ifelse(as.numeric(area) > 0, total_pop / area, 0),
         white_share = round(ifelse(total_pop > 0, white_pop / total_pop, 0) * 100, digits = 2),
         pop_under14 = B01001_003E + B01001_004E + B01001_005E + B01001_027E +
            B01001_028E + B01001_029E,
         pop_over65 = B01001_020E + B01001_021E + B01001_022E + B01001_023E +
            B01001_024E + B01001_025E + B01001_044E + B01001_045E + B01001_046E +
            B01001_047E + B01001_048E + B01001_049E,
         total_dep = round((pop_under14 + pop_over65) / (total_pop -
                               (pop_under14 + pop_over65)) * 100, digits = 2),
         elder_dep = round(pop_over65 / (total_pop - (pop_under14 + pop_over65)) * 100, digits = 2),
         bach_share = round(ifelse(edu_attain > 0, edu_bach/edu_attain, 0) * 100, digits = 2),
         employ_rate = round(ifelse(total_labor > 0, employ_labor / total_labor, 0) * 100, digits = 2),
         pover_rate = round(ifelse(total_pop > 0, poverty / total_pop, 0) * 100, digits = 2))

acsTractsSeattle.2015 <- acsTractsSeattle.2015 %>%
  mutate(bach_dum = factor(ifelse(bach_share > mean(acsTractsSeattle.2015$bach_share,
                                                        na.rm = TRUE), "above", "below"),
                           levels = c("below", "above")),
         median_hh_dum = factor(ifelse(median_hh_income > mean(acsTractsSeattle.2015$median_hh_income,
                                                        na.rm = TRUE), "above", "below"),
                           levels = c("below", "above")),
         employ_dum = factor(ifelse(employ_rate > mean(acsTractsSeattle.2015$employ_rate,
                                                        na.rm = TRUE), "above", "below"),
                           levels = c("below", "above")),
         pover_dum = factor(ifelse(pover_rate > mean(acsTractsSeattle.2015$pover_rate,
                                                        na.rm = TRUE), "above", "below"),
                           levels = c("below", "above")))%>%
  select(GEOID, NAME, pop_den, white_share, total_dep, elder_dep, bach_share, bach_dum, median_hh_income,
         median_hh_dum, employ_rate, employ_dum, pover_rate, pover_dum)

# assign census tract characteristics to house
house <- house %>%
  st_join(., acsTractsSeattle.2015) 

# remove NA, one house may outside the boundary of census tracts
house <- house %>%
  filter(!is.na(pop_den)) 

# frequency of dummy variables
table(house$bach_dum)
table(house$median_hh_dum)
table(house$employ_dum)
table(house$pover_dum)
```

**3. Amenities Services**  
- Subway Station (distance + category)  
- School District (category)  
- Park (area + count)  
- Medical Facilities (distance + category)  
- Commercial center (distance + count)  
- Crime rate (distance + )


```r
# subway station
sub <- sub %>%
  st_transform(st_crs(house))

## the distance to the nearest station 
house <-  house %>%
      mutate(sub_dis = nn_function(st_coordinates(house),
                                      st_coordinates(sub), k = 1))

## categories based on distance
house <- house %>%
  mutate(sub_cat = factor(case_when(
    sub_dis <= 2640 ~ "within0.5mile",
    sub_dis > 2640 & sub_dis <= 5280 ~ "0.5-1mile",
    sub_dis > 5280 ~ "1+mile"
  ),levels = c("within0.5mile","0.5-1mile","1+mile")))


# school district
sch <- sch %>%
  st_transform(st_crs(house))

## add the school district variable
house <- house %>% 
  st_join(sch%>%select(DIRDST), join = st_within)%>%
  rename(sch_cat = "DIRDST")%>%
  mutate(sch_cat = factor(sch_cat, levels = c("DD1","DD2","DD3","DD4","DD5","DD6","DD7"))) 


# park
park <- park %>%
  st_transform(st_crs(house))%>%
  st_intersection(seattle)%>%
  mutate(park_area = st_area(.))%>%
  mutate(park_area = set_units(x = park_area, value = "acres"))

## area and count of parks within 500 feet
house_parks <- st_join(st_buffer(house, dist = 500), park, join = st_intersects)%>%
  group_by(key) %>% 
  summarise(park_c = n_distinct(SITENAME, na.rm = TRUE),
            sum_park_area = sum(park_area, na.rm = TRUE))
house_parks$all_park_area <- as.numeric(house_parks$sum_park_area)
house_parks$park_cat <- as.factor(house_parks$park_c)

house <- house %>% 
    left_join(house_parks%>%
                st_drop_geometry()%>%
                select(key,park_cat, all_park_area), by = "key")%>%
  rename(parks_area = all_park_area)

# tree canopy
tree_canopy_2016 <- tree_canopy_2016 %>%
  st_transform(st_crs(house)) %>%
  mutate(tree_canopy = round(TreeCanopy_2016_Percent, digits = 2)) %>%
  select(tree_canopy)

house <- house %>%
  st_join(., tree_canopy_2016)


# medical facilities
med <- med %>%
  st_transform(st_crs(house))%>%
  st_intersection(seattle)

## calculate the distance to the nearest medical facilities 
house <-  house %>%
      mutate(med_dis1 = nn_function(st_coordinates(house),
                                      st_coordinates(med), k = 1),
             med_dis2 = nn_function(st_coordinates(house),
                                      st_coordinates(med), k = 2),
             med_dis3 = nn_function(st_coordinates(house),
                                      st_coordinates(med), k = 3))

## categories based on distance
house <- house %>%
  mutate(med_cat = case_when(
    med_dis1 <= 2640 ~ "within0.5mile",
    med_dis1 > 2640 & med_dis1 <= 5280 ~ "0.5-1mile",
    med_dis1 > 5280 ~ "1+mile"))


# commercial
mark <- mark %>%
  st_transform(st_crs(house))%>%
  st_intersection(seattle)

## select shopping center from landmrak dataset
mark_shop <- mark%>%
  filter(CODE == 690)

## calculate the distance to the nearest 1/2/3 shopping center(s) 
house <- house %>%
      mutate(shop_dis1 = nn_function(st_coordinates(house),
                                      st_coordinates(mark_shop), k = 1),
             shop_dis2 = nn_function(st_coordinates(house),
                                      st_coordinates(mark_shop), k = 2),
             shop_dis3 = nn_function(st_coordinates(house),
                                      st_coordinates(mark_shop), k = 3))

## categories based on distance
house <- house %>%
  mutate(shop_cat = factor(case_when(
    shop_dis1 <= 2640 ~ "within0.5mile",
    shop_dis1 > 2640 & shop_dis1 <= 5280 ~ "0.5-1mile",
    shop_dis1 > 5280 ~ "1+mile"
  ),levels = c("within0.5mile","0.5-1mile","1+mile")))


# crime

# crime <- read.csv(here::here("data/raw/Amenities/SPD_Crime_Data__2008-Present_20240328.csv"))
# get the target crime
# crime_clean <- crime %>%
#   mutate(year = str_sub(Report.Number, 1, 4)) %>%
#   filter(year %in% c("2013", "2014", "2015") )%>% #choose those before and in 2015
#   filter(!Offense %in% c(
#     "Bad Checks",
#     "Bribery",
#     "Embezzlement",
#     "Extortion/Blackmail",
#     "Credit Card/Automated Teller Machine Fraud",
#     "False Pretenses/Swindle/Confidence Game",
#     "Identity Theft",
#     "Impersonation",
#     "Welfare Fraud",
#     "Wire Fraud",
#     "Curfew/Loitering/Vagrancy Violations",
#     "Driving Under the Influence",
#     "Drug Equipment Violations",
#     "Drug/Narcotic Violations",
#     "Betting/Wagering",
#     "Gambling Equipment Violation",
#     "Operating/Promoting/Assisting Gambling",
#     "Liquor Law Violations",
#     "Pornography/Obscene Material",
#     "Assisting or Promoting Prostitution",
#     "Prostitution",
#     "Weapon Law Violations"
#   ))%>% #exclude those with little impact on housing price e.g.Financial Crimes, Public Order Offenses
#   filter(Longitude != 0 & Latitude != 0)# select the valid ones
#
# write.csv(crime_clean , here::here("data/processed/crime_clean.csv"), row.names = FALSE)

## read the cleaned data set
crime <- read.csv(here::here("data/processed/crime_clean.csv")) %>%
  st_as_sf(coords = c("Longitude", "Latitude"), crs = 4326)%>%
  st_transform(st_crs(house))

## count of crime within 1/8 mi
house$crime_c <- house %>%
    st_buffer(660) %>%
    aggregate(mutate(crime, counter = 1)%>%select(counter),., sum) %>%
    pull(counter)

## calculate the distance to the nearest 1/2/3/4/5 crime locations
house <-  house %>%
      mutate(crime_dis1 = nn_function(st_coordinates(house),
                                      st_coordinates(crime), k = 1),
             crime_dis2 = nn_function(st_coordinates(house),
                                      st_coordinates(crime), k = 2),
             crime_dis3 = nn_function(st_coordinates(house),
                                      st_coordinates(crime), k = 3),
             crime_dis4 = nn_function(st_coordinates(house),
                                      st_coordinates(crime), k = 4),
             crime_dis5 = nn_function(st_coordinates(house),
                                      st_coordinates(crime), k = 5))
```

**4. Spatial Structure**  
- Large District  
- Small Neighborhood  
- Census Tract


```r
# large District
neigh_large <- neigh_large%>%
  st_transform(st_crs(house)) %>%
  rename(L_NAME = L_HOOD) %>%
  select(L_NAME)

# small neighborhoods
neigh_small<- neigh_small%>%
  st_transform(st_crs(house)) %>%
  rename(S_NAME = S_HOOD) %>%
  select(S_NAME)

# census tracts
neigh_tract <- acsTractsSeattle.2015 %>%
  select(NAME) %>%
  rename(T_NAME = NAME) %>%
  st_intersection(seattle)
```

## Continuous Variable

- Exclude outliers based on scatter plots

```r
# exclude outlier
house <- house%>%
  filter(bedrooms<30 & crime_dis1 < 750 & bathrooms < 6)

# plot the final continuous variable
st_drop_geometry(house) %>% 
  select(-key)%>%
  select_if(is.numeric) %>%
  gather(Variable, Value, -price) %>% 
   ggplot(aes(Value, price)) +
     geom_point(size = .5) + geom_smooth(method = "lm", se=F, colour = "#FA7800") +
     facet_wrap(~Variable, ncol = 3, scales = "free") +
     labs(title = "Price as a Function of Continuous Variables") +
  theme(text = element_text(size = 12), # Default text size for all text
          plot.title = element_text(size = 12, face = "bold"), # Title
          axis.text = element_text(size = 8), # Axis text
          axis.title = element_text(size = 8), # Axis titles
          strip.text = element_text(size = 8)) # Facet label text
```

![](Midterm_files/figure-html/continuous_clean-1.png)<!-- -->

- Statistical summary

```r
# continuous variables
continuous_summary <- house %>%
  st_drop_geometry() %>%
  select(- key)%>%
  summarise(across(where(is.numeric),
                   list(max = ~ max(., na.rm = TRUE),
                        min = ~ min(., na.rm = TRUE),
                        mean = ~ mean(., na.rm = TRUE),
                        st.dev. = ~ sd(., na.rm = TRUE),
                        n = ~ sum(!is.na(.x))),
                   .names = "{.col}:{.fn}")) %>%
  pivot_longer(cols = everything(), names_to = c("variables", "statistic"),
               names_sep = ":", values_to = "value") %>%
  mutate(value = round(value, digits = 0)) %>%
  pivot_wider(names_from = statistic, values_from = value)

continuous_description <- data.frame(
  variables = continuous_summary$variables,
  category = rep(c("dependent","internal", "socio-economic", "amenities"),
                 times = c(1, 5, 8, 15)),
  description = c("Price: price of each unit",
                  "Year Used: years from built to 2015",
                  "No.bedrooms: the number of bedrooms in each unit",
                  "No.bathrooms: the number of bathrooms in each unit",
                  "Living Area: the area of living of each unit",
                  "Lot Area: the area of the lot of each unit",
                  "Population Density: the number of population per acre in the census tract",
                  "White Population Share:the ratio of white people to total population in the census tract",
                  "Total Dependency Ratio: the ratio of the number of children (0-14 years old) and older persons (65 years or over) to the working-age population (15-64 years old) in the census tract",
                  "Elderly Dependency Ratio: the ratio of older persons (65 years or over) to the working-age population (15-64 years old) in the census tract",
                  "Bachelor's Degree Rate: the percentage of with a bachelor's degree among adults age 25 and older in the census tract",
                  "Median Household Income: median househhold income in the census tract",
                  "Employment Rate: the ratio of the employed to the working age population in the census tract",
                  "Poverty Rate: the ratio of the number of people (in a given age group) whose income falls below the poverty line to total population in the census tract",
                  "Nearest Subway Distance: the distance to the nearest subway station",
                  "Parks' Area 500ft: the total area of parks located within a 500-foot radius of each unit",
                  "Tree Canopy Ratio: the ratio of the area of tree canopy to the total area in the measuring space",
                  "Nearest Medical Distance: the distance to the nearest medical facility",
                  "Average Distance to 2 Medicals: the average distance to the nearest 2 medical facilities",
                  "Average Distance to 3 Medicals:the average distance to the nearest 3 medical facilities",
                  "Nearest shopping Distance:the distance to the nearest shopping center",
                  "Average Distance to 2 Shoppings:the average distance to the nearest 2 shopping center",
                  "Average Distance to 3 Shoppings:the average distance to the nearest 3 shopping center",
                  "No.crime: the number of crimes within a 1/8-mile radius around each unit",
                  "Nearest Crime Distance: the distance to the nearest crime",
                  "Average Distance to 2 Crimes:the average distance to the nearest 2 crime",
                  "Average Distance to 3 Crimes:the average distance to the nearest 3 crime",
                  "Average Distance to 4 Crimes:the average distance to the nearest 4 crime",
                  "Average Distance to 5 Crimes:the average distance to the nearest 5 crime"),
  unit = c("$",
           "year", "#","#","sqft","sqft",
           "person / acre", "%","%","%","%","$","%","%",
           "feet","acre","%","feet","feet","feet","feet","feet","feet",
           "-", "feet","feet","feet","feet","feet"
           ))

continuous_description %>%
  left_join(continuous_summary, by = "variables")%>%
  kable() %>%
  kable_styling(bootstrap_options = c("striped", "hover"),
                full_width = T) %>%
    column_spec(1:9, extra_css = "text-align: left;")
```

<table class="table table-striped table-hover" style="margin-left: auto; margin-right: auto;">
 <thead>
  <tr>
   <th style="text-align:left;"> variables </th>
   <th style="text-align:left;"> category </th>
   <th style="text-align:left;"> description </th>
   <th style="text-align:left;"> unit </th>
   <th style="text-align:right;"> max </th>
   <th style="text-align:right;"> min </th>
   <th style="text-align:right;"> mean </th>
   <th style="text-align:right;"> st.dev. </th>
   <th style="text-align:right;"> n </th>
  </tr>
 </thead>
<tbody>
  <tr>
   <td style="text-align:left;text-align: left;"> price </td>
   <td style="text-align:left;text-align: left;"> dependent </td>
   <td style="text-align:left;text-align: left;"> Price: price of each unit </td>
   <td style="text-align:left;text-align: left;"> $ </td>
   <td style="text-align:right;text-align: left;"> 3800000 </td>
   <td style="text-align:right;text-align: left;"> 90000 </td>
   <td style="text-align:right;text-align: left;"> 589144 </td>
   <td style="text-align:right;text-align: left;"> 340388 </td>
   <td style="text-align:right;text-align: left;"> 6734 </td>
  </tr>
  <tr>
   <td style="text-align:left;text-align: left;"> year_used </td>
   <td style="text-align:left;text-align: left;"> internal </td>
   <td style="text-align:left;text-align: left;"> Year Used: years from built to 2015 </td>
   <td style="text-align:left;text-align: left;"> year </td>
   <td style="text-align:right;text-align: left;"> 115 </td>
   <td style="text-align:right;text-align: left;"> 0 </td>
   <td style="text-align:right;text-align: left;"> 62 </td>
   <td style="text-align:right;text-align: left;"> 35 </td>
   <td style="text-align:right;text-align: left;"> 6734 </td>
  </tr>
  <tr>
   <td style="text-align:left;text-align: left;"> bedrooms </td>
   <td style="text-align:left;text-align: left;"> internal </td>
   <td style="text-align:left;text-align: left;"> No.bedrooms: the number of bedrooms in each unit </td>
   <td style="text-align:left;text-align: left;"> # </td>
   <td style="text-align:right;text-align: left;"> 11 </td>
   <td style="text-align:right;text-align: left;"> 0 </td>
   <td style="text-align:right;text-align: left;"> 3 </td>
   <td style="text-align:right;text-align: left;"> 1 </td>
   <td style="text-align:right;text-align: left;"> 6734 </td>
  </tr>
  <tr>
   <td style="text-align:left;text-align: left;"> bathrooms </td>
   <td style="text-align:left;text-align: left;"> internal </td>
   <td style="text-align:left;text-align: left;"> No.bathrooms: the number of bathrooms in each unit </td>
   <td style="text-align:left;text-align: left;"> # </td>
   <td style="text-align:right;text-align: left;"> 5 </td>
   <td style="text-align:right;text-align: left;"> 0 </td>
   <td style="text-align:right;text-align: left;"> 2 </td>
   <td style="text-align:right;text-align: left;"> 1 </td>
   <td style="text-align:right;text-align: left;"> 6734 </td>
  </tr>
  <tr>
   <td style="text-align:left;text-align: left;"> sqft_living </td>
   <td style="text-align:left;text-align: left;"> internal </td>
   <td style="text-align:left;text-align: left;"> Living Area: the area of living of each unit </td>
   <td style="text-align:left;text-align: left;"> sqft </td>
   <td style="text-align:right;text-align: left;"> 7880 </td>
   <td style="text-align:right;text-align: left;"> 370 </td>
   <td style="text-align:right;text-align: left;"> 1799 </td>
   <td style="text-align:right;text-align: left;"> 799 </td>
   <td style="text-align:right;text-align: left;"> 6734 </td>
  </tr>
  <tr>
   <td style="text-align:left;text-align: left;"> sqft_lot </td>
   <td style="text-align:left;text-align: left;"> internal </td>
   <td style="text-align:left;text-align: left;"> Lot Area: the area of the lot of each unit </td>
   <td style="text-align:left;text-align: left;"> sqft </td>
   <td style="text-align:right;text-align: left;"> 91681 </td>
   <td style="text-align:right;text-align: left;"> 520 </td>
   <td style="text-align:right;text-align: left;"> 5105 </td>
   <td style="text-align:right;text-align: left;"> 3583 </td>
   <td style="text-align:right;text-align: left;"> 6734 </td>
  </tr>
  <tr>
   <td style="text-align:left;text-align: left;"> pop_den </td>
   <td style="text-align:left;text-align: left;"> socio-economic </td>
   <td style="text-align:left;text-align: left;"> Population Density: the number of population per acre in the census tract </td>
   <td style="text-align:left;text-align: left;"> person / acre </td>
   <td style="text-align:right;text-align: left;"> 76 </td>
   <td style="text-align:right;text-align: left;"> 1 </td>
   <td style="text-align:right;text-align: left;"> 12 </td>
   <td style="text-align:right;text-align: left;"> 7 </td>
   <td style="text-align:right;text-align: left;"> 6734 </td>
  </tr>
  <tr>
   <td style="text-align:left;text-align: left;"> white_share </td>
   <td style="text-align:left;text-align: left;"> socio-economic </td>
   <td style="text-align:left;text-align: left;"> White Population Share:the ratio of white people to total population in the census tract </td>
   <td style="text-align:left;text-align: left;"> % </td>
   <td style="text-align:right;text-align: left;"> 94 </td>
   <td style="text-align:right;text-align: left;"> 8 </td>
   <td style="text-align:right;text-align: left;"> 72 </td>
   <td style="text-align:right;text-align: left;"> 19 </td>
   <td style="text-align:right;text-align: left;"> 6734 </td>
  </tr>
  <tr>
   <td style="text-align:left;text-align: left;"> total_dep </td>
   <td style="text-align:left;text-align: left;"> socio-economic </td>
   <td style="text-align:left;text-align: left;"> Total Dependency Ratio: the ratio of the number of children (0-14 years old) and older persons (65 years or over) to the working-age population (15-64 years old) in the census tract </td>
   <td style="text-align:left;text-align: left;"> % </td>
   <td style="text-align:right;text-align: left;"> 73 </td>
   <td style="text-align:right;text-align: left;"> 3 </td>
   <td style="text-align:right;text-align: left;"> 39 </td>
   <td style="text-align:right;text-align: left;"> 12 </td>
   <td style="text-align:right;text-align: left;"> 6734 </td>
  </tr>
  <tr>
   <td style="text-align:left;text-align: left;"> elder_dep </td>
   <td style="text-align:left;text-align: left;"> socio-economic </td>
   <td style="text-align:left;text-align: left;"> Elderly Dependency Ratio: the ratio of older persons (65 years or over) to the working-age population (15-64 years old) in the census tract </td>
   <td style="text-align:left;text-align: left;"> % </td>
   <td style="text-align:right;text-align: left;"> 49 </td>
   <td style="text-align:right;text-align: left;"> 2 </td>
   <td style="text-align:right;text-align: left;"> 17 </td>
   <td style="text-align:right;text-align: left;"> 7 </td>
   <td style="text-align:right;text-align: left;"> 6734 </td>
  </tr>
  <tr>
   <td style="text-align:left;text-align: left;"> bach_share </td>
   <td style="text-align:left;text-align: left;"> socio-economic </td>
   <td style="text-align:left;text-align: left;"> Bachelor's Degree Rate: the percentage of with a bachelor's degree among adults age 25 and older in the census tract </td>
   <td style="text-align:left;text-align: left;"> % </td>
   <td style="text-align:right;text-align: left;"> 53 </td>
   <td style="text-align:right;text-align: left;"> 10 </td>
   <td style="text-align:right;text-align: left;"> 35 </td>
   <td style="text-align:right;text-align: left;"> 8 </td>
   <td style="text-align:right;text-align: left;"> 6734 </td>
  </tr>
  <tr>
   <td style="text-align:left;text-align: left;"> median_hh_income </td>
   <td style="text-align:left;text-align: left;"> socio-economic </td>
   <td style="text-align:left;text-align: left;"> Median Household Income: median househhold income in the census tract </td>
   <td style="text-align:left;text-align: left;"> $ </td>
   <td style="text-align:right;text-align: left;"> 157292 </td>
   <td style="text-align:right;text-align: left;"> 12269 </td>
   <td style="text-align:right;text-align: left;"> 82292 </td>
   <td style="text-align:right;text-align: left;"> 26471 </td>
   <td style="text-align:right;text-align: left;"> 6734 </td>
  </tr>
  <tr>
   <td style="text-align:left;text-align: left;"> employ_rate </td>
   <td style="text-align:left;text-align: left;"> socio-economic </td>
   <td style="text-align:left;text-align: left;"> Employment Rate: the ratio of the employed to the working age population in the census tract </td>
   <td style="text-align:left;text-align: left;"> % </td>
   <td style="text-align:right;text-align: left;"> 99 </td>
   <td style="text-align:right;text-align: left;"> 81 </td>
   <td style="text-align:right;text-align: left;"> 95 </td>
   <td style="text-align:right;text-align: left;"> 3 </td>
   <td style="text-align:right;text-align: left;"> 6734 </td>
  </tr>
  <tr>
   <td style="text-align:left;text-align: left;"> pover_rate </td>
   <td style="text-align:left;text-align: left;"> socio-economic </td>
   <td style="text-align:left;text-align: left;"> Poverty Rate: the ratio of the number of people (in a given age group) whose income falls below the poverty line to total population in the census tract </td>
   <td style="text-align:left;text-align: left;"> % </td>
   <td style="text-align:right;text-align: left;"> 43 </td>
   <td style="text-align:right;text-align: left;"> 3 </td>
   <td style="text-align:right;text-align: left;"> 11 </td>
   <td style="text-align:right;text-align: left;"> 8 </td>
   <td style="text-align:right;text-align: left;"> 6734 </td>
  </tr>
  <tr>
   <td style="text-align:left;text-align: left;"> sub_dis </td>
   <td style="text-align:left;text-align: left;"> amenities </td>
   <td style="text-align:left;text-align: left;"> Nearest Subway Distance: the distance to the nearest subway station </td>
   <td style="text-align:left;text-align: left;"> feet </td>
   <td style="text-align:right;text-align: left;"> 27441 </td>
   <td style="text-align:right;text-align: left;"> 27 </td>
   <td style="text-align:right;text-align: left;"> 9497 </td>
   <td style="text-align:right;text-align: left;"> 7439 </td>
   <td style="text-align:right;text-align: left;"> 6734 </td>
  </tr>
  <tr>
   <td style="text-align:left;text-align: left;"> parks_area </td>
   <td style="text-align:left;text-align: left;"> amenities </td>
   <td style="text-align:left;text-align: left;"> Parks' Area 500ft: the total area of parks located within a 500-foot radius of each unit </td>
   <td style="text-align:left;text-align: left;"> acre </td>
   <td style="text-align:right;text-align: left;"> 553 </td>
   <td style="text-align:right;text-align: left;"> 0 </td>
   <td style="text-align:right;text-align: left;"> 13 </td>
   <td style="text-align:right;text-align: left;"> 45 </td>
   <td style="text-align:right;text-align: left;"> 6734 </td>
  </tr>
  <tr>
   <td style="text-align:left;text-align: left;"> tree_canopy </td>
   <td style="text-align:left;text-align: left;"> amenities </td>
   <td style="text-align:left;text-align: left;"> Tree Canopy Ratio: the ratio of the area of tree canopy to the total area in the measuring space </td>
   <td style="text-align:left;text-align: left;"> % </td>
   <td style="text-align:right;text-align: left;"> 89 </td>
   <td style="text-align:right;text-align: left;"> 5 </td>
   <td style="text-align:right;text-align: left;"> 29 </td>
   <td style="text-align:right;text-align: left;"> 9 </td>
   <td style="text-align:right;text-align: left;"> 6734 </td>
  </tr>
  <tr>
   <td style="text-align:left;text-align: left;"> med_dis1 </td>
   <td style="text-align:left;text-align: left;"> amenities </td>
   <td style="text-align:left;text-align: left;"> Nearest Medical Distance: the distance to the nearest medical facility </td>
   <td style="text-align:left;text-align: left;"> feet </td>
   <td style="text-align:right;text-align: left;"> 13892 </td>
   <td style="text-align:right;text-align: left;"> 9 </td>
   <td style="text-align:right;text-align: left;"> 4385 </td>
   <td style="text-align:right;text-align: left;"> 2558 </td>
   <td style="text-align:right;text-align: left;"> 6734 </td>
  </tr>
  <tr>
   <td style="text-align:left;text-align: left;"> med_dis2 </td>
   <td style="text-align:left;text-align: left;"> amenities </td>
   <td style="text-align:left;text-align: left;"> Average Distance to 2 Medicals: the average distance to the nearest 2 medical facilities </td>
   <td style="text-align:left;text-align: left;"> feet </td>
   <td style="text-align:right;text-align: left;"> 17742 </td>
   <td style="text-align:right;text-align: left;"> 134 </td>
   <td style="text-align:right;text-align: left;"> 5616 </td>
   <td style="text-align:right;text-align: left;"> 2923 </td>
   <td style="text-align:right;text-align: left;"> 6734 </td>
  </tr>
  <tr>
   <td style="text-align:left;text-align: left;"> med_dis3 </td>
   <td style="text-align:left;text-align: left;"> amenities </td>
   <td style="text-align:left;text-align: left;"> Average Distance to 3 Medicals:the average distance to the nearest 3 medical facilities </td>
   <td style="text-align:left;text-align: left;"> feet </td>
   <td style="text-align:right;text-align: left;"> 20699 </td>
   <td style="text-align:right;text-align: left;"> 355 </td>
   <td style="text-align:right;text-align: left;"> 6726 </td>
   <td style="text-align:right;text-align: left;"> 3691 </td>
   <td style="text-align:right;text-align: left;"> 6734 </td>
  </tr>
  <tr>
   <td style="text-align:left;text-align: left;"> shop_dis1 </td>
   <td style="text-align:left;text-align: left;"> amenities </td>
   <td style="text-align:left;text-align: left;"> Nearest shopping Distance:the distance to the nearest shopping center </td>
   <td style="text-align:left;text-align: left;"> feet </td>
   <td style="text-align:right;text-align: left;"> 31507 </td>
   <td style="text-align:right;text-align: left;"> 99 </td>
   <td style="text-align:right;text-align: left;"> 9019 </td>
   <td style="text-align:right;text-align: left;"> 5459 </td>
   <td style="text-align:right;text-align: left;"> 6734 </td>
  </tr>
  <tr>
   <td style="text-align:left;text-align: left;"> shop_dis2 </td>
   <td style="text-align:left;text-align: left;"> amenities </td>
   <td style="text-align:left;text-align: left;"> Average Distance to 2 Shoppings:the average distance to the nearest 2 shopping center </td>
   <td style="text-align:left;text-align: left;"> feet </td>
   <td style="text-align:right;text-align: left;"> 34370 </td>
   <td style="text-align:right;text-align: left;"> 1505 </td>
   <td style="text-align:right;text-align: left;"> 10931 </td>
   <td style="text-align:right;text-align: left;"> 5149 </td>
   <td style="text-align:right;text-align: left;"> 6734 </td>
  </tr>
  <tr>
   <td style="text-align:left;text-align: left;"> shop_dis3 </td>
   <td style="text-align:left;text-align: left;"> amenities </td>
   <td style="text-align:left;text-align: left;"> Average Distance to 3 Shoppings:the average distance to the nearest 3 shopping center </td>
   <td style="text-align:left;text-align: left;"> feet </td>
   <td style="text-align:right;text-align: left;"> 35466 </td>
   <td style="text-align:right;text-align: left;"> 1781 </td>
   <td style="text-align:right;text-align: left;"> 12166 </td>
   <td style="text-align:right;text-align: left;"> 5218 </td>
   <td style="text-align:right;text-align: left;"> 6734 </td>
  </tr>
  <tr>
   <td style="text-align:left;text-align: left;"> crime_c </td>
   <td style="text-align:left;text-align: left;"> amenities </td>
   <td style="text-align:left;text-align: left;"> No.crime: the number of crimes within a 1/8-mile radius around each unit </td>
   <td style="text-align:left;text-align: left;"> - </td>
   <td style="text-align:right;text-align: left;"> 1044 </td>
   <td style="text-align:right;text-align: left;"> 2 </td>
   <td style="text-align:right;text-align: left;"> 83 </td>
   <td style="text-align:right;text-align: left;"> 72 </td>
   <td style="text-align:right;text-align: left;"> 6734 </td>
  </tr>
  <tr>
   <td style="text-align:left;text-align: left;"> crime_dis1 </td>
   <td style="text-align:left;text-align: left;"> amenities </td>
   <td style="text-align:left;text-align: left;"> Nearest Crime Distance: the distance to the nearest crime </td>
   <td style="text-align:left;text-align: left;"> feet </td>
   <td style="text-align:right;text-align: left;"> 569 </td>
   <td style="text-align:right;text-align: left;"> 4 </td>
   <td style="text-align:right;text-align: left;"> 133 </td>
   <td style="text-align:right;text-align: left;"> 67 </td>
   <td style="text-align:right;text-align: left;"> 6734 </td>
  </tr>
  <tr>
   <td style="text-align:left;text-align: left;"> crime_dis2 </td>
   <td style="text-align:left;text-align: left;"> amenities </td>
   <td style="text-align:left;text-align: left;"> Average Distance to 2 Crimes:the average distance to the nearest 2 crime </td>
   <td style="text-align:left;text-align: left;"> feet </td>
   <td style="text-align:right;text-align: left;"> 583 </td>
   <td style="text-align:right;text-align: left;"> 4 </td>
   <td style="text-align:right;text-align: left;"> 144 </td>
   <td style="text-align:right;text-align: left;"> 69 </td>
   <td style="text-align:right;text-align: left;"> 6734 </td>
  </tr>
  <tr>
   <td style="text-align:left;text-align: left;"> crime_dis3 </td>
   <td style="text-align:left;text-align: left;"> amenities </td>
   <td style="text-align:left;text-align: left;"> Average Distance to 3 Crimes:the average distance to the nearest 3 crime </td>
   <td style="text-align:left;text-align: left;"> feet </td>
   <td style="text-align:right;text-align: left;"> 598 </td>
   <td style="text-align:right;text-align: left;"> 4 </td>
   <td style="text-align:right;text-align: left;"> 153 </td>
   <td style="text-align:right;text-align: left;"> 72 </td>
   <td style="text-align:right;text-align: left;"> 6734 </td>
  </tr>
  <tr>
   <td style="text-align:left;text-align: left;"> crime_dis4 </td>
   <td style="text-align:left;text-align: left;"> amenities </td>
   <td style="text-align:left;text-align: left;"> Average Distance to 4 Crimes:the average distance to the nearest 4 crime </td>
   <td style="text-align:left;text-align: left;"> feet </td>
   <td style="text-align:right;text-align: left;"> 617 </td>
   <td style="text-align:right;text-align: left;"> 4 </td>
   <td style="text-align:right;text-align: left;"> 162 </td>
   <td style="text-align:right;text-align: left;"> 75 </td>
   <td style="text-align:right;text-align: left;"> 6734 </td>
  </tr>
  <tr>
   <td style="text-align:left;text-align: left;"> crime_dis5 </td>
   <td style="text-align:left;text-align: left;"> amenities </td>
   <td style="text-align:left;text-align: left;"> Average Distance to 5 Crimes:the average distance to the nearest 5 crime </td>
   <td style="text-align:left;text-align: left;"> feet </td>
   <td style="text-align:right;text-align: left;"> 688 </td>
   <td style="text-align:right;text-align: left;"> 4 </td>
   <td style="text-align:right;text-align: left;"> 171 </td>
   <td style="text-align:right;text-align: left;"> 78 </td>
   <td style="text-align:right;text-align: left;"> 6734 </td>
  </tr>
</tbody>
</table>

## Categorical Variable

- Make sure all category variables have significant difference between the means of housing price in different groups

```r
# exclude useless variable
house <- house %>%
  select(-med_cat)

#plot all the mean of price on each final categorical variable
house %>% 
  st_drop_geometry()%>%
  select(-GEOID,-NAME)%>%
  select(price,reno_dum, bed_cat, bath_dum,
                floor_cat,water_dum, view_cat, condition_cat, grade_dum, bach_dum, median_hh_dum, employ_dum, pover_dum, sub_cat, sch_cat, park_cat, shop_cat)%>%
  gather(Variable, Value, -price) %>% 
   ggplot(aes(Value, price)) +
     geom_bar(position = "dodge", stat = "summary", fun.y = "mean") +
     facet_wrap(~Variable, ncol = 3, scales = "free") +
     labs(title = "Price as a Function of Categorical Variables", y = "Mean_Price") +
  theme(text = element_text(size = 12), # Default text size for all text
          plot.title = element_text(size = 12, face = "bold"), # Title
          axis.text = element_text(size = 8), # Axis text
          axis.title = element_text(size = 8), # Axis titles
          strip.text = element_text(size = 8)) # Facet label text
```

![](Midterm_files/figure-html/category mean plot-1.png)<!-- -->


- Statistical summary

```r
### reno_dum
house %>%
  st_drop_geometry() %>%
  group_by(reno_dum) %>%
  summarise(count = n()) %>%
  mutate(percent = round(count/sum(count) * 100, digits = 2),
         description = c("haven't been renovated", "have been reivated"))%>%
  kable(caption = "<span style='font-weight: bold; color: black;'>Renovation Status</span>") %>%
  kable_styling(bootstrap_options = c("striped", "hover"),
                full_width = T) %>%
  column_spec(1:4, extra_css = "text-align: left;")
```

<table class="table table-striped table-hover" style="margin-left: auto; margin-right: auto;">
<caption><span style="font-weight: bold; color: black;">Renovation Status</span></caption>
 <thead>
  <tr>
   <th style="text-align:left;"> reno_dum </th>
   <th style="text-align:right;"> count </th>
   <th style="text-align:right;"> percent </th>
   <th style="text-align:left;"> description </th>
  </tr>
 </thead>
<tbody>
  <tr>
   <td style="text-align:left;text-align: left;"> 0 </td>
   <td style="text-align:right;text-align: left;"> 6289 </td>
   <td style="text-align:right;text-align: left;"> 93.39 </td>
   <td style="text-align:left;text-align: left;"> haven't been renovated </td>
  </tr>
  <tr>
   <td style="text-align:left;text-align: left;"> 1 </td>
   <td style="text-align:right;text-align: left;"> 445 </td>
   <td style="text-align:right;text-align: left;"> 6.61 </td>
   <td style="text-align:left;text-align: left;"> have been reivated </td>
  </tr>
</tbody>
</table>


```r
### bed_cat, 0-3,4-7,8+
house %>%
  st_drop_geometry() %>%
  group_by(bed_cat) %>%
  summarise(count = n()) %>%
  mutate(percent = round(count/sum(count) * 100, digits = 2))%>%
  mutate(bed_cat = factor(bed_cat, levels = c("few", "medium", "many"))) %>%
  arrange(bed_cat)%>%
  mutate(description = c("the unit has 0-3 bedrooms", 
                         "the unit has 4-7 bedrooms",
                         "the unit has more than 8 bedrooms"))%>%
  kable(caption = "<span style='font-weight: bold; color: black;'>Category of Bedroom Count</span>") %>%
  kable_styling(bootstrap_options = c("striped", "hover"),
                full_width = T) %>%
    column_spec(1:4, extra_css = "text-align: left;")
```

<table class="table table-striped table-hover" style="margin-left: auto; margin-right: auto;">
<caption><span style="font-weight: bold; color: black;">Category of Bedroom Count</span></caption>
 <thead>
  <tr>
   <th style="text-align:left;"> bed_cat </th>
   <th style="text-align:right;"> count </th>
   <th style="text-align:right;"> percent </th>
   <th style="text-align:left;"> description </th>
  </tr>
 </thead>
<tbody>
  <tr>
   <td style="text-align:left;text-align: left;"> few </td>
   <td style="text-align:right;text-align: left;"> 4695 </td>
   <td style="text-align:right;text-align: left;"> 69.72 </td>
   <td style="text-align:left;text-align: left;"> the unit has 0-3 bedrooms </td>
  </tr>
  <tr>
   <td style="text-align:left;text-align: left;"> medium </td>
   <td style="text-align:right;text-align: left;"> 2025 </td>
   <td style="text-align:right;text-align: left;"> 30.07 </td>
   <td style="text-align:left;text-align: left;"> the unit has 4-7 bedrooms </td>
  </tr>
  <tr>
   <td style="text-align:left;text-align: left;"> many </td>
   <td style="text-align:right;text-align: left;"> 14 </td>
   <td style="text-align:right;text-align: left;"> 0.21 </td>
   <td style="text-align:left;text-align: left;"> the unit has more than 8 bedrooms </td>
  </tr>
</tbody>
</table>


```r
### bath_dum, 0-4, 4+
house %>%
  st_drop_geometry() %>%
  group_by(bath_dum) %>%
  summarise(count = n()) %>%
  mutate(percent = round(count/sum(count) * 100, digits = 2))%>%
  mutate(bath_dum = factor(bath_dum, levels = levels(house$bath_dum))) %>%
  arrange(bath_dum)%>%
  mutate(description = c("the unit has 0-4 bathrooms", 
                         "the unit has more than 4 bedrooms"))%>%
  kable(caption = "<span style='font-weight: bold; color: black;'>Category of Bathroom Count</span>") %>%
  kable_styling(bootstrap_options = c("striped", "hover"),
                full_width = T) %>%
    column_spec(1:4, extra_css = "text-align: left;")
```

<table class="table table-striped table-hover" style="margin-left: auto; margin-right: auto;">
<caption><span style="font-weight: bold; color: black;">Category of Bathroom Count</span></caption>
 <thead>
  <tr>
   <th style="text-align:left;"> bath_dum </th>
   <th style="text-align:right;"> count </th>
   <th style="text-align:right;"> percent </th>
   <th style="text-align:left;"> description </th>
  </tr>
 </thead>
<tbody>
  <tr>
   <td style="text-align:left;text-align: left;"> few </td>
   <td style="text-align:right;text-align: left;"> 6680 </td>
   <td style="text-align:right;text-align: left;"> 99.2 </td>
   <td style="text-align:left;text-align: left;"> the unit has 0-4 bathrooms </td>
  </tr>
  <tr>
   <td style="text-align:left;text-align: left;"> many </td>
   <td style="text-align:right;text-align: left;"> 54 </td>
   <td style="text-align:right;text-align: left;"> 0.8 </td>
   <td style="text-align:left;text-align: left;"> the unit has more than 4 bedrooms </td>
  </tr>
</tbody>
</table>


```r
### floor_cat, 1-2+3, 2.5+3.5
house %>%
  st_drop_geometry() %>%
  group_by(floor_cat) %>%
  summarise(count = n()) %>%
  mutate(percent = round(count/sum(count) * 100, digits = 2))%>%
  mutate(floor_cat = factor(floor_cat, levels = levels(house$floor_cat))) %>%
  arrange(floor_cat)%>%
  mutate(description = c("the unit has 2.5/3.5 floors",
                         "the unit has 1/1.5/2/3 floors"))%>%
  kable(caption = "<span style='font-weight: bold; color: black;'>Category by Floors</span>") %>%
  kable_styling(bootstrap_options = c("striped", "hover"),
                full_width = T) %>%
    column_spec(1:4, extra_css = "text-align: left;")
```

<table class="table table-striped table-hover" style="margin-left: auto; margin-right: auto;">
<caption><span style="font-weight: bold; color: black;">Category by Floors</span></caption>
 <thead>
  <tr>
   <th style="text-align:left;"> floor_cat </th>
   <th style="text-align:right;"> count </th>
   <th style="text-align:right;"> percent </th>
   <th style="text-align:left;"> description </th>
  </tr>
 </thead>
<tbody>
  <tr>
   <td style="text-align:left;text-align: left;"> irregular </td>
   <td style="text-align:right;text-align: left;"> 104 </td>
   <td style="text-align:right;text-align: left;"> 1.54 </td>
   <td style="text-align:left;text-align: left;"> the unit has 2.5/3.5 floors </td>
  </tr>
  <tr>
   <td style="text-align:left;text-align: left;"> regular </td>
   <td style="text-align:right;text-align: left;"> 6630 </td>
   <td style="text-align:right;text-align: left;"> 98.46 </td>
   <td style="text-align:left;text-align: left;"> the unit has 1/1.5/2/3 floors </td>
  </tr>
</tbody>
</table>


```r
### water_dum 
house %>%
  st_drop_geometry() %>%
  group_by(water_dum) %>%
  summarise(count = n()) %>%
  mutate(percent = round(count/sum(count) * 100, digits = 2),
         description = c("the unit isn't located at waterfront area", 
                         "the unit is located at waterfront area"))%>%
  kable(caption = "<span style='font-weight: bold; color: black;'>Waterfront Factor</span>") %>%
  kable_styling(bootstrap_options = c("striped", "hover"),
                full_width = T) %>%
    column_spec(1:4, extra_css = "text-align: left;")
```

<table class="table table-striped table-hover" style="margin-left: auto; margin-right: auto;">
<caption><span style="font-weight: bold; color: black;">Waterfront Factor</span></caption>
 <thead>
  <tr>
   <th style="text-align:left;"> water_dum </th>
   <th style="text-align:right;"> count </th>
   <th style="text-align:right;"> percent </th>
   <th style="text-align:left;"> description </th>
  </tr>
 </thead>
<tbody>
  <tr>
   <td style="text-align:left;text-align: left;"> 0 </td>
   <td style="text-align:right;text-align: left;"> 6705 </td>
   <td style="text-align:right;text-align: left;"> 99.57 </td>
   <td style="text-align:left;text-align: left;"> the unit isn't located at waterfront area </td>
  </tr>
  <tr>
   <td style="text-align:left;text-align: left;"> 1 </td>
   <td style="text-align:right;text-align: left;"> 29 </td>
   <td style="text-align:right;text-align: left;"> 0.43 </td>
   <td style="text-align:left;text-align: left;"> the unit is located at waterfront area </td>
  </tr>
</tbody>
</table>


```r
### view_cat, 0,1,2,3,4
house %>%
  st_drop_geometry() %>%
  group_by(view_cat) %>%
  summarise(count = n()) %>%
  mutate(percent = round(count/sum(count) * 100, digits = 2))%>%
  mutate(view_cat = factor(view_cat, levels = levels(house$view_cat))) %>%
  arrange(view_cat)%>%
  mutate(description = c("the unit has a view scoring 0/4", 
                         "the unit has a view scoring 1/4",
                         "the unit has a view scoring 2/4", 
                         "the unit has a view scoring 3/4",
                         "the unit has a view scoring 4/4" ))%>%
  kable(caption = "<span style='font-weight: bold; color: black;'>View Quality</span>") %>%
  kable_styling(bootstrap_options = c("striped", "hover"),
                full_width = T) %>%
    column_spec(1:4, extra_css = "text-align: left;")
```

<table class="table table-striped table-hover" style="margin-left: auto; margin-right: auto;">
<caption><span style="font-weight: bold; color: black;">View Quality</span></caption>
 <thead>
  <tr>
   <th style="text-align:left;"> view_cat </th>
   <th style="text-align:right;"> count </th>
   <th style="text-align:right;"> percent </th>
   <th style="text-align:left;"> description </th>
  </tr>
 </thead>
<tbody>
  <tr>
   <td style="text-align:left;text-align: left;"> 0 </td>
   <td style="text-align:right;text-align: left;"> 5880 </td>
   <td style="text-align:right;text-align: left;"> 87.32 </td>
   <td style="text-align:left;text-align: left;"> the unit has a view scoring 0/4 </td>
  </tr>
  <tr>
   <td style="text-align:left;text-align: left;"> 1 </td>
   <td style="text-align:right;text-align: left;"> 143 </td>
   <td style="text-align:right;text-align: left;"> 2.12 </td>
   <td style="text-align:left;text-align: left;"> the unit has a view scoring 1/4 </td>
  </tr>
  <tr>
   <td style="text-align:left;text-align: left;"> 2 </td>
   <td style="text-align:right;text-align: left;"> 407 </td>
   <td style="text-align:right;text-align: left;"> 6.04 </td>
   <td style="text-align:left;text-align: left;"> the unit has a view scoring 2/4 </td>
  </tr>
  <tr>
   <td style="text-align:left;text-align: left;"> 3 </td>
   <td style="text-align:right;text-align: left;"> 202 </td>
   <td style="text-align:right;text-align: left;"> 3.00 </td>
   <td style="text-align:left;text-align: left;"> the unit has a view scoring 3/4 </td>
  </tr>
  <tr>
   <td style="text-align:left;text-align: left;"> 4 </td>
   <td style="text-align:right;text-align: left;"> 102 </td>
   <td style="text-align:right;text-align: left;"> 1.51 </td>
   <td style="text-align:left;text-align: left;"> the unit has a view scoring 4/4 </td>
  </tr>
</tbody>
</table>


```r
### condition_cat
house %>%
  st_drop_geometry() %>%
  group_by(condition_cat) %>%
  summarise(count = n()) %>%
  mutate(percent = round(count/sum(count) * 100, digits = 2))%>%
  mutate(condition_cat = factor(condition_cat, levels = levels(house$condition_cat))) %>%
  arrange(condition_cat)%>%
  mutate(description = c("the unit's condition scores 1/5", 
                         "the unit's condition scores 2/5",
                         "the unit's condition scores 3/5",
                         "the unit's condition scores 4/5",
                         "the unit's condition scores 5/5"))%>%
  kable(caption = "<span style='font-weight: bold; color: black;'>Condition Level</span>") %>%
  kable_styling(bootstrap_options = c("striped", "hover"),
                full_width = T) %>%
    column_spec(1:4, extra_css = "text-align: left;")
```

<table class="table table-striped table-hover" style="margin-left: auto; margin-right: auto;">
<caption><span style="font-weight: bold; color: black;">Condition Level</span></caption>
 <thead>
  <tr>
   <th style="text-align:left;"> condition_cat </th>
   <th style="text-align:right;"> count </th>
   <th style="text-align:right;"> percent </th>
   <th style="text-align:left;"> description </th>
  </tr>
 </thead>
<tbody>
  <tr>
   <td style="text-align:left;text-align: left;"> 1 </td>
   <td style="text-align:right;text-align: left;"> 12 </td>
   <td style="text-align:right;text-align: left;"> 0.18 </td>
   <td style="text-align:left;text-align: left;"> the unit's condition scores 1/5 </td>
  </tr>
  <tr>
   <td style="text-align:left;text-align: left;"> 2 </td>
   <td style="text-align:right;text-align: left;"> 57 </td>
   <td style="text-align:right;text-align: left;"> 0.85 </td>
   <td style="text-align:left;text-align: left;"> the unit's condition scores 2/5 </td>
  </tr>
  <tr>
   <td style="text-align:left;text-align: left;"> 3 </td>
   <td style="text-align:right;text-align: left;"> 4324 </td>
   <td style="text-align:right;text-align: left;"> 64.21 </td>
   <td style="text-align:left;text-align: left;"> the unit's condition scores 3/5 </td>
  </tr>
  <tr>
   <td style="text-align:left;text-align: left;"> 4 </td>
   <td style="text-align:right;text-align: left;"> 1566 </td>
   <td style="text-align:right;text-align: left;"> 23.26 </td>
   <td style="text-align:left;text-align: left;"> the unit's condition scores 4/5 </td>
  </tr>
  <tr>
   <td style="text-align:left;text-align: left;"> 5 </td>
   <td style="text-align:right;text-align: left;"> 775 </td>
   <td style="text-align:right;text-align: left;"> 11.51 </td>
   <td style="text-align:left;text-align: left;"> the unit's condition scores 5/5 </td>
  </tr>
</tbody>
</table>


```r
### grade_dum
house %>%
  st_drop_geometry() %>%
  group_by(grade_dum) %>%
  summarise(count = n()) %>%
  mutate(percent = round(count/sum(count) * 100, digits = 2))%>%
  mutate(grade_dum = factor(grade_dum, levels = levels(house$grade_dum))) %>%
  arrange(grade_dum)%>%
  mutate(description = c("the unit's grade is 4-9", 
                         "the unit's grade is 10-13"))%>%
  kable(caption = "<span style='font-weight: bold; color: black;'>Grade Level</span>") %>%
  kable_styling(bootstrap_options = c("striped", "hover"),
                full_width = T) %>%
    column_spec(1:4, extra_css = "text-align: left;")
```

<table class="table table-striped table-hover" style="margin-left: auto; margin-right: auto;">
<caption><span style="font-weight: bold; color: black;">Grade Level</span></caption>
 <thead>
  <tr>
   <th style="text-align:left;"> grade_dum </th>
   <th style="text-align:right;"> count </th>
   <th style="text-align:right;"> percent </th>
   <th style="text-align:left;"> description </th>
  </tr>
 </thead>
<tbody>
  <tr>
   <td style="text-align:left;text-align: left;"> low </td>
   <td style="text-align:right;text-align: left;"> 6476 </td>
   <td style="text-align:right;text-align: left;"> 96.17 </td>
   <td style="text-align:left;text-align: left;"> the unit's grade is 4-9 </td>
  </tr>
  <tr>
   <td style="text-align:left;text-align: left;"> high </td>
   <td style="text-align:right;text-align: left;"> 258 </td>
   <td style="text-align:right;text-align: left;"> 3.83 </td>
   <td style="text-align:left;text-align: left;"> the unit's grade is 10-13 </td>
  </tr>
</tbody>
</table>


```r
### bach_dum
house %>%
  st_drop_geometry() %>%
  group_by(bach_dum) %>%
  summarise(count = n()) %>%
  mutate(percent = round(count/sum(count) * 100, digits = 2))%>%
  mutate(bach_dum = factor(bach_dum, levels = levels(house$bach_dum))) %>%
  arrange(bach_dum)%>%
  mutate(description = c("the unit is in a census tract with a bachelor's degree rate below the Seattle average", 
                         "the unit is in a census tract with a bachelor's degree rate above the Seattle average"))%>%
  kable(caption = "<span style='font-weight: bold; color: black;'>Bachelor's Degree Rate Level</span>") %>%
  kable_styling(bootstrap_options = c("striped", "hover"),
                full_width = T) %>%
    column_spec(1:4, extra_css = "text-align: left;")
```

<table class="table table-striped table-hover" style="margin-left: auto; margin-right: auto;">
<caption><span style="font-weight: bold; color: black;">Bachelor's Degree Rate Level</span></caption>
 <thead>
  <tr>
   <th style="text-align:left;"> bach_dum </th>
   <th style="text-align:right;"> count </th>
   <th style="text-align:right;"> percent </th>
   <th style="text-align:left;"> description </th>
  </tr>
 </thead>
<tbody>
  <tr>
   <td style="text-align:left;text-align: left;"> below </td>
   <td style="text-align:right;text-align: left;"> 1462 </td>
   <td style="text-align:right;text-align: left;"> 21.71 </td>
   <td style="text-align:left;text-align: left;"> the unit is in a census tract with a bachelor's degree rate below the Seattle average </td>
  </tr>
  <tr>
   <td style="text-align:left;text-align: left;"> above </td>
   <td style="text-align:right;text-align: left;"> 5272 </td>
   <td style="text-align:right;text-align: left;"> 78.29 </td>
   <td style="text-align:left;text-align: left;"> the unit is in a census tract with a bachelor's degree rate above the Seattle average </td>
  </tr>
</tbody>
</table>


```r
### median_hh_dum
house %>%
  st_drop_geometry() %>%
  group_by(median_hh_dum) %>%
  summarise(count = n()) %>%
  mutate(percent = round(count/sum(count) * 100, digits = 2))%>%
  mutate(median_hh_dum = factor(median_hh_dum, levels = levels(house$median_hh_dum))) %>%
  arrange(median_hh_dum)%>%
  mutate(description = c("the unit is in a census tract with a median household income below the Seattle average", 
                         "the unit is in a census tract with a median household income above the Seattle average"))%>%
  kable(caption = "<span style='font-weight: bold; color: black;'>Median Household Income Level</span>") %>%
  kable_styling(bootstrap_options = c("striped", "hover"),
                full_width = T) %>%
    column_spec(1:4, extra_css = "text-align: left;")
```

<table class="table table-striped table-hover" style="margin-left: auto; margin-right: auto;">
<caption><span style="font-weight: bold; color: black;">Median Household Income Level</span></caption>
 <thead>
  <tr>
   <th style="text-align:left;"> median_hh_dum </th>
   <th style="text-align:right;"> count </th>
   <th style="text-align:right;"> percent </th>
   <th style="text-align:left;"> description </th>
  </tr>
 </thead>
<tbody>
  <tr>
   <td style="text-align:left;text-align: left;"> below </td>
   <td style="text-align:right;text-align: left;"> 3440 </td>
   <td style="text-align:right;text-align: left;"> 51.08 </td>
   <td style="text-align:left;text-align: left;"> the unit is in a census tract with a median household income below the Seattle average </td>
  </tr>
  <tr>
   <td style="text-align:left;text-align: left;"> above </td>
   <td style="text-align:right;text-align: left;"> 3294 </td>
   <td style="text-align:right;text-align: left;"> 48.92 </td>
   <td style="text-align:left;text-align: left;"> the unit is in a census tract with a median household income above the Seattle average </td>
  </tr>
</tbody>
</table>


```r
### employ_dum
house %>%
  st_drop_geometry() %>%
  group_by(employ_dum) %>%
  summarise(count = n()) %>%
  mutate(percent = round(count/sum(count) * 100, digits = 2))%>%
  mutate(employ_dum = factor(employ_dum, levels = levels(house$employ_dum))) %>%
  arrange(employ_dum)%>%
  mutate(description = c("the unit is in a census tract with a employment rate below the Seattle average", 
                         "the unit is in a census tract with a employment rate above the Seattle average"))%>%
  kable(caption = "<span style='font-weight: bold; color: black;'>Employment Rate Level</span>") %>%
  kable_styling(bootstrap_options = c("striped", "hover"),
                full_width = T) %>%
    column_spec(1:4, extra_css = "text-align: left;")
```

<table class="table table-striped table-hover" style="margin-left: auto; margin-right: auto;">
<caption><span style="font-weight: bold; color: black;">Employment Rate Level</span></caption>
 <thead>
  <tr>
   <th style="text-align:left;"> employ_dum </th>
   <th style="text-align:right;"> count </th>
   <th style="text-align:right;"> percent </th>
   <th style="text-align:left;"> description </th>
  </tr>
 </thead>
<tbody>
  <tr>
   <td style="text-align:left;text-align: left;"> below </td>
   <td style="text-align:right;text-align: left;"> 1517 </td>
   <td style="text-align:right;text-align: left;"> 22.53 </td>
   <td style="text-align:left;text-align: left;"> the unit is in a census tract with a employment rate below the Seattle average </td>
  </tr>
  <tr>
   <td style="text-align:left;text-align: left;"> above </td>
   <td style="text-align:right;text-align: left;"> 5217 </td>
   <td style="text-align:right;text-align: left;"> 77.47 </td>
   <td style="text-align:left;text-align: left;"> the unit is in a census tract with a employment rate above the Seattle average </td>
  </tr>
</tbody>
</table>


```r
### pover_dum
house %>%
  st_drop_geometry() %>%
  group_by(pover_dum) %>%
  summarise(count = n()) %>%
  mutate(percent = round(count/sum(count) * 100, digits = 2))%>%
  mutate(pover_dum = factor(pover_dum, levels = levels(house$pover_dum))) %>%
  arrange(pover_dum)%>%
  mutate(description = c("the unit is in a census tract with a poverty rate below the Seattle average", 
                         "the unit is in a census tract with a poverty rate above the Seattle average"))%>%
  kable(caption = "<span style='font-weight: bold; color: black;'>Poverty Rate Level</span>") %>%
  kable_styling(bootstrap_options = c("striped", "hover"),
                full_width = T) %>%
    column_spec(1:4, extra_css = "text-align: left;")
```

<table class="table table-striped table-hover" style="margin-left: auto; margin-right: auto;">
<caption><span style="font-weight: bold; color: black;">Poverty Rate Level</span></caption>
 <thead>
  <tr>
   <th style="text-align:left;"> pover_dum </th>
   <th style="text-align:right;"> count </th>
   <th style="text-align:right;"> percent </th>
   <th style="text-align:left;"> description </th>
  </tr>
 </thead>
<tbody>
  <tr>
   <td style="text-align:left;text-align: left;"> below </td>
   <td style="text-align:right;text-align: left;"> 4366 </td>
   <td style="text-align:right;text-align: left;"> 64.84 </td>
   <td style="text-align:left;text-align: left;"> the unit is in a census tract with a poverty rate below the Seattle average </td>
  </tr>
  <tr>
   <td style="text-align:left;text-align: left;"> above </td>
   <td style="text-align:right;text-align: left;"> 2368 </td>
   <td style="text-align:right;text-align: left;"> 35.16 </td>
   <td style="text-align:left;text-align: left;"> the unit is in a census tract with a poverty rate above the Seattle average </td>
  </tr>
</tbody>
</table>


```r
### sub_cat
house %>%
  st_drop_geometry() %>%
  group_by(sub_cat) %>%
  summarise(count = n()) %>%
  mutate(percent = round(count/sum(count) * 100, digits = 2))%>%
  mutate(sub_cat = factor(sub_cat, levels = levels(house$sub_cat))) %>%
  arrange(sub_cat)%>%
  mutate(description = c("the unit is within a 0.5-mile radius of a subway station", 
                         "the unit is within a 0.5-mile to 1-mile radius of a subway station",
                         "the unit is beyond a 1-mile radius of a subway station"))%>%
  kable(caption = "<span style='font-weight: bold; color: black;'>Category by Subway Distance</span>") %>%
  kable_styling(bootstrap_options = c("striped", "hover"),
                full_width = T) %>%
    column_spec(1:4, extra_css = "text-align: left;")
```

<table class="table table-striped table-hover" style="margin-left: auto; margin-right: auto;">
<caption><span style="font-weight: bold; color: black;">Category by Subway Distance</span></caption>
 <thead>
  <tr>
   <th style="text-align:left;"> sub_cat </th>
   <th style="text-align:right;"> count </th>
   <th style="text-align:right;"> percent </th>
   <th style="text-align:left;"> description </th>
  </tr>
 </thead>
<tbody>
  <tr>
   <td style="text-align:left;text-align: left;"> within0.5mile </td>
   <td style="text-align:right;text-align: left;"> 1505 </td>
   <td style="text-align:right;text-align: left;"> 22.35 </td>
   <td style="text-align:left;text-align: left;"> the unit is within a 0.5-mile radius of a subway station </td>
  </tr>
  <tr>
   <td style="text-align:left;text-align: left;"> 0.5-1mile </td>
   <td style="text-align:right;text-align: left;"> 1388 </td>
   <td style="text-align:right;text-align: left;"> 20.61 </td>
   <td style="text-align:left;text-align: left;"> the unit is within a 0.5-mile to 1-mile radius of a subway station </td>
  </tr>
  <tr>
   <td style="text-align:left;text-align: left;"> 1+mile </td>
   <td style="text-align:right;text-align: left;"> 3841 </td>
   <td style="text-align:right;text-align: left;"> 57.04 </td>
   <td style="text-align:left;text-align: left;"> the unit is beyond a 1-mile radius of a subway station </td>
  </tr>
</tbody>
</table>


```r
### sch_cat
house %>%
  st_drop_geometry() %>%
  group_by(sch_cat) %>%
  summarise(count = n()) %>%
  mutate(percent = round(count/sum(count) * 100, digits = 2))%>%
  mutate(sch_cat = factor(sch_cat, levels = c("DD1","DD2","DD3","DD4","DD5","DD6","DD7"))) %>%
  arrange(sch_cat)%>%
  mutate(description = c("the unit is in school district one",
                         "the unit is in school district two",
                         "the unit is in school district three",
                         "the unit is in school district four",
                         "the unit is in school district five",
                         "the unit is in school district six",
                         "the unit is in school district seven"))%>%
  kable(caption = "<span style='font-weight: bold; color: black;'>School District</span>") %>%
  kable_styling(bootstrap_options = c("striped", "hover"),
                full_width = T) %>%
    column_spec(1:4, extra_css = "text-align: left;")
```

<table class="table table-striped table-hover" style="margin-left: auto; margin-right: auto;">
<caption><span style="font-weight: bold; color: black;">School District</span></caption>
 <thead>
  <tr>
   <th style="text-align:left;"> sch_cat </th>
   <th style="text-align:right;"> count </th>
   <th style="text-align:right;"> percent </th>
   <th style="text-align:left;"> description </th>
  </tr>
 </thead>
<tbody>
  <tr>
   <td style="text-align:left;text-align: left;"> DD1 </td>
   <td style="text-align:right;text-align: left;"> 1172 </td>
   <td style="text-align:right;text-align: left;"> 17.40 </td>
   <td style="text-align:left;text-align: left;"> the unit is in school district one </td>
  </tr>
  <tr>
   <td style="text-align:left;text-align: left;"> DD2 </td>
   <td style="text-align:right;text-align: left;"> 1310 </td>
   <td style="text-align:right;text-align: left;"> 19.45 </td>
   <td style="text-align:left;text-align: left;"> the unit is in school district two </td>
  </tr>
  <tr>
   <td style="text-align:left;text-align: left;"> DD3 </td>
   <td style="text-align:right;text-align: left;"> 758 </td>
   <td style="text-align:right;text-align: left;"> 11.26 </td>
   <td style="text-align:left;text-align: left;"> the unit is in school district three </td>
  </tr>
  <tr>
   <td style="text-align:left;text-align: left;"> DD4 </td>
   <td style="text-align:right;text-align: left;"> 382 </td>
   <td style="text-align:right;text-align: left;"> 5.67 </td>
   <td style="text-align:left;text-align: left;"> the unit is in school district four </td>
  </tr>
  <tr>
   <td style="text-align:left;text-align: left;"> DD5 </td>
   <td style="text-align:right;text-align: left;"> 808 </td>
   <td style="text-align:right;text-align: left;"> 12.00 </td>
   <td style="text-align:left;text-align: left;"> the unit is in school district five </td>
  </tr>
  <tr>
   <td style="text-align:left;text-align: left;"> DD6 </td>
   <td style="text-align:right;text-align: left;"> 1369 </td>
   <td style="text-align:right;text-align: left;"> 20.33 </td>
   <td style="text-align:left;text-align: left;"> the unit is in school district six </td>
  </tr>
  <tr>
   <td style="text-align:left;text-align: left;"> DD7 </td>
   <td style="text-align:right;text-align: left;"> 935 </td>
   <td style="text-align:right;text-align: left;"> 13.88 </td>
   <td style="text-align:left;text-align: left;"> the unit is in school district seven </td>
  </tr>
</tbody>
</table>


```r
### park_cat
house %>%
  st_drop_geometry() %>%
  group_by(park_cat) %>%
  summarise(count = n()) %>%
  mutate(percent = round(count/sum(count) * 100, digits = 2))%>%
  mutate(park_cat = factor(park_cat, levels = levels(house$park_cat))) %>%
  arrange(park_cat)%>%
  mutate(description = c("the unit is beyond a 500-feet radius of any park", 
                         "the unit is within a 500-feet radius of one park",
                         "the unit is within a 500-feet radius of two parks",
                         "the unit is within a 500-feet radius of three park",
                         "the unit is within a 500-feet radius of four park"))%>%
  kable(caption = "<span style='font-weight: bold; color: black;'>Number of nearby Parks</span>") %>%
  kable_styling(bootstrap_options = c("striped", "hover"),
                full_width = T) %>%
    column_spec(1:4, extra_css = "text-align: left;")
```

<table class="table table-striped table-hover" style="margin-left: auto; margin-right: auto;">
<caption><span style="font-weight: bold; color: black;">Number of nearby Parks</span></caption>
 <thead>
  <tr>
   <th style="text-align:left;"> park_cat </th>
   <th style="text-align:right;"> count </th>
   <th style="text-align:right;"> percent </th>
   <th style="text-align:left;"> description </th>
  </tr>
 </thead>
<tbody>
  <tr>
   <td style="text-align:left;text-align: left;"> 0 </td>
   <td style="text-align:right;text-align: left;"> 4679 </td>
   <td style="text-align:right;text-align: left;"> 69.48 </td>
   <td style="text-align:left;text-align: left;"> the unit is beyond a 500-feet radius of any park </td>
  </tr>
  <tr>
   <td style="text-align:left;text-align: left;"> 1 </td>
   <td style="text-align:right;text-align: left;"> 1651 </td>
   <td style="text-align:right;text-align: left;"> 24.52 </td>
   <td style="text-align:left;text-align: left;"> the unit is within a 500-feet radius of one park </td>
  </tr>
  <tr>
   <td style="text-align:left;text-align: left;"> 2 </td>
   <td style="text-align:right;text-align: left;"> 327 </td>
   <td style="text-align:right;text-align: left;"> 4.86 </td>
   <td style="text-align:left;text-align: left;"> the unit is within a 500-feet radius of two parks </td>
  </tr>
  <tr>
   <td style="text-align:left;text-align: left;"> 3 </td>
   <td style="text-align:right;text-align: left;"> 67 </td>
   <td style="text-align:right;text-align: left;"> 0.99 </td>
   <td style="text-align:left;text-align: left;"> the unit is within a 500-feet radius of three park </td>
  </tr>
  <tr>
   <td style="text-align:left;text-align: left;"> 4 </td>
   <td style="text-align:right;text-align: left;"> 10 </td>
   <td style="text-align:right;text-align: left;"> 0.15 </td>
   <td style="text-align:left;text-align: left;"> the unit is within a 500-feet radius of four park </td>
  </tr>
</tbody>
</table>


```r
### shop_cat
house %>%
  st_drop_geometry() %>%
  group_by(shop_cat) %>%
  summarise(count = n()) %>%
  mutate(percent = round(count/sum(count) * 100, digits = 2))%>%
  mutate(shop_cat = factor(shop_cat, levels = levels(house$shop_cat))) %>%
  arrange(shop_cat)%>%
  mutate(description = c("the unit is within a 0.5-mile radius of a shopping center",
                         "the unit is within a 0.5-mile to 1-mile radius of a shopping center",
                         "the unit is beyond a 1-mile radius of a shopping center"))%>%
  kable(caption = "<span style='font-weight: bold; color: black;'>Category by Shopping Center Distance</span>") %>%
  kable_styling(bootstrap_options = c("striped", "hover"),
                full_width = T) %>%
    column_spec(1:4, extra_css = "text-align: left;")
```

<table class="table table-striped table-hover" style="margin-left: auto; margin-right: auto;">
<caption><span style="font-weight: bold; color: black;">Category by Shopping Center Distance</span></caption>
 <thead>
  <tr>
   <th style="text-align:left;"> shop_cat </th>
   <th style="text-align:right;"> count </th>
   <th style="text-align:right;"> percent </th>
   <th style="text-align:left;"> description </th>
  </tr>
 </thead>
<tbody>
  <tr>
   <td style="text-align:left;text-align: left;"> within0.5mile </td>
   <td style="text-align:right;text-align: left;"> 464 </td>
   <td style="text-align:right;text-align: left;"> 6.89 </td>
   <td style="text-align:left;text-align: left;"> the unit is within a 0.5-mile radius of a shopping center </td>
  </tr>
  <tr>
   <td style="text-align:left;text-align: left;"> 0.5-1mile </td>
   <td style="text-align:right;text-align: left;"> 1245 </td>
   <td style="text-align:right;text-align: left;"> 18.49 </td>
   <td style="text-align:left;text-align: left;"> the unit is within a 0.5-mile to 1-mile radius of a shopping center </td>
  </tr>
  <tr>
   <td style="text-align:left;text-align: left;"> 1+mile </td>
   <td style="text-align:right;text-align: left;"> 5025 </td>
   <td style="text-align:right;text-align: left;"> 74.62 </td>
   <td style="text-align:left;text-align: left;"> the unit is beyond a 1-mile radius of a shopping center </td>
  </tr>
</tbody>
</table>


# Exploratory Data Analysis

## Correlation Matrix


```r
# Select only numeric variables and remove rows with missing values
house_numeric <- house %>%
  st_drop_geometry() %>%  # Remove geometry column if present
  select(-key) %>% # delete unrelative column
  select_if(is.numeric) %>%  # Select only numeric variables
  na.omit()%>%  # Remove rows with missing values
  setNames(c("Price","Year Used", "No.bedrooms","No.bathrooms",
           "Living Area","Lot Area","Population Density","White Population Share",
           "Total Dependency Ratio","Elderly Dependency Ratio","Bachelor's Degree Rate","Median Household Income",
           "Employment Rate","Poverty Rate","Nearest Subway Distance","Parks' Area 500ft","Tree Canopy Ratio",
           "Nearest Medical Distance","Average Distance to 2 Medicals","Average Distance to 3 Medicals",
           "Nearest Shopping Distance","Average Distance to 2 Shoppings","Average Distance to 3 Shoppings",
           "No.crime","Nearest Crime Distance","Average Distance to 2 Crimes","Average Distance to 3 Crimes","Average Distance to 4 Crimes","Average Distance to 5 Crimes"))

# Calculate correlation matrix
correlation_matrix <- cor(house_numeric)

#plot the correlation plot using the corrr library
house_numeric %>% 
  corrr::correlate() %>% 
  autoplot() +
  geom_text(aes(label = round(r,digits=2)),size = 1)
```

![](Midterm_files/figure-html/correlation-1.png)<!-- -->

## Four Home Price Correlation Scatterplots

**Living Area Square Feet**


```r
ggplot(house) +
  geom_point(aes(x = sqft_living, y = price), color = "black", pch = 16, size = 1.6) +
  labs(title = "Seattle House Price vs. Living Area Square Feet",
       x = "Living Sqft",
       y = "House Price") +
  theme_minimal() +
  theme(plot.title = element_text(hjust = 0.5, face = "bold"))
```

![](Midterm_files/figure-html/sqft_living-1.png)<!-- -->

**Median Household Income (Census Tract)**


```r
ggplot(house) +
  geom_point(aes(x = median_hh_income, y = price), color = "black", pch = 16, size = 1.6) +
  labs(title = "Seattle House Price vs. Median Household Income (Census Tract)",
       x = "Income",
       y = "House Price") +
  theme_minimal() +
  theme(plot.title = element_text(hjust = 0.5, face = "bold"))
```

![](Midterm_files/figure-html/median_hh_income-1.png)<!-- -->

**Distance to the Nearest Shopping Center**


```r
ggplot(house) +
  geom_point(aes(x = shop_dis1, y = price), color = "black", pch = 16, size = 1.6) +
  labs(title = "Seattle House Price vs. Distance to the Nearest Shopping Center",
       x = "Distance",
       y = "House Price") +
  theme_minimal() +
  theme(plot.title = element_text(hjust = 0.5, face = "bold"))
```

![](Midterm_files/figure-html/shop_dis1-1.png)<!-- -->

**Crime Count within 1/8 mile of Each House**


```r
ggplot(house) +
  geom_point(aes(x = crime_c, y = price), color = "black", pch = 16, size = 1.6) +
  labs(title = "Seattle House Price vs. Crime Count within 1/8 mile of Each House",
       x = "Crime Count",
       y = "House Price") +
  theme_minimal() +
  theme(plot.title = element_text(hjust = 0.5, face = "bold"))
```

![](Midterm_files/figure-html/crime_c-1.png)<!-- -->

## Map of the Dependent Variable (House Price)


```r
# quantile break and color palette
breaks_quantiles <- classIntervals(house$price, n = 5, style = "quantile")
colors <- brewer.pal(n = 5, name = "YlOrRd")
labels <- paste0(formatC(breaks_quantiles$brks[-length(breaks_quantiles$brks)], format = "f", digits = 0, big.mark = ","), 
                 " - ", 
                 formatC(breaks_quantiles$brks[-1], format = "f", digits = 0, big.mark = ","))

# plot house price
ggplot() +
  geom_sf(data = seattle, fill = "#ECECEC", color = "#2166ac", linewidth = 0.3) +
  geom_sf(data = house,
          aes(color = cut(price, breaks = breaks_quantiles$brks, include.lowest = TRUE)), size = 0.3) +
  scale_color_manual(values = colors,
                    labels = labels,
                    name = "House Price (Quantile)") +
  labs(title = "House Price in Seattle, 2015") +
  theme_void() +
  theme(plot.title = element_text(hjust = 0.5, face = "bold"))
```

![](Midterm_files/figure-html/house_price-1.png)<!-- -->

## Three Maps of Independent Variables

**Lot Square Feet**


```r
# quantile break and color palette
breaks_quantiles <- classIntervals(house$sqft_lot, n = 5, style = "quantile")
colors <- brewer.pal(n = 5, name = "YlOrRd")
labels <- paste0(formatC(breaks_quantiles$brks[-length(breaks_quantiles$brks)], format = "f", digits = 0, big.mark = ","), 
                 " - ", 
                 formatC(breaks_quantiles$brks[-1], format = "f", digits = 0, big.mark = ","))

# plot lot square feet
ggplot() +
  geom_sf(data = seattle, fill = "#ECECEC", color = "#2166ac", linewidth = 0.3) +
  geom_sf(data = house, aes(color = cut(sqft_lot, breaks = breaks_quantiles$brks,
                                        include.lowest = TRUE)), size = 0.3) +
  scale_color_manual(values = colors,
                    labels = labels,
                    name = "Lot Square Feet (Quantile)") +
  labs(title = "Lot Square Feet of Houses in Seattle, 2015") +
  theme_void() +
  theme(plot.title = element_text(hjust = 0.5, face = "bold"))
```

![](Midterm_files/figure-html/sqft_lot-1.png)<!-- -->

**School District**


```r
# color palette
colors <- brewer.pal(n = 7, name = "Set3")
labels <- c("District One", "District Two", "District Three", "District Four",
            "District Five", "District Six", "District Seven")

# plot school district
ggplot() +
  geom_sf(data = sch, fill = "#ECECEC", color = "#2166ac", linewidth = 0.3) +
  geom_sf(data = house, aes(color = sch_cat), size = 0.3) +
  scale_color_manual(values = colors,
                     labels = labels,
                    name = "School District") +
  labs(title = "School Districts in Seattle, 2015") +
  theme_void() +
  theme(plot.title = element_text(hjust = 0.5, face = "bold"))
```

![](Midterm_files/figure-html/sch_cat-1.png)<!-- -->

**White Population Share**


```r
# quantile break and color palette
breaks_quantiles <- classIntervals(acsTractsSeattle.2015$white_share, n = 5, style = "quantile")
colors <- brewer.pal(n = 5, name = "Blues")
labels <- paste0(formatC(breaks_quantiles$brks[-length(breaks_quantiles$brks)], format = "f", digits = 0, big.mark = ","), 
                 " - ", 
                 formatC(breaks_quantiles$brks[-1], format = "f", digits = 0, big.mark = ","))

# plot white population share
ggplot() +
  geom_sf(data = st_intersection(acsTractsSeattle.2015, seattle),
          aes(fill = cut(white_share, breaks = breaks_quantiles$brks, include.lowest = TRUE)),
          color = "#ECECEC") +
  scale_fill_manual(values = colors,
                    labels = labels,
                    name = "White Share (Quantile)") +
  labs(title = "White Population Share of Census Tracts in Seattle, 2015") +
  theme_void() +
  theme(plot.title = element_text(hjust = 0.5, face = "bold"))
```

![](Midterm_files/figure-html/white_pop-1.png)<!-- -->


# Modeling

## Initial Variables Selection

1) Whole data set -> run lm -> select one var in each category 
- Selection rule: for each factor, choose only one variable with the significance

```r
# finalize regression dataset
house <- house %>%
  select(-GEOID, -NAME) %>%
  st_drop_geometry()

# select continuous or categorical variable
lm1 <- lm(price ~ .-key, data = house)
lm1.sum <- summary(lm1)

## bedroom
lm2 <- lm(price ~ .-bed_cat-key, data = house)
lm2.2 <- lm(price ~ .-bedrooms-key, data = house)
stargazer(lm2,lm2.2, type="text")

house <- house %>%
  select(-bed_cat)

## bathroom
lm3 <- lm(price ~ .-bathrooms-key, data = house)
lm3.2 <- lm(price ~ .-bath_dum-key, data = house)
stargazer(lm3,lm3.2, type="text")

house <- house %>%
  select(-bathrooms)

## dependency
lm4 <- lm(price ~ .-total_dep-key, data = house)
lm4.2 <- lm(price ~ .-elder_dep-key, data = house)
stargazer(lm4,lm4.2, type="text")

house <- house %>%
  select(-total_dep)

## bachelor degree
lm5 <- lm(price ~ .-bach_dum-key, data = house)
lm5.2 <- lm(price ~ .-bach_share-key, data = house)
stargazer(lm5,lm5.2, type="text")

house <- house %>%
  select(-bach_dum)

## median household income
lm6 <- lm(price ~ .-median_hh_dum-key, data = house)
lm6.2 <- lm(price ~ .-median_hh_income-key, data = house)
stargazer(lm6,lm6.2, type="text")

house <- house %>%
  select(-median_hh_dum)

## employment
lm7 <- lm(price ~ .-employ_dum-key, data = house)
lm7.2 <- lm(price ~ .-employ_rate-key, data = house)
stargazer(lm7,lm7.2, type="text")

house <- house %>%
  select(-employ_dum)

## poverty
lm8 <- lm(price ~ .-pover_dum-key, data = house)
lm8.2 <- lm(price ~ .-pover_rate-key, data = house)
stargazer(lm8,lm8.2, type="text")

house <- house %>%
  select(-pover_dum)

## subway station
lm9 <- lm(price ~ .-sub_cat-key, data = house)
lm9.2 <- lm(price ~ .-sub_dis-key, data = house)
stargazer(lm9,lm9.2, type="text")

house <- house %>%
  select(-sub_cat)

## park
lm10 <- lm(price ~ .-parks_area-key, data = house)
lm10.2 <- lm(price ~ .-park_cat-key, data = house)
stargazer(lm10,lm10.2, type="text")

house <- house %>%
  select(-parks_area)

## medical facility
lm11 <- lm(price ~ .-med_dis2-med_dis3-key, data = house)
lm11.2 <- lm(price ~ .-med_dis1-med_dis2-key, data = house)
lm11.3 <- lm(price ~ .-med_dis1-med_dis3-key, data = house)
stargazer(lm11,lm11.2,lm11.3, type="text")

house <- house %>%
  select(-med_dis2, -med_dis3)

## shopping center
lm12 <- lm(price ~ .-shop_dis2-shop_dis3-shop_cat-key, data = house)
lm12.2 <- lm(price ~ .-shop_dis1-shop_dis2-shop_cat-key, data = house)
lm12.3 <- lm(price ~ .-shop_dis1-shop_dis3-shop_cat-key, data = house)
lm12.4 <- lm(price ~ .-shop_dis1-shop_dis2-shop_dis3-key, data = house)
stargazer(lm12,lm12.2,lm12.3,lm12.4, type="text")

house <- house %>%
  select(-shop_dis2, -shop_dis3,-shop_cat)

## crime
lm13 <- lm(price ~ .-crime_dis1-crime_dis2-crime_dis3-crime_dis4-crime_dis5-key, data = house)
lm13.2 <- lm(price ~ .-crime_dis2-crime_dis3-crime_dis4-crime_dis5-crime_c-key, data = house)
lm13.3 <- lm(price ~ .-crime_dis1-crime_dis3-crime_dis4-crime_dis5-crime_c-key, data = house)
lm13.4 <- lm(price ~ .-crime_dis2-crime_dis1-crime_dis4-crime_dis5-crime_c-key, data = house)
lm13.5 <- lm(price ~ .-crime_dis2-crime_dis3-crime_dis1-crime_dis5-crime_c-key, data = house)
lm13.6 <- lm(price ~ .-crime_dis2-crime_dis3-crime_dis4-crime_dis1-crime_c-key, data = house)
stargazer(lm13,lm13.2,lm13.3,lm13.4,lm13.5,lm13.6, type="text")

house <- house %>%
  select(-crime_dis1,-crime_dis2, -crime_dis3, -crime_dis4, -crime_dis5)

## add fixed effect
house_lm <- house %>%
  left_join(hh%>%select(key), by = "key") %>%
  st_as_sf() %>%
  st_join(., neigh_large) %>%
  st_join(., neigh_small) %>%
  st_join(., neigh_tract) %>%
  st_drop_geometry() %>%
  select(-key)
str(house_lm)
```

## Split Training and Testing Datasets

2) Split data set -> run lm on training data set

```r
# split data 0.7/0.3
set.seed(1)
inTrain <- createDataPartition(
              y = paste(house_lm$reno_dum, house_lm$bath_dum, house_lm$floor_cat,
                        house_lm$water_dum, house_lm$view_cat, house_lm$condition_cat,
                        house_lm$grade_dum, house_lm$sch_cat, house_lm$park_cat, house_lm$L_NAME), 
              p = .70, list = FALSE)  # create a vector for the training set (70%)

# subset the dataset to create the training set
seattle.train.lm <- house_lm[inTrain,] # training set

# subset the dataset to create the testing set
seattle.test.lm <- house_lm[-inTrain,] # testing set

rbind(seattle.train.lm%>%mutate(dataset = "training"),
      seattle.test.lm %>% mutate(dataset = "testing"))%>%
  group_by(dataset)%>%
  summarise(count = n())%>%
  mutate(percent = round(count/sum(count) * 100, digits = 2))%>%
  kable() %>%
  kable_styling(bootstrap_options = c("striped", "hover"),
                full_width = T) %>%
    column_spec(1:3, extra_css = "text-align: left;")
```

<table class="table table-striped table-hover" style="margin-left: auto; margin-right: auto;">
 <thead>
  <tr>
   <th style="text-align:left;"> dataset </th>
   <th style="text-align:right;"> count </th>
   <th style="text-align:right;"> percent </th>
  </tr>
 </thead>
<tbody>
  <tr>
   <td style="text-align:left;text-align: left;"> testing </td>
   <td style="text-align:right;text-align: left;"> 1663 </td>
   <td style="text-align:right;text-align: left;"> 24.7 </td>
  </tr>
  <tr>
   <td style="text-align:left;text-align: left;"> training </td>
   <td style="text-align:right;text-align: left;"> 5071 </td>
   <td style="text-align:right;text-align: left;"> 75.3 </td>
  </tr>
</tbody>
</table>

## Model Diagnostics

### Address Multicollinearity

3) Run vif and cor -> deal with multicollinearity  
-> decision rule: biggest vif -> run cor on continuous variables
4) Exclude insignificance until all varibles significant at P < 0.1

```r
# ignore neighborhood effect first
seattle.train <- seattle.train.lm %>%
  select(-L_NAME, -S_NAME, -T_NAME)

seattle.test <- seattle.test.lm %>%
  select(-L_NAME, -S_NAME, -T_NAME)

# build regression model on training dataset
lm14 <- lm(price ~ ., data = seattle.train)
summary(lm14)
vif(lm14) #sub_dis = 2.664951

## cor on continuous variable
cor.multi <- seattle.train %>% 
  select_if(is.numeric)%>%
  corrr::correlate() %>% 
  autoplot() +
  geom_text(aes(label = round(r,digits=2)),size = 1)
## result:
## sub_dis have low correlation with other numeric variables
## high correlation: white share & bachelor degree
## high correlation: white share & poverty rate

## lm on sub_dis
lm.sub <- lm(sub_dis ~ ., data = seattle.train)
summary(lm.sub) # R^2 = 0.8597 -> highly correlated with other category variables
vif(lm.sub)

lm15 <- lm(price ~ .-sub_dis, data = seattle.train)
stargazer(lm14,lm15, type = "text") #R^2: 0.805->0.803

vif(lm15) #white_share = 2.452959

seattle.train <- seattle.train %>%
  select(-sub_dis)

## high correlation: white share & bachelor degree
lm16 <- lm(price ~ .-bach_share, data = seattle.train)
lm16.2 <- lm(price ~ .-white_share, data = seattle.train)
stargazer(lm16, lm16.2, type="text")
vif(lm16)

seattle.train <- seattle.train %>%
  select(-bach_share)

## high correlation: white share & poverty rate
lm17 <- lm(price ~ .-pover_rate, data = seattle.train)
lm17.2 <- lm(price ~ .-white_share, data = seattle.train)
stargazer(lm17, lm17.2, type="text")

vif(lm17)

seattle.train <- seattle.train %>%
  select(-pover_rate)

## insignificance: tree canopy
lm18 <- lm(price ~ .-tree_canopy, data = seattle.train)
summary(lm18)
vif(lm18)

seattle.train <- seattle.train %>%
  select(-tree_canopy)
```

### Trade Off on Accuracy and Generalizability

- All variables are already significant at p=0.1
- Delete var. one by one to balance the accuracy and generalizability: adjusted R^2 and AbsError, APE

```r
# make predictions on the test set and evaluate model performance
lm19 <- lm(price ~ ., data = seattle.train)
summary(lm19) 
# adjusted R-squared = 0.8013

## first regression model from last step
seattle.test1 <-
  seattle.test %>%
  mutate(regression = "baseline regression", # add a column indicating the type of regression model used
         price.predict = predict(lm19, seattle.test), # predict house prices using the trained regression model
         # calculate the difference between predicted and actual house prices
         price.error = price.predict - price,
         # calculate the absolute difference between predicted and actual house prices
         price.AbsError = abs(price.predict - price),
         # calculate the absolute percentage error
         price.APE = round((abs(price.predict - price)) / price * 100, digits = 2))

mean(seattle.test1$price.AbsError) # 93754.08
mean(seattle.test1$price.APE) # 18.97%

## "year_used" p = 0.051837 -> remove
lm20 <- lm(price ~ .-year_used, data = seattle.train)

seattle.test2 <-
  seattle.test %>%
  mutate(regression = "baseline regression", # add a column indicating the type of regression model used
         price.predict = predict(lm20, seattle.test), # predict house prices using the trained regression model
         # calculate the difference between predicted and actual house prices
         price.error = price.predict - price,
         # calculate the absolute difference between predicted and actual house prices
         price.AbsError = abs(price.predict - price),
         # calculate the absolute percentage error
         price.APE = round((abs(price.predict - price)) / price * 100, digits = 2))

summary(lm20) # adjusted R-squared = 0.8012 (-0.0001)
mean(seattle.test2$price.AbsError) # 93637.18 (-116.9)
mean(seattle.test2$price.APE) # 18.93% (-0.04%)
#-> accuracy worse but generalizability better

seattle.train <- seattle.train %>%
  select(-year_used)

## "crime_c" p = 0.002917 -> remove
lm21 <- lm(price ~ .-crime_c, data = seattle.train)

seattle.test3 <-
  seattle.test %>%
  mutate(regression = "baseline regression", # add a column indicating the type of regression model used
         price.predict = predict(lm21, seattle.test), # predict house prices using the trained regression model
         # calculate the difference between predicted and actual house prices
         price.error = price.predict - price,
         # calculate the absolute difference between predicted and actual house prices
         price.AbsError = abs(price.predict - price),
         # calculate the absolute percentage error
         price.APE = round((abs(price.predict - price)) / price * 100, digits = 2))

summary(lm21) # adjusted R-squared = 0.8009 (-0.003)
mean(seattle.test3$price.AbsError) # 93507.13 (-130.05)
mean(seattle.test3$price.APE) # 18.89% (-0.04)
#-> accuracy worse but generalizability better

seattle.train <- seattle.train %>%
  select(-crime_c)

## "employ_rate" -> remove
lm22 <- lm(price ~ .-employ_rate, data = seattle.train)

seattle.test4 <-
  seattle.test %>%
  mutate(regression = "baseline regression", # add a column indicating the type of regression model used
         price.predict = predict(lm22, seattle.test), # predict house prices using the trained regression model
         # calculate the difference between predicted and actual house prices
         price.error = price.predict - price,
         # calculate the absolute difference between predicted and actual house prices
         price.AbsError = abs(price.predict - price),
         # calculate the absolute percentage error
         price.APE = round((abs(price.predict - price)) / price * 100, digits = 2))

summary(lm22) # adjusted R-squared = 0.8004 (-0.0005)
mean(seattle.test4$price.AbsError) # 93339.94 (-167.19)
mean(seattle.test4$price.APE) # 18.85% (-0.04%)
#-> accuracy worse but generalizability better

seattle.train <- seattle.train %>%
  select(-employ_rate)

## "med_dis1" -> remove
lm23 <- lm(price ~ .-med_dis1, data = seattle.train)

seattle.test5 <-
  seattle.test %>%
  mutate(regression = "baseline regression", # add a column indicating the type of regression model used
         price.predict = predict(lm23, seattle.test), # predict house prices using the trained regression model
         # calculate the difference between predicted and actual house prices
         price.error = price.predict - price,
         # calculate the absolute difference between predicted and actual house prices
         price.AbsError = abs(price.predict - price),
         # calculate the absolute percentage error
         price.APE = round((abs(price.predict - price)) / price * 100, digits = 2))

summary(lm23) # adjusted R-squared = 0.7995 (-0.0009)
mean(seattle.test5$price.AbsError) # 93014.7 (-325.24)
mean(seattle.test5$price.APE) # 18.72% (-0.13%)
#-> accuracy worse but generalizability better

seattle.train <- seattle.train %>%
  select(-med_dis1)

## "elder_dep" -> remove
lm24 <- lm(price ~ .-elder_dep, data = seattle.train)

seattle.test6 <-
  seattle.test %>%
  mutate(regression = "baseline regression", # add a column indicating the type of regression model used
         price.predict = predict(lm24, seattle.test), # predict house prices using the trained regression model
         # calculate the difference between predicted and actual house prices
         price.error = price.predict - price,
         # calculate the absolute difference between predicted and actual house prices
         price.AbsError = abs(price.predict - price),
         # calculate the absolute percentage error
         price.APE = round((abs(price.predict - price)) / price * 100, digits = 2))

summary(lm24) # adjusted R-squared = 0.7962 (-0.0033)
mean(seattle.test6$price.AbsError) # 92474.93 (-539.77)
mean(seattle.test6$price.APE) # 18.50% (-0.22%)
#-> accuracy worse but generalizability better

seattle.train <- seattle.train %>%
  select(-elder_dep)

## "pop_den" -> remove
lm25 <- lm(price ~ .-pop_den, data = seattle.train)

seattle.test7 <-
  seattle.test %>%
  mutate(regression = "baseline regression", # add a column indicating the type of regression model used
         price.predict = predict(lm25, seattle.test), # predict house prices using the trained regression model
         # calculate the difference between predicted and actual house prices
         price.error = price.predict - price,
         # calculate the absolute difference between predicted and actual house prices
         price.AbsError = abs(price.predict - price),
         # calculate the absolute percentage error
         price.APE = round((abs(price.predict - price)) / price * 100, digits = 2))

summary(lm25) # adjusted R-squared = 0.7963 (-0.0001)
mean(seattle.test7$price.AbsError) # 92440.65 (-34.28)
mean(seattle.test7$price.APE) # 18.49% (-0.01)
#-> accuracy worse but generalizability better

seattle.train <- seattle.train %>%
  select(-pop_den)

## "reno_dum","bedrooms","bath_dum", "sqft_living", "sqft_lot", "floor_cat", "water_dum", "view_cat", "condition_cat", "grade_dum", "white_share", "median_hh_income", "sch_cat", "park_cat", "shop_dis1" -> keep

#-> both worse
```

### Spatial Autocorrelation


```r
# spatial lag of price
house.sf <- hh %>%
  select(geometry,key)%>%
  right_join(house, by = "key")

coords <- st_coordinates(house.sf)
neighborList <- knn2nb(knearneigh(coords, 5))
spatialWeights <- nb2listw(neighborList, style="W")

house.sf %>%
  mutate(lagPrice = lag.listw(spatialWeights, price))%>%
  ggplot()+
  geom_point(aes(x = lagPrice, y = price), color = "black", pch = 16, size = 1.6)+
  stat_smooth(aes(lagPrice, price), 
             method = "lm", se = FALSE, size = 1, color="#b2182b")+
  labs(title="Price as a Function of the Spatial Lag of Price") +
  theme_minimal() +
  theme(plot.title = element_text(hjust = 0.5, face = "bold"))
```

![](Midterm_files/figure-html/spatial autocorrelation-1.png)<!-- -->

```r
# spatial lag of price error
house.test.sf <- hh %>%
  select(geometry,key)%>%
  right_join(house[-inTrain,], by = "key")

coords.test <- sf::st_coordinates(house.test.sf)
neighborList.test <- knn2nb(knearneigh(coords.test, 5))
spatialWeights.test <- nb2listw(neighborList.test, style="W")

seattle.test7 %>% 
  mutate(lagPriceError = lag.listw(spatialWeights.test, price.error)) %>%
  ggplot()+
  geom_point(aes(x = lagPriceError, y = price.error), color = "black", pch = 16, size = 1.6)+
  stat_smooth(aes(lagPriceError, price.error), 
             method = "lm", se = FALSE, size = 1, color="#b2182b")+
  labs(title="Error as a Function of the Spatial Lag of Error")+
  theme_minimal() +
  theme(plot.title = element_text(hjust = 0.5, face = "bold"))
```

![](Midterm_files/figure-html/spatial autocorrelation-2.png)<!-- -->

```r
# moran's I test
moranTest <- moran.mc(seattle.test7$price.error, 
                      spatialWeights.test, nsim = 999)
#p-value = 0.001, pattern is slight 0.09

ggplot(as.data.frame(moranTest$res[c(1:999)]), aes(moranTest$res[c(1:999)])) +
  geom_histogram(binwidth = 0.01) +
  geom_vline(aes(xintercept = moranTest$statistic), colour = "#FA7800",size=1) +
  scale_x_continuous(limits = c(-1, 1)) +
  labs(title="Observed and permuted Moran's I",
       subtitle= "Observed Moran's I in orange",
       x="Moran's I",
       y="Count") +
  theme_minimal() +
  theme(plot.title = element_text(hjust = 0.5, face = "bold"),
        plot.subtitle = element_text(hjust = 0.5))
```

![](Midterm_files/figure-html/spatial autocorrelation-3.png)<!-- -->

```r
moranTest.stats <- moranTest$statistic # 0.2198671
```

### Cross Validation


```r
# without fixed effect
set.seed(1)
fitControl <- trainControl(method = "cv", number = 100)

## "reno_dum","bedrooms","bath_dum", "sqft_living", "sqft_lot", "floor_cat", "water_dum", "view_cat", "condition_cat", "grade_dum", "white_share", "median_hh_income", "sch_cat", "park_cat", "shop_dis1" 

seattle.cv <- 
  train(price ~ ., data = house_lm %>% select(price, reno_dum, bedrooms, bath_dum,
                                              sqft_living, sqft_lot, floor_cat,
                                              water_dum, view_cat, condition_cat,
                                              grade_dum, white_share, median_hh_income, sch_cat, park_cat, shop_dis1),
     method = "lm", trControl = fitControl, na.action = na.pass)

### Large: L_NAME
seattle.neighL.cv <- 
  train(price ~ ., data = house_lm %>% select(price, reno_dum, bedrooms, bath_dum,
                                              sqft_living, sqft_lot, floor_cat,
                                              water_dum, view_cat, condition_cat,
                                              grade_dum, white_share, median_hh_income, sch_cat, park_cat, shop_dis1, L_NAME),
     method = "lm", trControl = fitControl, na.action = na.pass)


### Small: S_NAME
seattle.neighS.cv <- 
  train(price ~ ., data = house_lm %>% select(price, reno_dum, bedrooms, bath_dum,
                                              sqft_living, sqft_lot, floor_cat,
                                              water_dum, view_cat, condition_cat,
                                              grade_dum, white_share, median_hh_income, sch_cat, park_cat, shop_dis1, S_NAME),
     method = "lm", trControl = fitControl, na.action = na.pass)


### tract: T_NAME
seattle.neighT.cv <- 
  train(price ~ ., data = house_lm %>% select(price, reno_dum, bedrooms, bath_dum,
                                              sqft_living, sqft_lot, floor_cat,
                                              water_dum, view_cat, condition_cat,
                                              grade_dum, white_share, median_hh_income, sch_cat, park_cat, shop_dis1, T_NAME),
     method = "lm", trControl = fitControl, na.action = na.pass)


#-> census tract as the fixed effect improves the model most

data.frame(rbind(seattle.cv$results,seattle.neighL.cv$results, seattle.neighS.cv$results,seattle.neighT.cv$results))%>%
  mutate(intercept = c("baseline","fix_effect_large_district", "fix_effect_small_neighborhood", "fix_effect_census_tract"))%>%
  rename(model = "intercept")%>%
  select(-RMSESD, -RsquaredSD, -MAESD) %>%
  kable()%>%
  kable_styling(bootstrap_options = c("striped", "hover"),
                full_width = T) %>%
  column_spec(1:4, extra_css = "text-align: left;")
```

<table class="table table-striped table-hover" style="margin-left: auto; margin-right: auto;">
 <thead>
  <tr>
   <th style="text-align:left;"> model </th>
   <th style="text-align:right;"> RMSE </th>
   <th style="text-align:right;"> Rsquared </th>
   <th style="text-align:right;"> MAE </th>
  </tr>
 </thead>
<tbody>
  <tr>
   <td style="text-align:left;text-align: left;"> baseline </td>
   <td style="text-align:right;text-align: left;"> 156295.9 </td>
   <td style="text-align:right;text-align: left;"> 0.7848814 </td>
   <td style="text-align:right;text-align: left;"> 105016.78 </td>
  </tr>
  <tr>
   <td style="text-align:left;text-align: left;"> fix_effect_large_district </td>
   <td style="text-align:right;text-align: left;"> 153985.8 </td>
   <td style="text-align:right;text-align: left;"> 0.7890730 </td>
   <td style="text-align:right;text-align: left;"> 102215.99 </td>
  </tr>
  <tr>
   <td style="text-align:left;text-align: left;"> fix_effect_small_neighborhood </td>
   <td style="text-align:right;text-align: left;"> 143735.5 </td>
   <td style="text-align:right;text-align: left;"> 0.8130965 </td>
   <td style="text-align:right;text-align: left;"> 95447.55 </td>
  </tr>
  <tr>
   <td style="text-align:left;text-align: left;"> fix_effect_census_tract </td>
   <td style="text-align:right;text-align: left;"> 142551.1 </td>
   <td style="text-align:right;text-align: left;"> 0.8194192 </td>
   <td style="text-align:right;text-align: left;"> 94686.55 </td>
  </tr>
</tbody>
</table>

## Final Model

**Dependent variable:**  
House Price

**Independent variables:**  
1. reno_dum (renovation status)  
2. bedrooms (number of bedrooms)  
3. bath_dum (category of bathroom count)  
4. sqft_living (living area square feet)  
5. sqft_lot (lot square feet)  
6. floor_cat (category by floors)  
7. water_dum (waterfront factor)  
8. view_cat (view quality)  
9. condition_cat (condition level)  
10. grade_dum (grade level)  
11. white_share (white population share)  
12. median_hh_income (median household income)  
13. sch_cat (school districts)  
14. park_cat (number of nearby parks)  
15. shop_dis1 (distance to the nearest shopping center)  
16. T_NAME (census tracts)


```r
seattle.train.lm <- seattle.train.lm %>%
  mutate(T_NAME = as.factor(T_NAME))

par(mfrow = c(2, 2))

# with fixed effect
lm.final.nhood <- lm(price ~ reno_dum + bedrooms + bath_dum + sqft_living + sqft_lot +
                 floor_cat + water_dum + view_cat + condition_cat + grade_dum +
                 white_share + median_hh_income + sch_cat + park_cat + shop_dis1 +
                 T_NAME, data = seattle.train.lm)
plot(lm.final.nhood)
mtext("Diagnostic Plots for Linear Model with Fixed Effect", side = 3, line = -2, outer = TRUE, cex = 1.1, font = 2)
```

![](Midterm_files/figure-html/final model-1.png)<!-- -->

## Visualize Model Results

### Predicted Prices vs. Observed Prices

![](Midterm_files/figure-html/predict_plot-1.png)<!-- -->

### Map of Residuals

![](Midterm_files/figure-html/residual_map-1.png)<!-- -->![](Midterm_files/figure-html/residual_map-2.png)<!-- -->


# Conclusion
balabalbala


# Appendix: Variable Selection Reference

## Socio-economic Characteristics

### Age Structure

- variable
1)  total dependency ratio
2)  elderly dependency ratio
- reference: "total dependency ratio and elderly dependency ratio are on an inverse relationship towards ordinary residence price." https://www.scirp.org/journal/paperinformation?paperid=74919

### Education Level

- reference: "The findings show that higher education does have a positive relationship to the house prices in Sweden."
https://www.diva-portal.org/smash/get/diva2:1346009/FULLTEXT01.pdf

## Amenities Characteristics

Amenities impact: https://www.rentseattle.com/blog/how-local-amenities-help-seattle-investors-find-good-properties  

### Subway Station

- variable  
1) distance to the nearest station
2) categories based on distance (0-0.5/0.5-1/1+ mile) (0.5mile = 2640ft)  

- reference: "houses within a quarter mile to a half mile of a Metro station sold for 7.5% more, and houses within a half mile to a mile sold for 3.9% more." https://www.freddiemac.com/research/insight/20191002-metro-station-impact#:~:text=Similarly%2C%20houses%20within%20a%20quarter,mile%20sold%20for%203.9%25%20more.

- source: https://gis-kingcounty.opendata.arcgis.com/datasets/7fb1b64925db450e8f024940f697823e_390/explore?location=47.584121%2C-122.115870%2C10.45

### School District

- variable: the name of the school district

- reference: "the price differentials for similar homes — same square footage, number of bedrooms and baths — that are located near each other but served by different school districts can range from tens of thousands to hundreds of thousands of dollars." https://www.seattletimes.com/business/how-much-do-good-schools-boost-home-prices/  

- source: https://gis-kingcounty.opendata.arcgis.com/datasets/94eb521c71f2401586c6ce6a34d68166_406/explore?location=47.672575%2C-122.604064%2C18.86

### Parks

- variable: 
1) area of parks within 500 feet radius
2) count of parks within 500 feet radius

- reference: "Most community-sized parks (~40 acres) had a substantial impact on home prices up to a distance of 500 to 600 feet. While the influence of larger parks extended out to 2,000 feet, beyond 500 feet the influence was relatively small." https://www.naturequant.com/blog/Impact-of-Parks-on-Property-Value/  

- source: https://gis-kingcounty.opendata.arcgis.com/datasets/a0c94c33228146c5ad95a1dff3b6963d_228/explore?location=47.557674%2C-122.213839%2C11.45

### Tree Canopy

- variable:
1) percent of existing tree canopy (2016, hexagons)

- reference: "a city facing major development pressure — trees were associated with an increase in single family home values"
https://www.vibrantcitieslab.com/resources/urban-trees-increase-home-values/

- source: https://data-seattlecitygis.opendata.arcgis.com/datasets/SeattleCityGIS::seattle-tree-canopy-2016-2021-50-acre-hexagons/explore?layer=1&location=47.580733%2C-122.309741%2C11.26

### Medical Facilities

- variable: 
1) average distance to the nearest 1/2/3 medical facilities
2) categories based on distance (0-0.5/0.5-1/1+ mile)

- reference: "hospitals would only be highly evaluated in a ‘close-but-not-too-close’ geographic location" https://www.researchgate.net/publication/282942128_The_non-linearity_of_hospitals'_proximity_on_property_prices_experiences_from_Taipei_Taiwan

- source: https://gis-kingcounty.opendata.arcgis.com/datasets/1b7f0fb5179a400f91a35c0b6bfd77c9_733/explore

### Commercial

- variable:  
1) average distance to the nearest 1/2/3 shops
2) categories based on distance (0-0.5/0.5-1/1+ mile)

- reference: 
1) "an inverse relationship between the housing price and its distance from the shopping mall" https://www.diva-portal.org/smash/get/diva2:1450713/FULLTEXT01.pdf  
2) "Notwithstanding the negative externalities of shopping centres, residential
properties within 100-metre radius of shopping centres command a higher premium
than those farther away although the price-distance relationship is not monotonic
while the proximity factor varies from housing estate to housing estate." https://www.prres.org/uploads/746/1752/Addae-Dapaah_Shopping_Centres_Proximate_Residential_Properties.pdf

- source:https://gis-kingcounty.opendata.arcgis.com/datasets/4fdb4709874b46cf8dbb284182ca0094_383/explore?showTable=true  
Select type by "CODE": https://www.arcgis.com/sharing/rest/content/items/4fdb4709874b46cf8dbb284182ca0094/info/metadata/metadata.xml?format=default&output=html
- 690 Shopping Centers

### Crime

- variable:
1) count of crimes within a 1/8 mi
2) average distance to the nearest 1/2/3/4/5 crimes (robbery and  assault only)

- reference:"only robbery and aggravated assault crimes (per acre) exert a meaningful influence upon neighborhood housing values." https://sciencedirect.com/science/article/pii/S0166046210000086#aep-abstract-id3
"The overall effect on house prices of crime (measured as crime rates) is relatively small, but if its impact is measured by distance to a crime hot spot, the effect is non-negligible." https://www.researchgate.net/publication/335773241_Do_crime_hot_spots_affect_housing_prices_Do_crime_hot_spots_affect_housing_prices

- source:https://data.seattle.gov/Public-Safety/SPD-Crime-Data-2008-Present/tazs-3rd5/about_data
