
# MLDA Regression Disontinuity

MLDA Regression Discontinuity (based on data from Carpenter and Dobkin 2009) from 
Chapter 4 of *Mastering Metrics*,  Table 4.1 and Figures 4.2, 4.4, and 4.5 in Mastering Metrics. 
These present sharp RD estimates of the effect of the minimum legal drinking age (MLDA) on mortality.

Derived from [master_cd_rd.do](http://masteringmetrics.com/wp-content/uploads/2015/01/master_cd_rd.do).[^cd_rd].
See the [documentation](http://masteringmetrics.com/wp-content/uploads/2015/01/ReadMe_MLDA.txt).

[^cd_rd]: Gabriel Kreindler, June 13, 2014; Modified (lightly) by Jon Petkun, January 20, 2015.

Load libraries.

```r
library("tidyverse")
library("haven")
library("rlang")
library("broom")
```

Load the data from [AEJfigs.dta](http://masteringmetrics.com/wp-content/uploads/2015/01/AEJfigs.dta).

```r
filename <- here::here("data", "AEJfigs.dta")
```

```r
mlda <- read_dta(filename) %>%
  map_dfc(~ set_attrs(.x, NULL))
```

Add an indicator variable for individuals over 21 years of age.

```r
mlda <- mutate(mlda, 
               age = agecell - 21,
               over21 = as.integer(agecell >= 21))
```
Add a variable for other causes of death.

```r
mlda <- mutate(mlda, ext_oth = external - homicide - suicide - mva)
```


For "all causes", "motor vehicle accidents", and "internal causes" plot the
linear and quadratic trends on each side of 21.

```r
varlist <- c("all" = "All Causes", "mva" = "Motor Vehicle Accidents", 
             "internal" = "Internal Causes")
```

```r
mlda %>%
  select(agecell, over21, one_of(names(varlist))) %>%
  gather(response, value, -agecell, -over21, na.rm = TRUE) %>%
  mutate(response = recode(response, !!!as.list(varlist))) %>%
  ggplot(aes(x = agecell, y = value)) +
  geom_point() +
  geom_smooth(mapping = aes(group = over21), se = FALSE, method = "lm",
              formula = y ~ poly(x, 2)) +
  geom_smooth(mapping = aes(group = over21), se = FALSE, method = "lm",
              formula = y ~ x, color = "black") +
  facet_grid(response ~ ., scales = "free_y") +
  labs(y = "Mortality rate (per 100,000)", x = "Age")
```

<img src="mlda-rd_files/figure-html/unnamed-chunk-4-1.png" width="70%" style="display: block; margin: auto;" />


```r
responses <- c("mva", "suicide", "homicide", "ext_oth", "internal", "alcohol")
```

Function to run four regressions for a given response

```r
run_reg <- function(y) {
  mods <- list(
    "Ages 19-22, Linear" = 
      lm(quo(!!sym(y) ~ age * over21), data = mlda),
    "Ages 19-22, Quadratic" =
      lm(quo(!!sym(y) ~ poly(age, 2, raw = TRUE) * over21), data = mlda),
    "Ages 20-21, Linear" =
      lm(quo(!!sym(y) ~ age * over21), 
             data = filter(mlda, agecell >= 20, agecell <= 22)),
    "Ages 20-21, Quadratic" = 
      lm(quo(!!sym(y) ~ poly(age, 2, raw = TRUE) * over21), 
             data = filter(mlda, agecell >= 20, agecell <= 22))
  )
  out <- tibble(
    model = names(mods),
    ages = rep(c("19-22", "20-21"), each = 2),
    trend = rep(c("Linear", "Quadratic"), 2),
    model_num = seq_along(mods)
  ) %>%
  bind_cols(
    select(filter(map_df(mods, tidy), term == "over21"),  
           estimate, std.error)) %>%
    mutate(response = y)
  out[["df.residual"]] <- map_dfr(mods, glance)[["df.residual"]]
  out
}
   
mlda_regs <- map_dfr(responses, run_reg)
```


```r
mlda_regs %>%
  select(model, response, estimate, std.error) %>%
  gather(stat, value, estimate, std.error) %>%
  spread(model, value)
#> # A tibble: 12 x 6
#>   response stat    `Ages 19-22, Line… `Ages 19-22, Quad… `Ages 20-21, Lin…
#>   <chr>    <chr>                <dbl>              <dbl>             <dbl>
#> 1 alcohol  estima…              0.442              0.799             0.740
#> 2 alcohol  std.er…              0.152              0.222             0.227
#> 3 ext_oth  estima…              0.838              1.80              1.41 
#> 4 ext_oth  std.er…              0.489              0.724             0.636
#> 5 homicide estima…              0.104              0.200             0.164
#> 6 homicide std.er…              0.345              0.522             0.517
#> # ... with 6 more rows, and 1 more variable: `Ages 20-21, Quadratic` <dbl>
```


