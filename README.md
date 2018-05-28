
<!-- README.md is generated from README.Rmd. Please edit that file -->
A detailed description can be found [here.](https://adaemmerp.github.io/lpirfs/README_docs.html)

lpirfs
======

An R-package which estimates linear and non-linear impulse responses with local projections by [Jordà (2005)](https://www.aeaweb.org/articles?id=10.1257/0002828053828518).

Main features
-------------

-   Estimate linear and non-linear impulse responses with local projections.
-   Functions to plot linear and non-linear impulse responses.
-   Functions are partly implemented in *Rcpp* and *RcppArmadillo* to improve efficiency.
-   High performance with parallel computation.

Installation
------------

You can install the released version of **lpirfs** from [CRAN](https://CRAN.R-project.org):

``` r
install.packages("lpirfs")
```

And the development version from [GitHub](https://github.com/):

``` r
# install.packages("devtools")
devtools::install_github("AdaemmerP/lpirfs")
```

Example: Linear impulse responses
---------------------------------

Load libraries:

``` r
# Load packages
  library(dplyr)
  library(doSNOW)
  library(parallel)
  library(vars)
  library(Rcpp)
  library(lpirfs)
```

Load data set from package to estimate a simple, new-Keynesian, closed- economy model. These data are used by [Jordà (2005)](https://www.aeaweb.org/articles?id=10.1257/0002828053828518) in chapter IV. Please see the data's help file or the original paper for a detailed description.

``` r
# Load data (from package)
  data_set_df <- interest_rules_var_data
```

Make list and specify input variables to estimate linear impulse responses.

``` r
# Make list for inputs
  specs <- list()

# Specify inputs
  specs$lags_lin       <- 4L      # Number of lags
  specs$lags_criterion <- NaN     # Lag length criterion (AICc, AIC or BIC)
  specs$max_lags       <- NaN     # If lags_criterion is chosen, set maximum number of lags  
  specs$trend          <- 0L      # 0 = no trend, 1 = trend, 2 = trend and quadratic trend
  specs$shock_type     <- 1L      # 0 = standard deviation shock, 1 = unit shock
  specs$confint        <- 1.96    # Width of confidence bands: 1 = 68%, 1.67 = 90%, 1.96 = 95%
  specs$hor            <- 12L     # Length of horizon
```

Estimate linear impulse responses.

``` r
  results_lin  <- lp_lin(data_set_df, specs)
```

Make plots for impulse responses of linear model.

``` r
  linear_plots <- plot_lin_irfs(results_lin)
```

Display single impulse responses:

-   The first plot shows the response of the first variable (GDP\_gap) to a shock in the first variable in (GDP\_gap).
-   The second plot shows the response of the first variable (GDP\_gap) to a shock in the second variable (Inflation).

``` r
  linear_plots[[1]]
```

<img src="man/figures/README-unnamed-chunk-7-1.png" style="display: block; margin: auto;" />

``` r
  linear_plots[[2]]
```

<img src="man/figures/README-unnamed-chunk-7-2.png" style="display: block; margin: auto;" />

Display all plots:

-   This graph is similar to Figure 5 in Jordà (2005), p. 176, but has slightly wider confidence bands.
-   Figure 5 in Jordà (2005) is replicated when specs$confint = 1.67 (90% error bands).

``` r
# Show all plots
  library(ggpubr)
  library(gridExtra)

  lin_plots_all <- sapply(linear_plots, ggplotGrob)
  marrangeGrob(lin_plots_all, nrow = ncol(data_set_df), ncol = ncol(data_set_df), top=NULL)
```

<img src="man/figures/README-unnamed-chunk-8-1.png" style="display: block; margin: auto;" />

Example: Non-linear impulse responses
-------------------------------------

Load libraries:

``` r
# Load packages
  library(dplyr)
  library(doSNOW)
  library(parallel)
  library(Rcpp)
  library(lpirfs)
  library(mFilter)
  library(vars)
```

Load data set from package to estimate a non-linear, new-Keynesian, closed- economy model. This data set is used by [Jordà (2005)](https://www.aeaweb.org/articles?id=10.1257/0002828053828518) in chapter IV. Please see the data's help file or the original paper for a detailed description.

``` r
# Load data (from package)
  data_set_df <- interest_rules_var_data
```

Make list and specify input variables to estimate non-linear impulse responses.

``` r
# Make list for inputs
  specs <- list()

# Specify inputs
  specs$lags_lin       <- 4L      # Number of lags for linear VAR to identify the shock
  specs$lags_nl        <- 3L      # Number of lags
  specs$lags_criterion <- NaN     # Lag length criterion (AICc, AIC or BIC)
  specs$max_lags       <- NaN     # If lags_criterion is chosen, maximum number of lags  
  specs$trend          <- 0L      # 0 = no trend, 1 = trend, 2 = trend and quadratic trend
  specs$shock_type     <- 1L      # 0 = standard deviation shock, 1 = unit shock
  specs$confint        <- 1.96    # Width of confidence bands: 1 = 68%, 1.67 = 90%, 1.96 = 95%
  specs$hor            <- 12L     # Length of horizon
```

Provide a switching variable to separate the data into two regimes.

``` r
# Specifications for switching variable
  specs$switching      <- data_set_df$GDP_gap # The output gap is used to differentiate between two regimes
  specs$hp_filter      <- 1                   # 0 = Do not use HP-filter to decompose switching-variable, 
                                              # 1 = Use HP-filter to decompose switching-variable
  specs$lambda         <- 1600                # Monthly = 129600, Quarterly = 1600, Annual = 6.25
  specs$gamma          <- 3                   # Numeric value > 0
```

The switching variable (*z*) is either decomposed by the Hodrick-Prescott filter (*specs$hp\_filter = 1*) or directly plugged into the follwing switching function.

$$F({z\_t})  = \\  \\frac{exp(-\\gamma \\ z\_t)}{(1 + exp(-\\gamma \\ z\_t))}$$

The lagged exogenous variables (*X*<sub>*t* − *p*</sub>) are then multiplied with the values of the transition function at *t − 1* where

$$\\hat{IR}(t,s,d) = \\hat{B}\_{1, }^s   $$
 - Regime 1: *X*<sub>*t* − *p*</sub> \* (1 − *F*(*z*<sub>*t* − 1</sub>)), - Regime 2: *X*<sub>*t* − *p*</sub> \* *F*(*z*<sub>*t* − 1</sub>).

$$
E = \\frac{mc^2}{\\sqrt{1-\\frac{v^2}{c^2}}}
$$

IMPORTANT:The index of *z* is set to *t − 1* to avoid contemporaneous feedback (see [Auerbach and Gorodnichenko; 2012](https://www.aeaweb.org/articles?id=10.1257/pol.4.2.1) for details). The lag is created automatically in *create\_nl\_data* of the package. If you do not want the exogenous variables to be multiplied with lagged values of *F*(*z*<sub>*t*</sub>), you have to provide the switching variable with a lead of one.

Estimate non-linear impulse responses.

``` r
  results_nl <- lp_nl(data_set_df, specs)
```

Save values from transition function.

``` r
  fz      <- results_nl$fz
```

Plot output gap

``` r
# Make date sequence. Start in October because the non-linear model is estimated with three lags.
  dates   <- seq(as.Date("1955/10/1"), as.Date("2003/1/1"), by = "quarter")
  data_df <- data_frame(x = dates, fz = fz, gdp_gap = specs$switching[(specs$lags_nl+1):length(data_set_df$FF)])

# Plot  
  ggplot(data = data_df) +
    geom_line(aes(x = x, y = gdp_gap)) +
    theme_bw() +
    ylab("") +
    xlab("Date") +
    scale_x_date(date_breaks = "5 year",  date_labels = "%Y")
```

<img src="man/figures/README-unnamed-chunk-15-1.png" style="display: block; margin: auto;" />

Plot tranistion function

``` r
  ggplot(data = data_df) +
    geom_line(aes(x = x, y = fz)) +
    theme_bw() +
    ylab("") +
    xlab("Date") +
    scale_x_date(date_breaks = "5 year",  date_labels = "%Y")
```

<img src="man/figures/README-unnamed-chunk-16-1.png" style="display: block; margin: auto;" />

Create and save all plots.

``` r
    nl_plots <- plot_nl_irfs(results_nl)
```

Show first impulse response of each regime:

-   The first plot shows the response of the first variable (GDP\_gap) to a shock in the first variable (GDP\_gap) in regime 1.
-   The second plot shows the response of the first variable (GDP\_gap) to a shock in the second variable (Inflation) in regime 2.

``` r
# Load packages
  library(ggpubr)
  library(gridExtra)

# Save plots based on states
  s1_plots <- sapply(nl_plots$gg_s1, ggplotGrob)
  s2_plots <- sapply(nl_plots$gg_s2, ggplotGrob)

  plot(s1_plots[[1]])
```

<img src="man/figures/README-unnamed-chunk-18-1.png" style="display: block; margin: auto;" />

``` r
  plot(s2_plots[[2]])
```

<img src="man/figures/README-unnamed-chunk-18-2.png" style="display: block; margin: auto;" />

Show all impulse responses of regime 1.

``` r
  marrangeGrob(s1_plots, nrow = ncol(data_set_df), ncol = ncol(data_set_df), top =  NULL)
```

<img src="man/figures/README-unnamed-chunk-19-1.png" style="display: block; margin: auto;" />

Show all impulse responses of regime 2.

``` r
  marrangeGrob(s2_plots, nrow = ncol(data_set_df), ncol = ncol(data_set_df), top = NULL)
```

<img src="man/figures/README-unnamed-chunk-20-1.png" style="display: block; margin: auto;" />

References
----------

-   Auerbach, A. J., and Gorodnichenko Y. (2012). "Measuring the Output Responses to Fiscal Policy." *American Economic Journal: Economic Policy*, 4 (2): 1-27. [doi:10.1257/pol.4.2.1](https://www.aeaweb.org/articles?id=10.1257/pol.4.2.1)

-   Jordà, O. (2005) "Estimation and Inference of Impulse Responses by Local Projections." *American Economic Review*, 95 (1): 161-182. [doi:10.1257/0002828053828518](https://www.aeaweb.org/articles?id=10.1257/0002828053828518)

-   Newey W.K., and West K.D. (1987). “A Simple, Positive-Definite, Heteroskedasticity and Autocorrelation Consistent Covariance Matrix.” *Econometrica*, 55, 703–708. [doi:10.2307/1913610](https://www.jstor.org/stable/1913610?seq=1#page_scan_tab_contents)

-   Ramey, V.A., and Zubairy, S. (2018). "Government Spending Multipliers in Good Times and in Bad: Evidence from US Historical Data." Journal of Political Economy, 126 (2), 850-901. [doi:10.1086/696277](https://www.journals.uchicago.edu/doi/10.1086/696277)

Acknowledgements
----------------

I greatly benefitted from the profound *R* and *Rcpp* knowledge of [Philipp Wittenberg](https://github.com/wittenberg). I am also grateful to [Sarah Zubairy](https://sites.google.com/site/sarahzubairy/) for sending me the *Matlab* code before the publication their [paper](https://www.journals.uchicago.edu/doi/10.1086/696277).

Development
-----------

I am currently working on a function to include exogenous variables and a function that allows to identify the shocks besides a Cholesky decomposition.

### Author

Philipp Adämmer

### License

GPL (&gt;= 2)