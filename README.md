Drug overdose deaths data analysis
================
Anton Malko
March 19 2018

-   [Outline](#outline)
-   [Data cleaning / prep](#data-cleaning-prep)
-   [Data visualization](#data-visualization)
    -   [Drug related deaths by state](#drug-related-deaths-by-state)
    -   [Change in drug related deaths across the years](#change-in-drug-related-deaths-across-the-years)
    -   [Deaths by drug type](#deaths-by-drug-type)
-   [Conclusions](#conclusions)
    -   [Comparison to similar analyses](#comparison-to-similar-analyses)
-   [R session info](#r-session-info)

Outline
=======

The primary goal of this analysis is to practice creating maps in R using `ggplot2`. The dataset we choose to use is the CDC data on drug overdose deaths. It comes from [here](https://data.cdc.gov/NCHS/VSRR-Provisional-Drug-Overdose-Death-Counts/xkb8-kh2a). The R code assumes that the dataset is located in the same folder and is named `data.csv`.

The dataset contains data on drug overdoses by state, year and month from January 2015 to August 2017. It reports overall number of deaths, number of deaths from drug overdose, and (for some states) more detailed breakdown of deaths causes by drug type (following the classification of ICD-10 - see [here](http://www.icd10data.com/ICD10CM/Codes/S00-T88/T36-T50/T40-) for codes explanations). More details on the dataset are available [here](https://www.cdc.gov/nchs/nvss/vsrr/drug-overdose-data.htm).

Our goal will be to estimate the number of drug users in each state. Since this is just a learning project, we will follow a very simple strategy: for each state, calculate the proportion of drug-related deaths to overall number of deaths, and assume this as a proxy of drug usage in this state. In a real life scenario, we would obviously want to use a more sophisticated strategy:

-   By looking at the proprtion of drug-related deaths to overall deaths, we are assuming that the number of overall deaths is a good proxy for the population density in a given state. This may not be a safe assumption, and a better strategy would be to just look up population statistics directly.
-   We are also assuming that the number of deaths is a good proxy for the number of drug users. This is likely not a valid assumption: e.g. one could imagine that a good health system in a given state would reduce number of deaths among drug addicts. Mortality rates can differ depending on the age group, type of the drug used, frequency of use and other factors. Ideally one would use multiple predictors to estimate the number of drug addicts in a given state. Furthemore, the dataset itself comes with several limitations:

-   The death causes in the `Indicator` column are not exclusive, as outlined in *Technical details* [here](https://www.cdc.gov/nchs/nvss/vsrr/drug-overdose-data.htm). There is no way of knowing how many deaths were counted multiple times from this dataset alone, so the results we get will necessarily be approximate.
-   The number of deaths (overall and drug-related ones) may be underreported for certain states and time periods, as indicated by the notes in the dataset.

Finally, let us note a couple of details about interpreting the deaths counts in this dataset:

-   It appears to me that the deaths specified as "Number of Drug Overdose Deaths" do not include deaths listed under more specific causes. It probably indicates that "Overdose deaths" include deaths by other types of drugs and/or deaths for which the type of drug was not specified. I will provide some evidence for this being the case. The documentation for the datset doesn't seem to discuss this.

-   Deaths categorized under "Opioids" are those deaths for which no specific class of opioids was provided. Thus, some proportion of these deaths would have been classified as "Natural & semi-synthetic opioids" and "Synthetic opioids, excl. methadone", if we had complete information. This means that the estimates for these two categories are likely underestimates.

Data cleaning / prep
====================

As the documentation for the data specifies, the figures provided for each month are in fact figures for 12-month-ending periods, "defined as the number of deaths occurring in the 12-month period ending in the month indicated. For example, the 12-month ending period in June 2017 would include deaths occurring from July 1, 2016, through June 30, 2017." The data for year 2017 ends in August (i.e. covering the period from September 1, 2016 to August 31, 2017). Since this is the time window covering the most recent period, we choose to use the same time window for the other years. That is, in the discussion below "2015" will stand for the period from Sep 1, 2014 to August 31, 2015 and similarly for 2016 and 2017.

The code block below deals with a few data enhancements:

-   Turning necessary columns to factors: `Year`, `Month`, `Indicator`
-   Putting ICD indicators of deaths cause for specific drugs into a separate column
-   Adding short convenient names for death causes

<details>
    <summary>
        <b>Show code</b>
    </summary>

``` r
library(dplyr)
library(ggplot2)
library(ggmap)
library(grid)
library(gridExtra)
```

``` r
dat <- read.csv("data.csv", header = TRUE)

# Only choose data with month == "August", as discussed above
dat <- dat[dat$Month == "August",]

# ---- Data enhancements for convenience ----

# change Year to factor
dat$Year <- factor(dat$Year)

# rename values in the Indicator column for convenience. First, some of them
# have ICD-10-CM codes. Wouldn't want to completely lose them - so put them in a 
# separate column
dat <- tidyr::separate(dat, Indicator, sep = "\\(", fill = "right", into = c("Indicator", "ICD_code"))
dat$ICD_code <- gsub("\\)", "", dat$ICD_code)
dat$Indicator <- stringr::str_trim(dat$Indicator)


# Add short names for the Indicator variable. First, we make a table
# listing which short name corresponds to which long name, and then we merge
# it into the full dataset
dat$Indicator <- factor(dat$Indicator, levels = 
                          c("Number of Deaths",
                            "Number of Drug Overdose Deaths",
                            "Heroin",
                            "Cocaine",
                            "Methadone",
                            "Natural & semi-synthetic opioids",
                            "Opioids",
                            "Percent with drugs specified",
                            "Psychostimulants with abuse potential",
                            "Synthetic opioids, excl. methadone"))

indic_short <- data.frame(Indicator = factor(c("Number of Deaths",
                                               "Number of Drug Overdose Deaths",
                                               "Heroin",
                                               "Cocaine",
                                               "Methadone",
                                               "Natural & semi-synthetic opioids",
                                               "Opioids",
                                               "Percent with drugs specified",
                                               "Psychostimulants with abuse potential",
                                               "Synthetic opioids, excl. methadone"),
                                             levels = c("Number of Deaths",
                                                        "Number of Drug Overdose Deaths",
                                                        "Heroin",
                                                        "Cocaine",
                                                        "Methadone",
                                                        "Natural & semi-synthetic opioids",
                                                        "Opioids",
                                                        "Percent with drugs specified",
                                                        "Psychostimulants with abuse potential",
                                                        "Synthetic opioids, excl. methadone")),
                          
                          indic_short = factor(c("n_deaths",
                                                 "n_od_deaths",
                                                 "heroin",
                                                 "cocaine",
                                                 "methadone",
                                                 "nat_opioids",
                                                 "nonspec_opioids",
                                                 "perc_drugs_spec",
                                                 "psychostim",
                                                 "synth_opioids"),
                                               levels = c("n_deaths",
                                                          "n_od_deaths",
                                                          "heroin",
                                                          "cocaine",
                                                          "methadone",
                                                          "nat_opioids",
                                                          "nonspec_opioids",
                                                          "perc_drugs_spec",
                                                          "psychostim",
                                                          "synth_opioids")))

dat <- dplyr::inner_join(dat, indic_short, by = "Indicator")
```

</details>

------------------------------------------------------------------------

Below I briefly discuss the evidence for my earlier claim that deaths listed as "Number of Drug Overdose Deaths" do not include deaths listed with a more specific cause. We check whether total number of deaths from drug overdose include deaths from specific drugs.

First we figure out for which states we even have specific death causes listed:

``` r
specific_states <- dat %>%
  dplyr::filter(!Indicator %in% c("Number of Deaths", "Number of Drug Overdose Deaths", "Percent with drugs specified")) %>% # only leave states with specific death causes
  dplyr::pull(State) %>% # pull out state column
  unique() %>% # find unique values
  droplevels()

levels(specific_states)
```

    ##  [1] "CT" "DC" "MD" "ME" "NC" "NH" "NM" "NV" "NY" "OK" "OR" "RI" "SC" "US"
    ## [15] "UT" "VA" "VT" "WA" "WV" "YC"

Then for each state reporting specific causes of death we count the number of "specific" deaths and the number of death listed as simply overdose (OD) death. If specific deaths are only a subset of OD deaths, we would expect the counts for OD to be equal or bigger than the sum of counts for specific causes.

It doesn't seem to be the case though: the first output below is the first few counts of OD deaths, and the second output is the first few counts of specific deaths. In fact, there are more deaths which are listed with specific death causes. This makes me think that the OD deaths is the "elsewhere" category.

<details><summary><b>Show code</b></summary>

``` r
# count deaths
deaths_by_cause <- dat %>%
  dplyr::filter(State %in% specific_states, !Indicator %in% c("Number of Deaths", "Number of Drug Overdose Deaths", "Percent with drugs specified")) %>% 
  dplyr::group_by(State, Year) %>%
  dplyr::summarize(count = sum(Data.Value))

deaths_general <- dat %>%
  dplyr::filter(Indicator == "Number of Drug Overdose Deaths", State %in% specific_states) %>%
  dplyr::group_by(State, Year) %>%
  dplyr::summarize(count = sum(Data.Value))

head(deaths_general)
```

    ## # A tibble: 6 x 3
    ## # Groups:   State [2]
    ##    State   Year count
    ##   <fctr> <fctr> <dbl>
    ## 1     CT   2015   728
    ## 2     CT   2016   928
    ## 3     CT   2017  1056
    ## 4     DC   2015   115
    ## 5     DC   2016   239
    ## 6     DC   2017   356

``` r
head(deaths_by_cause)
```

    ## # A tibble: 6 x 3
    ## # Groups:   State [2]
    ##    State   Year count
    ##   <fctr> <fctr> <dbl>
    ## 1     CT   2015  1516
    ## 2     CT   2016  2199
    ## 3     CT   2017  2648
    ## 4     DC   2015   217
    ## 5     DC   2016   529
    ## 6     DC   2017   867

</details>

Data visualization
==================

Let's first make a map showing the number of drug deaths by state. Obviously, death counts alone will be misleading (states with higher population will have more deaths in any event), so we will plot the proportion of drug deaths to all deaths in a states. As discussed in the beginning, this is still going to be a very crude estimate of number of drug-related deaths and especially of the number of drug users.

First, we calculate the proportion of drug deaths. We'll make it in two variants: pooling all drug related deaths, and keeping all categories separate. We will also add a column showing change in (proportion of) drug deaths from year to year. To get difference scores, we will subtract the proportion of drug deaths for year N-1 from the proportion of drug deaths for year N. Positive numbers will indicate increase in proportion of drug deaths relative to the previous year, negative numbers - decrease.

The first code block below deals with the transformations of the drug data; the second one - prepares the map data.

<details><summary><b>Show data transformation code</b></summary>

``` r
# ---- Prepare drug deaths data ---- 

# lower case the state full names - will be useful later for mapping
dat$State.Name_lower <- tolower(dat$State.Name)

deaths_props <- list()

deaths_props$pooled <- dat %>%
  dplyr::group_by(State, Year, indic_short, State.Name_lower) %>%
  dplyr::filter(!Indicator %in% c("perc_drug_spec")) %>%
  dplyr::mutate(type = ifelse(indic_short == "n_deaths", "all", "drugs")) %>%
  dplyr::group_by(State, State.Name_lower, Year, type) %>%
  dplyr::summarize(count = sum(Data.Value)) %>%
  tidyr::spread(key = type, value = count) %>%
  dplyr::summarize(prop = drugs/all) %>%
  tidyr::spread(key = Year, value = prop) %>%
  dplyr::mutate(`2015_dif` = 0, # there are no data on previous years, so pretend there is no diff.
                `2016_dif` = `2016` - `2015`,
                `2017_dif` = `2017` - `2016`) %>%
  tidyr::gather(key = "yr_diff", value = "diff", `2015_dif`, `2016_dif`, `2017_dif`) %>%
  tidyr::gather(key = "Year", value = "prop", `2015`, `2016`, `2017`) %>%
  dplyr::mutate(yr_diff = gsub("_dif", "", yr_diff)) %>%
  dplyr::filter(yr_diff == Year)
  

deaths_props$split <- dat  %>%
  dplyr::group_by(State, Year, indic_short, State.Name_lower) %>%
  dplyr::filter(!indic_short %in% c("perc_drug_spec"), State %in% specific_states) %>%
  dplyr::summarize(count = sum(Data.Value)) %>%
  tidyr::spread(key = indic_short, value = count) %>%
  dplyr::group_by(State, Year, State.Name_lower) %>%
  dplyr::summarize(od_prop = n_od_deaths/n_deaths,
                   heroin_prop = heroin/n_deaths,
                   cocaine_prop = cocaine/n_deaths,
                   methadone_prop = methadone/n_deaths,
                   nonspec_opioids_prop = nonspec_opioids/n_deaths,
                   nat_opioids_prop = nat_opioids/n_deaths,
                   synth_opioids_prop = synth_opioids/n_deaths,
                   psychostim_prop = psychostim/n_deaths) %>%
  tidyr::gather(key = "type", value = "prop", ends_with("_prop")) %>%
  dplyr::mutate(type = gsub("_prop", "", type)) %>%
  dplyr::mutate(type = factor(type, levels = c("cocaine", "heroin", "methadone",
                                               "nat_opioids", "synth_opioids",
                                               "nonspec_opioids", "psychostim", "od")))
```

</details>

<details><summary><b>Show map preparation code</b></summary>

``` r
#  ---- Prepare map data ----

# Most of the advice taken from:
# https://eriqande.github.io/rep-res-web/lectures/making-maps-with-R.html

# load map data for stats
states <- map_data("state")

plot_data <- dplyr::inner_join(states, deaths_props$pooled, by = c("region" = "State.Name_lower"))

# Advice on names centering is taken from here:
# https://stackoverflow.com/questions/9441436/ggplot-centered-names-on-a-map#9441528
stnames <- aggregate(cbind(long, lat) ~ State, plot_data, 
                     FUN=function(x)mean(range(x)))

# adjust labels for FL, MI and ID - otherwise they don't quite stay within
# their respective states
stnames[stnames$State == "FL", "long"] <- -81.5
stnames[stnames$State == "MI", "long"] <- -84.5
stnames[stnames$State == "ID", "long"] <- -114.5
stnames[stnames$State == "ID", "lat"] <- 44

# define some common theme elements
theme_map <- theme(axis.text = element_blank(),
  axis.line = element_blank(),
  axis.ticks = element_blank(),
  panel.border = element_blank(),
  panel.grid = element_blank(),
  axis.title = element_blank())

theme_legend <- theme(strip.text.x = element_text(size = 15),
                      legend.title=element_text(size = 15), 
                      legend.text=element_text(size = 13),
                      plot.title = element_text(size=16))
```

</details>

Drug related deaths by state
----------------------------

The plot below visualizes the proportion of drug related deaths by state and year, pooling all deaths. Since for most of the states the information on specific death causes is not provided, we do not create maps facetted by cause of deaths.

The maps suggest that there are a few regions in the US where (reported) drug deaths are more common. If we take this as a proxy measure for drug use (with the caveats listed earlier), it would appear that the following regions have a higher proportion of drug users:

-   New Mexico
-   Utah
-   Nevada
-   Oregon
-   Washington
-   A number of East Coast states, in particular West Virgina, New Hampshire and Maine

<details><summary><b>Show code</b></summary>

``` r
# merge map data and drug data and feed this to ggplot2
plot_data %>%
  ggplot(aes(x = long, y = lat)) +
  geom_polygon(aes(group = group, fill = prop), color = "darkgrey") +
  coord_map() +
  scale_fill_distiller("Proportion of \ndrug deaths\n",
                       palette = "YlOrRd", direction = 1) + 
  geom_text(data = stnames, aes(x = long, y = lat, label = State)) +
  facet_wrap( ~ Year, ncol = 2) +
  theme_map +
  theme_legend +
  ggtitle("Proportion of drug related deaths")
```
</details>

![](README_files/figure-markdown_github/make-maps-1.png)  

Change in drug related deaths across the years
----------------------------------------------

### Plots building details

The plots below show change in the proportion of drug deaths over the years. The plot for 2015 is just the baseline plot with proportion of drug deaths in the period from Sep 2014 to Aug 2015. It appears different from the 2015 plot in the previous image because the color scale is different - in the previous plots it had to account for the range of data in 2015-2017, so it had a bigger max value. So in the current plot the values of 0.1 will appear as red, while in the previous plot the where only orange-ish (0.2 corresponding to red).

Each plot shows the change in the proportion of drug related death relative to the previous 12 months period. Notice that in general deaths proportions are non-decreasing. If we plot it as is, the color scale will be non-symmetrical (with more mass above 0). To avoid this, we artifically extend the color scale in the negative direction (assigning min = -max), so that the absence of change is plotted in the "neutral" color in the middle of the color scale.

### Plots discussion

The plots suggest that the proportions of drug related deaths mostly remain stable. The most noticeable exception are the states in the East Coast. In the period from Sep 2015 to Aug 2016, proportion of drug related deaths grew (compared to the same period in 2014-2015) in a number of East Coast states, including Maryland, Maine, West Virgina and Connecticut, as well as DC. In the period from Sep 2016 to Aug 2017 the increase from the previous 12-month period is less pronounced , although proprtion of deaths is still slightly growing in the East Coast compared to the rest of the country; this is especially true for Maryland and West Virginia.

Interestingly, in the period of from Sep 2016 to Aug 2017 a few of the "hot" states see a decrease in the proportion of drug related deaths: these include Utah, Wyoming and Washington. However, these states experienced a slight increase in the previous yearly period, from Sep 2015 to Aug 2016, so it is an interesting question of whether the decrease in the most recent time period indicates just a return to the baseline.

<details><summary><b>Show code</b></summary>

``` r
plot_2015 <- plot_data %>%
  dplyr::filter(Year == 2015) %>%
  ggplot(aes(x = long, y = lat)) +
  geom_polygon(aes(group = group, fill = prop), color = "darkgrey") +
  coord_map() +
  scale_fill_distiller("Proportion of \ndrug deaths\n",
                       palette = "YlOrRd", direction = 1) + 
  geom_text(data = stnames, aes(x = long, y = lat, label = State),
            size = 3) +
  theme_map +
  theme_legend + 
  ggtitle("Sep 2014 - Aug 2015")

plot_2016 <- plot_data %>%
  dplyr::filter(Year == 2016) %>%
  ggplot(aes(x = long, y = lat)) +
  geom_polygon(aes(group = group, fill = diff), color = "darkgrey") +
  coord_map() +
  scale_fill_distiller("Change in \nproportion of \ndrug deaths\n", 
                       palette = "RdBu", limits = c(-max(plot_data$diff), max(plot_data$diff))) + 
  geom_text(data = stnames, aes(x = long, y = lat, label = State),
            size = 3) +
  theme_map +
  theme_legend + 
  ggtitle("Sep 2015 - Aug 2016")

plot_2017 <- plot_data %>%
  dplyr::filter(Year == 2017) %>%
  ggplot(aes(x = long, y = lat)) +
  geom_polygon(aes(group = group, fill = diff), color = "darkgrey") +
  coord_map() +
  scale_fill_distiller("Change in \nproportion of \ndrug deaths\n",
                       palette = "RdBu", limits = c(-max(plot_data$diff), max(plot_data$diff))) + 
  geom_text(data = stnames, aes(x = long, y = lat, label = State), size = 3) +
  theme_map +
  theme_legend +
  ggtitle("Sep 2016 - Aug 2017")

grid.arrange(plot_2015, plot_2016, plot_2017, ncol = 2, nrow = 2)
```
</details>

![](README_files/figure-markdown_github/diff-maps-1.png) 

Deaths by drug type
-------------------

Since the detailed breakdown of deaths causes is only available for 20 states, we are not making maps broken down by drug type. It is still, interesting, however, to see whether the proportion of deaths caused by a certain drug changes through time. This is what the next plot visualizes.

Keep in mind that "Non-specific opioids" likely include deaths caused by "natural opioids" and "synthetic opioids" - so, being a catch-all category, it will be inflated, and the estimates for more specific categories will be underestimates. Also keep in mind that it is not clear waht "Other" category actually includes: only deaths by drugs other than the specified ones, or deaths by drugs for which no details on the drug were provided, or both.

The plots suggest that in most of the states, for which detailed information is available, the causes of death remain relatively stable through time. A few exceptions jump in the eye, however: DC, Maryland, West Virginia, where we observe increase in deaths caused by opioids (yellow line), in particular, synthetic ones (orange line). Deaths from other/non-specified drugs also increase (pink line). This observation is consistent with what we observed earlier in the maps.

<details><summary><b>Show code</b></summary>

``` r
dplyr::inner_join(states, deaths_props$split, by = c("region" = "State.Name_lower")) %>%
  ggplot() +
  geom_line(aes(x = Year, y = prop, color = type, group = type)) +
  theme(axis.text.x = element_text(angle = 45, hjust = 1, size = 9)) +
  scale_color_brewer("Drug type",
                     type = "qual", palette = "Set1",
                     labels = c("Cocaine", "Heroine", "Methadone", 
                                "Natural/semisynthetic opioids", "Synthetic opioids (excl. methadone)",
                                "Non-specific opioids", "Psychostimulants", "Other/Not specified")) +
  facet_wrap(~State) + 
  theme_legend + 
  theme(axis.text.x = element_text(size =13 ),
        axis.text.y = element_text(size = 13),
        axis.title.x = element_text(size = 13),
        axis.title.y = element_text(size = 13),
        plot.title = element_text(size=16)) +
  ggtitle("Change in proportion of drug related deaths by drug type")
```
</details>

![](README_files/figure-markdown_github/deaths-by-drug-type-1.png) 

Conclusions
===========

Our short analysis appears to suggest two things.

First, the states with high proportions of drug related deaths are mostly concentrated on the East and West of the country. Of course, this is were the population is concentrated, but since we are looking at proportions, the differences in the baseline for the population size would have been already accounted for to some degree. If we take the death data to be a valid proxy for prevalence of drug users, the above would suggest that eastern and western state are in the worse shape realtive to the rest of the country. However, with the caveats discussed in the beginning of the report, this remains purely speculative.

Second, the states on the East Coast, especially Maryland and West Virginia (as well as DC), seem to have an increase in the proportion of drug related deaths for the last couple of years (in particular, due to synthetic opioid drugs other than methadone).

### Comparison to similar analyses

It is interesting to see how our analysis compares to already existing ones.

1.  First, we can look at the maps of drug deaths data provided by CDC, the source of the data we used. They can be accessed [here](https://www.cdc.gov/nchs/pressroom/sosmap/drug_poisoning_mortality/drug_poisoning.htm). They look quite similar to our first plot ("Proportion of drug related deaths") in the overall pattern (many states with high proportion of deaths on the East and West of the country), although the CDC plots identifies more states as problematic (including Ohio, Kentucky, Tennessee and Pennsylvania). It is worth noting that CDC normalizes the data using actual population density (they report number of deaths per 100 000 people), while we were using the overall number of deaths as proxy for population density. The differences between ours and CDC plots suggest that this proxy is not a very good one.

2.  Another analysis we can look at is [this](https://wallethub.com/edu/drug-use-by-state/35150/) analysis of the number of drug users, discussed at WalletHub. We have made an assumption tht the proportion of drug related deaths can be used as a good proxy for the number of drug addicts in a given states. If this is true, the states with most drug addicts would include Utah, New Mexico, West Virginia, New Hampshire, Nevada, Washington, Maine. The analysis reported on WalletHub:

    -   lists Washington among the 5 states with the highest number of adult drug users (keep in mind, though, that our data was not broken down by age);
    -   lists West Virginia and New Hampshire as the two states with the highest number of drug deaths (which is consistent with what we observed; although neither of other states we identified in our dataset made it in the top 5 according to WalletHub)
    -   lists New Hampshire as number 4 in the overall rating of drug-related issues (with number 1 being the worst). Maine is listed 7th, Washington 8th, New Mexico - 11th, West Virginia - 14th, Nevada - 19th, Utah - 45th.

Overall, it appears that the states that our analysis identifed as problematic, do appear as problematic in WalletHub analysis as well, at least along some dimensions (except for Utah, which appears to be a blatant outlier - no idea why it seems to have a high proportion of drug related deaths, but ranks so low on problmeatic states list). Since WalletHub used more predictors and more information, it should not be surprising that their predictions are more fine-grained. It should also be kept in mind that the discussion of methodology used in WalletHub analysis is rether informal - i.e. they mention the weights that were assigned to their predictor variables, but not why those particular weights were chosen. It is conceiavable that choosing a different weighting schema or a different set of predictors would affect their ratings.

R session info
==============

<details><summary><b>Show</b></summary>

``` r
sessionInfo()
```

    ## R version 3.3.2 (2016-10-31)
    ## Platform: x86_64-apple-darwin13.4.0 (64-bit)
    ## Running under: OS X El Capitan 10.11.6
    ## 
    ## locale:
    ## [1] en_US.UTF-8/en_US.UTF-8/en_US.UTF-8/C/en_US.UTF-8/en_US.UTF-8
    ## 
    ## attached base packages:
    ## [1] grid      stats     graphics  grDevices utils     datasets  methods  
    ## [8] base     
    ## 
    ## other attached packages:
    ## [1] maps_3.2.0    bindrcpp_0.2  gridExtra_2.3 ggmap_2.7.900 ggplot2_2.2.1
    ## [6] dplyr_0.7.4  
    ## 
    ## loaded via a namespace (and not attached):
    ##  [1] Rcpp_0.12.14       RColorBrewer_1.1-2 plyr_1.8.4        
    ##  [4] bindr_0.1          bitops_1.0-6       tools_3.3.2       
    ##  [7] digest_0.6.13      evaluate_0.10.1    tibble_1.3.4      
    ## [10] gtable_0.2.0       pkgconfig_2.0.1    png_0.1-7         
    ## [13] rlang_0.2.0        mapproj_1.2-4      yaml_2.1.16       
    ## [16] stringr_1.2.0      knitr_1.18         RgoogleMaps_1.4.1 
    ## [19] rprojroot_1.3-2    tidyselect_0.2.3   glue_1.2.0        
    ## [22] R6_2.2.2           jpeg_0.1-8         rmarkdown_1.8     
    ## [25] tidyr_0.7.2        purrr_0.2.4        magrittr_1.5      
    ## [28] backports_1.1.2    scales_0.5.0.9000  htmltools_0.3.6   
    ## [31] assertthat_0.2.0   colorspace_1.3-2   labeling_0.3      
    ## [34] stringi_1.1.6      lazyeval_0.2.1     munsell_0.4.3     
    ## [37] rjson_0.2.15

</details>
