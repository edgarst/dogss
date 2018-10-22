Network reconstruction with dogss: DREAM5
================
Edgar Steiger
2018

-   [Simulations](#simulations)

This document shows and explains how to use the dogss package and how to reproduce Figures 14 and 15 from our paper [Sparse-Group Bayesian Feature Selection Using Expectation Propagation for Signal Recovery and Network Reconstruction](https://arxiv.org/abs/1809.09367).

First we need to load some packages that are required for comparisons and plotting (please install if not available on your machine):

``` r
library(dogss) # our method for sparse-group Bayesian feature selection with EP
library(glmnet) # standard lasso
library(gglasso) # group lasso
library(SGL) # sparse-group lasso
library(MBSGS) # Bayesian feature selection with Gibbs sampling

library(ggplot2) # for nice plots
library(ggthemes) # for even nicer plots
library(grid); library(gridExtra) # to arrange plots pleasantly

library(reshape2) # to melt data into "tidy" long-format

library(DescTools) # for area computations (AUROC, AUPR)
```

Furthermore we need to load three R files with additional code:

``` r
source("../auxiliary_rfunctions/my_cvSGL.R") # proper cross validation for SGL package

source("../auxiliary_rfunctions/my_theme.R") # functions to adjust ggplots
```

Finally, we provide all of the results on our simulated data to reconstruct the plots from the publication. If you wish to re-do all of the simulations/calculations, change the following parameter to `TRUE` (only do this if you have access to multiple cores):

``` r
selfcompute <- FALSE
B <- 100 # number of simulations
ncores <- 50 # number of cores used for parallelization
```

Simulations
-----------

...