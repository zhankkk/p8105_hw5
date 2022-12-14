hw5
================

``` r
library(tidyverse)
```

    ## ── Attaching packages ─────────────────────────────────────── tidyverse 1.3.2 ──
    ## ✔ ggplot2 3.4.0      ✔ purrr   0.3.5 
    ## ✔ tibble  3.1.8      ✔ dplyr   1.0.10
    ## ✔ tidyr   1.2.1      ✔ stringr 1.4.1 
    ## ✔ readr   2.1.3      ✔ forcats 0.5.2 
    ## ── Conflicts ────────────────────────────────────────── tidyverse_conflicts() ──
    ## ✖ dplyr::filter() masks stats::filter()
    ## ✖ dplyr::lag()    masks stats::lag()

``` r
library(viridis)
```

    ## Loading required package: viridisLite

``` r
knitr::opts_chunk$set(
    echo = TRUE,
    warning = FALSE,
    fig.width = 8, 
  fig.height = 6,
  out.width = "90%"
)

options(
  ggplot2.continuous.colour = "viridis",
  ggplot2.continuous.fill = "viridis"
)

scale_colour_discrete = scale_colour_viridis_d
scale_fill_discrete = scale_fill_viridis_d

theme_set(theme_minimal() + theme(legend.position = "bottom"))
```

problem 2

load and describe raw dataset

``` r
homicide_raw=read.csv("homicide.csv")
```

The homicide dataset describes homicides happened in 50 major American
cities with 52179 cases recorded over the past
decades. There are 12 variables included in the
data. Among all the variables, reported date,latitude and longitude are
recorded in form of numeric values while other 9 variables are character
values.

Some race ,age and gender are unknown for victims, accounting for part
of missing values other than ’na’s in latitude and longitude.

For latitude and longitude, the missing amounts are 60 and 60 as
suggested by raw homicide data

Specifically, a huge amount of victims with unknown race, gender and age
are observed in city Albuquerque in New Mexico state.

A typo error “Tulsa, AL” os observed, this entry could be removed in
later step during data cleaning.

data cleaning for homicide raw data

``` r
  homicide_df=
  homicide_raw%>%janitor::clean_names()%>%
  drop_na(lat,lon)%>%
   mutate(reported_date = as.Date(as.character(reported_date), format = "%Y%m%d")) %>% 
  mutate(
    city_state = str_c(city, state, sep = ", "),
    resolution = case_when(
      disposition == "Closed without arrest" ~ "unsolved",
      disposition == "Open/No arrest"        ~ "unsolved",
      disposition == "Closed by arrest"      ~ "solved"
    ))%>%
       filter(city_state != "Tulsa, AL") 
```

’na’s observed in latitude and longitude variables are dropped for
lucidity.Report date is modified to year\_month\_day format for better
visualization.

In addition, a new variable ‘city\_state’ is created combining city and
state separated by comma. Another variable disposition is created based
on whether the homicide case is solved with binary outcome of ‘solved’
and ‘unsolved’.  

The typo error “Tulsa, AL” is excluded in this data cleaning step.

Summarize total homicide and unsolved homicide based on variable
city\_state

``` r
homicide_city_df = 
  homicide_df %>% 
  select(city_state, disposition, resolution) %>% 
  group_by(city_state) %>% 
  summarize(hom_total = n(),
            hom_unsolved = sum(resolution == "unsolved"))
```

Based on the results, there are significantly much more cases in several
cities. For instance, Baltimore, Chicago and Philadelphia. Amount of
unsolved case is proportional to total amount of cases happened in the
city .

Using prop.test for summarizing homicide statistics in Baltimore

``` r
baltimore = 
  prop.test(x = filter(homicide_city_df, city_state == "Baltimore, MD") %>% pull(hom_unsolved),
            n = filter(homicide_city_df, city_state == "Baltimore, MD") %>% pull(hom_total)) 

broom::tidy(baltimore) %>% 
  knitr::kable(digits = 3)
```

| estimate | statistic | p.value | parameter | conf.low | conf.high | method                                               | alternative |
|---------:|----------:|--------:|----------:|---------:|----------:|:-----------------------------------------------------|:------------|
|    0.646 |   239.011 |       0 |         1 |    0.628 |     0.663 | 1-sample proportions test with continuity correction | two.sided   |

The estimate is 0.646 with a confidence interval of (0.628,0.663)
applying 1-sample proprotion test for proportion of unsolved homicides
in baltimore.

computing proportions of unsolved cases in each city using purrr package

``` r
test_results = 
  homicide_city_df %>% 
  mutate(prop_tests = map2(.x = hom_unsolved, .y = hom_total, ~prop.test(x = .x, n = .y)),
         tidy_tests = map(prop_tests, broom::tidy)) %>% 
  select(-prop_tests) %>% 
  unnest() %>% 
  select(city_state, estimate, conf.low, conf.high) %>% 
  mutate(city_state = fct_reorder(city_state, estimate))

test_results
```

    ## # A tibble: 50 × 4
    ##    city_state      estimate conf.low conf.high
    ##    <fct>              <dbl>    <dbl>     <dbl>
    ##  1 Albuquerque, NM    0.384    0.335     0.436
    ##  2 Atlanta, GA        0.383    0.353     0.415
    ##  3 Baltimore, MD      0.646    0.628     0.663
    ##  4 Baton Rouge, LA    0.462    0.414     0.511
    ##  5 Birmingham, AL     0.433    0.398     0.468
    ##  6 Boston, MA         0.505    0.465     0.545
    ##  7 Buffalo, NY        0.612    0.568     0.653
    ##  8 Charlotte, NC      0.300    0.266     0.336
    ##  9 Chicago, IL        0.736    0.724     0.747
    ## 10 Cincinnati, OH     0.445    0.408     0.483
    ## # … with 40 more rows

plotting a graph displaying estimate and confidence interval for number
of homicide cases in each city including error bars

``` r
test_results %>% 
  mutate(city_state = fct_reorder(city_state, estimate)) %>% 
  ggplot(aes(x = city_state, y = estimate)) + 
  geom_point() + 
  geom_errorbar(aes(ymin = conf.low, ymax = conf.high))+
  theme(axis.text.x = element_text(angle = 90, vjust = 0.5, hjust = 1)) +
  labs(
    title = "Estimates and 95% CIs of Proportion of Unsolved Homicides in All Cities",
    x = "City",
    y = "Estimate")
```

<img src="hw5_files/figure-gfm/unnamed-chunk-7-1.png" width="90%" />

Among all the cities involved, Richmond displayed the lowest unsolved
homicide proportion estimate with estimate around 0.27 while estimate
for Chicago is especially high reaching 0.75 for upper confidence level.

problem 3 simulation

conduct simulation to conduct t test to explore power

``` r
# Construct function for one-sample t-test
t_test = function(n = 30, mu, sigma = 5){
  sim_data = tibble(x = rnorm(n, mean = mu, sd = sigma)) 
    
  test_data = t.test(sim_data, mu = 0, conf.level = 0.95)
  
  sim_data %>% 
    summarize(
      estimate = pull(broom::tidy(test_data), estimate),
      p_value = pull(broom::tidy(test_data), p.value)
    )
}
# Generate 5000 data from the model and repeat the t-test for mu = 0,1,2,3,4,5,6
set.seed(1)
sim_result = 
  tibble(mu = c(0:6)) %>% 
  mutate(
    output_list = map(.x = mu, ~rerun(5000, t_test(mu = .x))),
    estimate_df = map(output_list, bind_rows)
    ) %>% 
  select(-output_list) %>% 
  unnest(estimate_df)
```

proportion of null hypothesis rejected Vs true mean

``` r
sim_result  %>% 
  group_by(mu) %>%
  filter(p_value < 0.05) %>% 
  summarize(rej_num = n(), rej_prop = rej_num/5000) %>% 
  ggplot(aes(x = mu, y = rej_prop)) + 
  geom_point(alpha = 0.8) +
  geom_line(alpha = 0.8) +
  geom_text(aes(label = round(rej_prop, 3)), vjust = -1, size = 3) + 
  scale_x_continuous(limits = c(0,6), breaks = seq(0,6,1)) +
  scale_y_continuous(limits = c(0,1), breaks = seq(0,1,0.2)) +
  labs(
    title = "rejected proprotion Vs true mean ",
    x = "True mean",
    y = "Proportion of times the null was rejected")
```

<img src="hw5_files/figure-gfm/unnamed-chunk-9-1.png" width="90%" />

Proportion of null hypothesis rejected increases from 0.051 to 1 as true
mean increases from 0 to 6. The proportion increases significantly with
mu from 0 to 4 reaching 0.992 then remains as 1 from 5 to 6 as it
reaches the maximum value.

a plot including average estimated mu (y axis) Vs true mu (x axis)

``` r
sim_result %>% 
  group_by(mu) %>% 
  summarise(avg_estimate = mean(estimate)) %>% 
  ggplot(aes(x = mu, y = avg_estimate)) +
  geom_point(alpha = 0.5) +
  geom_line(alpha = 0.5) +
  geom_text(aes(label = round(avg_estimate,2)), vjust = -1, size = 3) + 
  scale_x_continuous(limits = c(0,6.5), breaks = seq(0,6,1)) +
  scale_y_continuous(limits = c(-0.1,6.5), breaks = seq(0,6,1)) +
  labs(
    title = "average estimated mean Vs true mean ",
    x = "True mean",
    y = "Average estimated mean"
  ) 
```

<img src="hw5_files/figure-gfm/unnamed-chunk-10-1.png" width="90%" />

A linear relationship is observed between average estimated mean and
true mean. Average estimated mean is positively proportional to true
mean as indicated.

Make a plot showing the average estimate of mu on the y axis and the
true value of μ on the x axis. Make a second plot (or overlay on the
first) the average estimate of mu only in samples for which the null was
rejected on the y axis and the true value of mu on the x axis.

``` r
sim_rej = sim_result %>% 
  filter(p_value < 0.05) %>% 
  group_by(mu) %>% 
  summarise(avg_estimate = mean(estimate)) 
sim_result %>% 
  group_by(mu) %>% 
  summarise(avg_estimate = mean(estimate)) %>% 
  ggplot(aes(x = mu, y = avg_estimate, color = "Total samples")) +
  geom_point() +
  geom_line() + 
  geom_text(aes(label = round(avg_estimate,2)), vjust = 2, size = 5) + 
  geom_point(data = sim_rej, aes(color = "Rejected samples")) +
  geom_line(data = sim_rej, aes(x = mu, y = avg_estimate, color = "Rejected samples")) + 
  geom_text(data = sim_rej, aes(label = round(avg_estimate,2), color = "Rejected samples"), vjust = -1, size = 5) + 
  scale_x_continuous(limits = c(0,6.5), breaks = seq(0,6,1)) +
  scale_y_continuous(limits = c(-0.5,6.5), breaks = seq(0,6,1)) +
  labs(x = "true mean",
       y = "average estimate mean",
       title = "average estimates Vs true mean",
       color = "Type") +
  scale_color_manual(values = c("Total samples" = "#4393C3", "Rejected samples" = "#D6604D"))
```

<img src="hw5_files/figure-gfm/unnamed-chunk-11-1.png" width="90%" />

Average estimated mean and true mean are similar for total samples.
However, it is not the same case for rejected samples. Estimated average
means fluctuate corresponding to true mean with value 1 to 2. Both
estimated average means are higher than true mean and estimated average
mean is even higher when true mean is 1 comapred with that when true
mean is 2.

The difference could be caused by increase in effect size. Probability
of rejecting null hypothesis increases as effect size increases.
Specifically, null hypothesis is more easily rejected as significantly
detectable effect size is generated as true mean increases from 0 to 6.
The effect size is detected to be significant enough to reject null
hypothesis since true mean reaches 3.
