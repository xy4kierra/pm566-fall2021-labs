Lab5-Data Wrangling
================
Xiaoyu Zhu
9/24/2021

## R Markdown

This is an R Markdown document. Markdown is a simple formatting syntax
for authoring HTML, PDF, and MS Word documents. For more details on
using R Markdown see <http://rmarkdown.rstudio.com>.

When you click the **Knit** button a document will be generated that
includes both content as well as the output of any embedded R code
chunks within the document. You can embed an R code chunk like this:

``` r
summary(cars)
```

    ##      speed           dist       
    ##  Min.   : 4.0   Min.   :  2.00  
    ##  1st Qu.:12.0   1st Qu.: 26.00  
    ##  Median :15.0   Median : 36.00  
    ##  Mean   :15.4   Mean   : 42.98  
    ##  3rd Qu.:19.0   3rd Qu.: 56.00  
    ##  Max.   :25.0   Max.   :120.00

## Including Plots

You can also embed plots, for example:

![](README_files/figure-gfm/pressure-1.png)<!-- -->

Note that the `echo = FALSE` parameter was added to the code chunk to
prevent printing of the R code that generated the plot.

``` r
if (!file.exists("met_all.gz"))
  download.file(
    url = "https://raw.githubusercontent.com/USCbiostats/data-science-data/master/02_met/met_all.gz",
    destfile = "met_all.gz",
    method   = "libcurl",
    timeout  = 60
    )
met <- data.table::fread("met_all.gz")
```

``` r
if (!require(data.table)){
  install.packages("data.table")}
```

    ## Loading required package: data.table

``` r
if (!require(gifski)){
  install.packages("gifski")}
```

    ## Loading required package: gifski

``` r
if (!require(gganimate)){
  install.packages("gganimate")}
```

    ## Loading required package: gganimate

    ## Loading required package: ggplot2

``` r
library(data.table)
library(tidyverse)
```

    ## ── Attaching packages ─────────────────────────────────────── tidyverse 1.3.1 ──

    ## ✓ tibble  3.1.5     ✓ dplyr   1.0.7
    ## ✓ tidyr   1.1.3     ✓ stringr 1.4.0
    ## ✓ readr   2.0.1     ✓ forcats 0.5.1
    ## ✓ purrr   0.3.4

    ## ── Conflicts ────────────────────────────────────────── tidyverse_conflicts() ──
    ## x dplyr::between()   masks data.table::between()
    ## x dplyr::filter()    masks stats::filter()
    ## x dplyr::first()     masks data.table::first()
    ## x dplyr::lag()       masks stats::lag()
    ## x dplyr::last()      masks data.table::last()
    ## x purrr::transpose() masks data.table::transpose()

``` r
library(dtplyr)
library(dplyr)
library(knitr)
library(leaflet)
```

``` r
stations <- fread("ftp://ftp.ncdc.noaa.gov/pub/data/noaa/isd-history.csv")
stations[, USAF := as.integer(USAF)]
```

    ## Warning in eval(jsub, SDenv, parent.frame()): NAs introduced by coercion

``` r
stations[, USAF   := fifelse(USAF == 999999, NA_integer_, USAF)]
stations[, CTRY   := fifelse(CTRY == "", NA_character_, CTRY)]
stations[, STATE  := fifelse(STATE == "", NA_character_, STATE)]
```

``` r
stations <- unique(stations[, list(USAF, CTRY, STATE)])

stations <- stations[!is.na(USAF)]

stations[, n := 1:.N, by = .(USAF)]
stations <- stations[n == 1,][, n := NULL]
```

## Merge station& met

``` r
df<-merge(
  x=met,
  y=stations,
  all.x = T,
  all.y = F,
  by.x = "USAFID",
  by.y = "USAF"
)
```

## Question 1: Representative station for the US

### generate a representative of each station. We will use the average (meadian could also be a good way to represent it, but it will depend on the case)

``` r
station_avg<-
df[,.(
  temp     = mean(temp,na.rm=T),
  wind.sp  = mean(wind.sp,na.rm =T),
  atm.press= mean(atm.press,na.rm= T)
),by=.(USAFID,STATE)]
```

### now we need to identify quantiles per variables

``` r
medians<-
  station_avg[,.(
  temp_50=quantile(temp,probs=.5,na.rm=T),
  wind.sp_50=quantile(wind.sp, probs=.5,na.rm=T),
  atm.press_50=quantile(atm.press,probs=.5,na.rm=T)
)]
medians
```

    ##     temp_50 wind.sp_50 atm.press_50
    ## 1: 23.68406   2.461838     1014.691

### now we can find the stations that are the closest to these values

``` r
station_avg[,temp_dist:=abs(temp-medians$temp_50)]
median_temp_station<-station_avg[order(temp_dist)][1]
median_temp_station
```

    ##    USAFID STATE     temp  wind.sp atm.press   temp_dist
    ## 1: 720458    KY 23.68173 1.209682       NaN 0.002328907

the median temperature

``` r
station_avg[,wind.sp_dist:=abs(wind.sp-medians$wind.sp_50)]
median_windsp_station<-station_avg[order(wind.sp_dist)][1]
median_windsp_station
```

    ##    USAFID STATE     temp  wind.sp atm.press temp_dist wind.sp_dist
    ## 1: 720929    WI 17.43278 2.461838       NaN  6.251284            0

the median wind speed

``` r
station_avg[,atm.press_dist:=abs(atm.press-medians$atm.press_50)]
median_atmpress_station<-station_avg[order(atm.press_dist)][1]
median_atmpress_station
```

    ##    USAFID STATE     temp  wind.sp atm.press temp_dist wind.sp_dist
    ## 1: 723200    GA 25.82436 1.537661  1014.692  2.140304    0.9241768
    ##    atm.press_dist
    ## 1:   0.0005376377

the median atm press

The station that is closest to the median temperature is 720458. The
station that is closest to the median wind speed is 720929. The station
that is closest to the median wind speed is 722238.

## Question 2: Representative station per state

### Just like the previous question, you are asked to identify what is the most representative, the median, station per state. This time, instead of looking at one variable at a time, look at the euclidean distance. If multiple stations show in the median, select the one located at the lowest latitude.

we first need to recover the state variable, by MERGEING :)

``` r
# we first need to recover the state variable, by MERGEING :)
station_avg2<-
df[,.(
  temp     = mean(temp,na.rm=T),
  wind.sp  = mean(wind.sp,na.rm =T),
  atm.press= mean(atm.press,na.rm= T)
),by=.(USAFID,STATE)]
```

``` r
# Get the medians by state for temperature, wind speed, and atm press
station_avg2[,temp_50:=quantile(temp,probs=.5,na.rm=T),by=STATE]

station_avg2[,wind.sp_50:=quantile(wind.sp,probs=.5,na.rm=T),by=STATE]

station_avg2[,atm.press_50:=quantile(atm.press,probs=.5,na.rm=T),by=STATE]
```

``` r
# Calculate Euclidean Distance
station_avg2[, eudist := sqrt(
  (temp - temp_50)^2 + (wind.sp - wind.sp_50)^2)]
```

``` r
# Lowest euclidean distance by State
station_avg2[ , .SD[which.min(eudist)], by = STATE]
```

    ##     STATE USAFID     temp  wind.sp atm.press  temp_50 wind.sp_50 atm.press_50
    ##  1:    CA 722970 22.76040 2.325982  1012.710 22.66268   2.565445     1012.557
    ##  2:    TX 722598 29.81293 3.521417       NaN 29.75188   3.413737     1012.460
    ##  3:    MI 725395 20.44096 2.357275  1015.245 20.51970   2.273423     1014.927
    ##  4:    SC 723107 25.95831 1.599275       NaN 25.80545   1.696119     1015.281
    ##  5:    IL 722076 22.34403 2.244115       NaN 22.43194   2.237622     1014.760
    ##  6:    MO 720479 24.14775 2.508153       NaN 23.95109   2.453547     1014.522
    ##  7:    AR 722054 26.58944 1.707136  1014.127 26.24296   1.938625     1014.591
    ##  8:    OR 720202 17.16329 1.828437       NaN 17.98061   2.011436     1015.269
    ##  9:    WA 720254 19.24684 1.268571       NaN 19.24684   1.268571           NA
    ## 10:    GA 722197 26.70404 1.544133  1015.574 26.70404   1.495596     1015.208
    ## 11:    MN 726553 19.67552 2.393582       NaN 19.63017   2.617071     1015.042
    ## 12:    AL 722286 26.35793 1.675828  1014.909 26.33664   1.662132     1014.959
    ## 13:    IN 724386 22.32575 2.243013  1014.797 22.25059   2.344333     1015.063
    ## 14:    NC 720864 24.82394 1.612864       NaN 24.72953   1.627306     1015.420
    ## 15:    VA 724006 24.31662 1.650539       NaN 24.37799   1.653032     1015.158
    ## 16:    IA 725464 21.37948 2.679227       NaN 21.33461   2.680875     1014.964
    ## 17:    PA 725204 21.87141 1.825605       NaN 21.69177   1.784167     1015.435
    ## 18:    NE 725565 21.86100 3.098367  1015.068 21.87354   3.192539     1014.332
    ## 19:    ID 725867 20.81272 2.702517  1012.802 20.56798   2.568944     1012.855
    ## 20:    WI 726413 18.94233 2.028610       NaN 18.85524   2.053283     1014.893
    ## 21:    WV 724176 21.94072 1.649151  1015.982 21.94446   1.633487     1015.762
    ## 22:    MD 722218 24.89883 1.883499       NaN 24.89883   1.883499     1014.824
    ## 23:    AZ 722745 30.31538 3.307632  1010.144 30.32372   3.074359     1010.144
    ## 24:    OK 720625 27.06188 3.865717       NaN 27.14427   3.852697     1012.567
    ## 25:    WY 726654 19.85844 3.775443  1014.107 19.80699   3.873392     1013.157
    ## 26:    LA 722041 27.84758 1.476664       NaN 27.87430   1.592840     1014.593
    ## 27:    KY 720448 23.52994 1.604905       NaN 23.88844   1.895486     1015.245
    ## 28:    FL 722011 27.56952 2.674074  1016.063 27.57325   2.705069     1015.335
    ## 29:    CO 724699 21.94228 2.844072       NaN 21.49638   3.098777     1013.334
    ## 30:    OH 724295 21.97211 2.803524  1015.742 22.02062   2.554397     1015.351
    ## 31:    NJ 724090 23.47238 2.148606  1015.095 23.47238   2.148606     1014.825
    ## 32:    NM 723658 24.94447 3.569281  1013.917 24.94447   3.776083     1012.525
    ## 33:    KS 724550 24.14958 3.449278  1013.315 24.21220   3.680613     1013.389
    ## 34:    ND 720911 18.34248 3.940128       NaN 18.52849   3.956459           NA
    ## 35:    VT 726115 18.60548 1.101301  1014.985 18.61379   1.408247     1014.792
    ## 36:    MS 722358 26.54093 1.747426  1014.722 26.69258   1.636392     1014.836
    ## 37:    CT 725087 22.57539 2.126514  1014.534 22.36880   2.101801     1014.810
    ## 38:    NV 724885 24.78430 2.600266  1013.855 24.56293   3.035050     1012.204
    ## 39:    UT 725750 24.23571 3.040962  1011.521 24.35182   3.145427     1011.972
    ## 40:    SD 726590 19.95928 3.550722  1014.284 20.35662   3.665638     1014.398
    ## 41:    TN 720974 24.71645 1.483411       NaN 24.88657   1.576035     1015.144
    ## 42:    NY 724988 20.44142 2.394383  1016.233 20.40674   2.304075     1014.887
    ## 43:    RI 725079 22.27697 2.583469  1014.620 22.53551   2.583469     1014.728
    ## 44:    MA 725088 21.20391 2.773018  1013.718 21.30662   2.710944     1014.751
    ## 45:    DE 724180 24.56026 2.752929  1015.046 24.56026   2.752929     1015.046
    ## 46:    NH 726116 19.23920 1.465766  1013.840 19.55054   1.563826     1014.689
    ## 47:    ME 726077 18.49969 2.337241  1014.475 18.79016   2.237210     1014.399
    ## 48:    MT 726798 19.47014 4.445783  1014.072 19.15492   4.151737     1014.185
    ##     STATE USAFID     temp  wind.sp atm.press  temp_50 wind.sp_50 atm.press_50
    ##         eudist
    ##  1: 0.25863745
    ##  2: 0.12378522
    ##  3: 0.11503023
    ##  4: 0.18095742
    ##  5: 0.08815689
    ##  6: 0.20409808
    ##  7: 0.41669854
    ##  8: 0.83755473
    ##  9: 0.00000000
    ## 10: 0.04853710
    ## 11: 0.22804190
    ## 12: 0.02531829
    ## 13: 0.12615513
    ## 14: 0.09550599
    ## 15: 0.06141630
    ## 16: 0.04490047
    ## 17: 0.18435880
    ## 18: 0.09500413
    ## 19: 0.27881642
    ## 20: 0.09051251
    ## 21: 0.01610412
    ## 22: 0.00000000
    ## 23: 0.23342190
    ## 24: 0.08341749
    ## 25: 0.11063960
    ## 26: 0.11920878
    ## 27: 0.46147583
    ## 28: 0.03121757
    ## 29: 0.51351277
    ## 30: 0.25380753
    ## 31: 0.00000000
    ## 32: 0.20680190
    ## 33: 0.23966295
    ## 34: 0.18672684
    ## 35: 0.30705883
    ## 36: 0.18795706
    ## 37: 0.20806041
    ## 38: 0.48789368
    ## 39: 0.15618171
    ## 40: 0.41362317
    ## 41: 0.19370133
    ## 42: 0.09673718
    ## 43: 0.25853660
    ## 44: 0.12000936
    ## 45: 0.00000000
    ## 46: 0.32641573
    ## 47: 0.30721662
    ## 48: 0.43107915
    ##         eudist

The station with the lowest euclidean distance between temperature and
wind speed is 723200.

## Question 3: In the middle?

### For each state, identify what is the station that is closest to the mid-point of the state. Combining these with the stations you identified in the previous question, use leaflet() to visualize all \~100 points in the same figure, applying different colors for those identified in this question.

``` r
# Calculate midpoint by State
station_avg2[, midpoint := sqrt(
  ((temp - temp_50)^2 + (wind.sp - wind.sp_50)^2) / 2
                              )]
```

``` r
# Lowest midpoint by State
map <- station_avg2[ , .SD[which.min(midpoint)], by = STATE]
```

``` r
# Create table to map lat/lon
hashtable <- df %>%
  select(USAFID, lat, lon)
hashtable <- distinct(hashtable, USAFID, .keep_all = TRUE)
```

``` r
# Merge lat/lon data
# Merge state information
map2 <- merge(x = map, y = hashtable, by.x = 'USAFID', by.y = "USAFID", all.x = TRUE, all.y = FALSE)
```

``` r
# Create leaflet map
mp.pal <- colorNumeric(c('darkgreen','goldenrod','brown'), domain=map2$midpoint)
# Leadlet map
tempmap <- leaflet(map2) %>% 
# The looks of the Map
  addProviderTiles('CartoDB.Positron') %>% 
# Some circles
  addCircles(
    lat = ~lat, lng=~lon,
    label = ~paste0(round(temp,2), ' C'), 
    color = ~ mp.pal(midpoint),
    opacity = 1, 
    fillOpacity = 1, 
    radius = 500
    ) %>%
# And a pretty legend
  addLegend('bottomleft', pal=mp.pal, values=map2$midpoint,
          title='Midpoint by State', opacity=1)
tempmap
```

<div id="htmlwidget-1994d8936f0d09e8a365" style="width:672px;height:480px;" class="leaflet html-widget"></div>
<script type="application/json" data-for="htmlwidget-1994d8936f0d09e8a365">{"x":{"options":{"crs":{"crsClass":"L.CRS.EPSG3857","code":null,"proj4def":null,"projectedBounds":null,"options":{}}},"calls":[{"method":"addProviderTiles","args":["CartoDB.Positron",null,null,{"errorTileUrl":"","noWrap":false,"detectRetina":false}]},{"method":"addCircles","args":[[45.417,46.683,37.578,36.9,36.75,35.937,46.942,35.178,28.29,29.445,35.135,40.2,33.355,38.533,33.212,31.183,32.969,32.167,33.817,33.587,36.744,36.666,40.033,39.643,39.674,39.84,40.412,39.05,39.909,39.417,42.571,41.533,41.736,42.584,40.767,42.267,41.674,41.433,41.196,42.542,44.45,43.344,43.626,43.417,44.969,45.45,44.381,45.698],[-123.817,-122.983,-84.77,-94.017,-97.35,-77.547,-98.018,-86.066,-81.437,-90.261,-90.234,-87.6,-84.567,-76.033,-87.616,-90.471,-96.836,-110.883,-118.15,-80.209,-108.229,-76.321,-74.35,-79.916,-75.606,-83.84,-86.937,-96.767,-105.117,-118.716,-77.713,-71.283,-72.651,-70.918,-80.4,-84.467,-93.022,-97.35,-112.011,-113.766,-68.367,-72.518,-72.305,-88.133,-95.71,-98.417,-106.721,-110.44],500,null,null,{"interactive":true,"className":"","stroke":true,"color":["#A52A2A","#006400","#D59923","#7B850D","#427204","#487405","#73830C","#76840C","#236901","#547806","#D9A520","#447304","#2E6C02","#006400","#1E6801","#73830C","#567906","#888A10","#938E12","#70820B","#7C860D","#366E03","#006400","#166701","#006400","#918D12","#577907","#8A8B10","#CF8B26","#D29225","#497405","#938E12","#7C860E","#547806","#72820B","#527706","#2C6C02","#487404","#657E09","#9C9114","#A89516","#A89516","#B19818","#457304","#85890F","#D8A420","#4F7706","#D9A221"],"weight":5,"opacity":1,"fill":true,"fillColor":["#A52A2A","#006400","#D59923","#7B850D","#427204","#487405","#73830C","#76840C","#236901","#547806","#D9A520","#447304","#2E6C02","#006400","#1E6801","#73830C","#567906","#888A10","#938E12","#70820B","#7C860D","#366E03","#006400","#166701","#006400","#918D12","#577907","#8A8B10","#CF8B26","#D29225","#497405","#938E12","#7C860E","#547806","#72820B","#527706","#2C6C02","#487404","#657E09","#9C9114","#A89516","#A89516","#B19818","#457304","#85890F","#D8A420","#4F7706","#D9A221"],"fillOpacity":1},null,null,["17.16 C","19.25 C","23.53 C","24.15 C","27.06 C","24.82 C","18.34 C","24.72 C","27.57 C","27.85 C","26.59 C","22.34 C","26.7 C","24.9 C","26.36 C","26.54 C","29.81 C","30.32 C","22.76 C","25.96 C","24.94 C","24.32 C","23.47 C","21.94 C","24.56 C","21.97 C","22.33 C","24.15 C","21.94 C","24.78 C","20.44 C","22.28 C","22.58 C","21.2 C","21.87 C","20.44 C","21.38 C","21.86 C","24.24 C","20.81 C","18.5 C","18.61 C","19.24 C","18.94 C","19.68 C","19.96 C","19.86 C","19.47 C"],{"interactive":false,"permanent":false,"direction":"auto","opacity":1,"offset":[0,0],"textsize":"10px","textOnly":false,"className":"","sticky":true},null,null]},{"method":"addLegend","args":[{"colors":["#006400 , #006400 0%, #5E7C08 16.88502862268%, #9E9114 33.77005724536%, #D9A420 50.65508586804%, #C97D28 67.54011449072%, #B7552A 84.4251431134%, #A52A2A "],"labels":["0.0","0.1","0.2","0.3","0.4","0.5"],"na_color":null,"na_label":"NA","opacity":1,"position":"bottomleft","type":"numeric","title":"Midpoint by State","extra":{"p_1":0,"p_n":0.844251431134001},"layerId":null,"className":"info legend","group":null}]}],"limits":{"lat":[28.29,46.942],"lng":[-123.817,-68.367]}},"evals":[],"jsHooks":[]}</script>

## Question 4: Means of means

### Using the quantile() function, generate a summary table that shows the number of states included, average temperature, wind-speed, and atmospheric pressure by the variable “average temperature level,” which you’ll need to create.

### Start by computing the states’ average temperature. Use that measurement to classify them according to the following criteria:low: temp &lt; 20 Mid: temp &gt;= 20 and temp &lt; 25 High: temp &gt;= 25

``` r
# create
df[,state_temp :=mean(temp,na.rm=T),by= STATE]
df[,temp_cat   :=fifelse(
  state_temp<20,"low-temp",
  fifelse(state_temp<25,"mid-temp","high_temp"))
  ]
```

### Once you are done with that, you can compute the following:

### Number of entries (records),

### Number of NA entries,

### Number of stations,

### Number of states included, and

### Mean temperature, wind-speed, and atmospheric pressure.

### All by the levels described before.

``` r
# Summary table
df[, .(
  N_entries = .N,
  N_stations = length(unique(USAFID)),
  N_missing = sum(is.na(.SD)),
  N_states = length(unique(STATE)),
  mean_temperature = mean(temp, na.rm = TRUE),
  mean_windspeed = mean(wind.sp, na.rm = TRUE),
  mean_atmpress = mean(atm.press, na.rm = TRUE)
), by = temp_cat]
```

    ##     temp_cat N_entries N_stations N_missing N_states mean_temperature
    ## 1:  mid-temp   1135423        781   1361521       25         22.39909
    ## 2: high_temp    811126        555   1015145       12         27.75066
    ## 3:  low-temp    430794        259    549625       11         18.96446
    ##    mean_windspeed mean_atmpress
    ## 1:       2.352712      1014.383
    ## 2:       2.514644      1013.738
    ## 3:       2.637410      1014.366

``` r
table(met$temp_cat,useNA = "always")
```


    ## 
    ## <NA> 
    ##    0
