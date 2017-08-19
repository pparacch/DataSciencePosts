# Building Data Visualization Tools (Part 4)
Pier Lorenzo Paracchini, `r format(Sys.time(), '%d.%m.%Y')`  



The content of this blog is based on examples/ notes/ experiments related to the material presented in the "Building Data Visualization Tools" module of the "[Mastering Software Development in R](https://www.coursera.org/specializations/r)" Specialization (Coursera) created by __Johns Hopkins University__ [1].

### Required Packages

* `ggplot2`, a system for 'declaratively' creating graphics, based on "The Grammar of Graphics".
* `gridExtra`, provides a number of user-level functions to work with "grid" graphics.
* `dplyr`, a tool for working with data frame like objects, both in memory and out of memory.
* `viridis`, the viridis color palette.


```r
# If necessary to install a package run
# install.packages("packageName")

# Load packages
library(ggplot2)
library(gridExtra)
library(dplyr)
library(viridis)
```

### Data

The `ggplot2` package includes some datasets with geographic information. The `ggplot2::map_data()` function allows to get map data from the `maps` package (use `?map_data` form more information). 

Specifically the `italy` dataset [2] is used for some of the examples below. Please note that this dataset was prepared aroind 1989 so it is out of date especially information pertaing provinces (see `?maps::italy`).


```r
# Get the italy dataset from ggplot2
# Consider only the following provinces "Bergamo" , "Como", "Lecco", "Milano", "Varese"
# and arrange by group and order (ascending order)
italy_map <- ggplot2::map_data(map = "italy")
italy_map_subset <- italy_map %>%
  filter(region %in% c("Bergamo" , "Como", "Lecco", "Milano", "Varese")) %>%
  arrange(group, order)
```

Each observation in the dataframe defines a geographical point with some extra information:

* `long` & `lat`, longitude and latitude of the geographical point
* `group`, an identifier connected with the specific polygon points are part of
    * a map can be made of different polygons (e.g. one polygon for the main land and one for each islands, one polygon for each state, ...)  
* `order`, the order of the point within the specific `group`
    * how the all of the points being part of the same `group` should be connected in order to create the polygon  
* `region`, the name of the province (Italy) or state (USA) 
    

```r
head(italy_map, 3)
##       long      lat group order        region subregion
## 1 11.83295 46.50011     1     1 Bolzano-Bozen      <NA>
## 2 11.81089 46.52784     1     2 Bolzano-Bozen      <NA>
## 3 11.73068 46.51890     1     3 Bolzano-Bozen      <NA>
```

## How to work with maps 

Having spatial information in the data gives the opportunity to map the data or, in other words, __visualizing the information contained in the data in a geographical context__. R has different possibilities to map data, from normal plots using **longitude**/ **latitude** as `x`/ `y` to more complex spatial data objects (e.g. shapefiles).

### Mapping with `ggplot2`

The most basic way to create maps with your data is to use `ggplot2`, create a ggplot object and then, add a specific __geom__ mapping **longitute** to `x` aesthetic and **latitude** to `y` aesthetic [4] [5]. This simple approach can be used to:

* create maps of geographical areas (states, country, etc.)  
* map locations as points, lines, etc.

__Create a map showing "Bergamo", Como", "Varese" and "Milano" provinces in Italy using simple points...__

When plotting simple points the `geom_point` function is used. In this case the polygon and order of the points is not important when plotting.


```r
italy_map_subset %>%
  ggplot(aes(x = long, y = lat)) +
  geom_point(aes(color = region))
```

![](buildingDataVisualizationTools_part_04_files/figure-html/mapItalyExampleAsPoints-1.png)<!-- -->

__Create a map showing "Bergamo", Como", "Varese" and "Milano" provinces in Italy using lines...__

The `geom_path` function is used  to create such plots. From the R documentation, `geom_path` _"... connects the observation in the order in which they appear in the data"_. When plotting using `geom_path` is important to consider the polygon and the order within the polygon for each point in the map. 

The points in the dataset are grouped by `region` and ordered by `order`. If information about the region is not provided then the sequential order of the observations will be the order used to connect the points and, for this reason, "unexpected" lines will be drawn when moving from a `region` to the other. On the other hand if information about the region is provided using the `group` or `color` aesthetic, mapping to `region`, the "unexpected" lines are removed (see example below). 



```r
plot_1 <- italy_map_subset %>%
  ggplot(aes(x = long, y = lat)) +
  geom_path() +
  ggtitle("No mapping with 'region', unexpected lines")

plot_2 <- italy_map_subset %>%
  ggplot(aes(x = long, y = lat)) +
  geom_path(aes(group = region)) +
  ggtitle("With 'group' mapping")

plot_3 <- italy_map_subset %>%
  ggplot(aes(x = long, y = lat)) +
  geom_path(aes(color = region)) +
  ggtitle("With 'color' mapping")

grid.arrange(plot_1, plot_2, plot_3, ncol = 2, layout_matrix = rbind(c(1,1), c(2,3)))
```

![](buildingDataVisualizationTools_part_04_files/figure-html/mapItalyExampleAsLines-1.png)<!-- -->

Mapping with `ggplot2` is possible to create more sophisticated maps like choropleth maps [3]. The example below, extracted from [1],  shows how to visualize the percentage of republican votes in 1976 by states.


```r
# Get the USA/ state map from ggplot2
us_map <- ggplot2::map_data("state")

# Use the 'votes.repub' dataset (maps package), containing the percentage of 
# republican votes in the 1900 elections by state. Note
# - the dataset is a matrix so it needs to be converted to a dataframe
# - the row name defines the relevant state 

votes.repub %>%
  tbl_df() %>%
  mutate(state = rownames(votes.repub), state = tolower(state)) %>%
  right_join(us_map, by = c("state" = "region")) %>%
  ggplot(mapping = aes(x = long, y = lat, group = group, fill = `1976`)) +
  geom_polygon(color = "black") + 
  theme_void() +
  scale_fill_viridis(name = "Republican\nVotes (%)")
```

![](buildingDataVisualizationTools_part_04_files/figure-html/choroplethExampleUsa-1.png)<!-- -->





### Maps with `ggmap`, Google Maps API

TO BE DEFINED

### Maps using spatial objects in R 

TO BE DEFINED


## More on mapping

TO BE DEFINED

# References

[1] "Mapping" chapter in "[Mastering Software Development in R](https://bookdown.org/rdpeng/RProgDA/mapping.html/)" by Roger D. Peng, Sean Cross and Brooke Anderson, 2017  
[2] Italy Map, "UNESCO (1987) through UNEP/GRID-Geneva"  
[3] Choropleth map [Wikipedia](https://en.wikipedia.org/wiki/Choropleth_map)


## Previous "Building Data Visualization Tools" blogs

[4] "[Basic plotting with R and ggplot2](https://pparacch.github.io/2017/07/06/plotting_in_R_ggplot2_part_1.html)", Part 1  
[5] "['ggplot2', essential concepts](https://pparacch.github.io/2017/07/14/plotting_in_R_ggplot2_part_2.html)", Part 2  
[6] "[Guidelines for good plots](https://pparacch.github.io/2017/07/18/plotting_in_R_ggplot2_part_3.html)", Part 3

