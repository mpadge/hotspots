[![Build Status](https://travis-ci.org/mpadge/hotspotr.svg?branch=master)](https://travis-ci.org/mpadge/hotspotr) [![codecov](https://codecov.io/gh/mpadge/hotspotr/branch/master/graph/badge.svg)](https://codecov.io/gh/mpadge/hotspotr)

There are a number of statistical tools for evaluating the significance of observed spatial patterns of hotspots (such as those in [`spdep`](https://cran.r-project.org/package=spdep)). While such tools enable hotspots to be identified within a given set of spatial data, they do not allow quantification of whether the entire data set in fact reflects significant spatial structuring. For example, numbers of locally significant hotspots must be expected to increase with numbers of spatial observations.

The `R` package `hotspotr` enables the global significance of an observed spatial data set to be quantified by comparing both raw values and their local spatial relationships with those generated from a neutral model. If the global pattern of observed values is able to be reproduced by a neutral model, then any local hotspots may not be presumed significant regardless of the values of local statistics. Conversely, if the global pattern is unable to be reproduced by a neutral model, then local hotspots may indeed be presumed to be statistically significant.

The package is inspired by the work of Brown, Mehlman, & Stevens (Ecology 1995) and Ives & Klopfer (Ecology 1997). `hotspotr` follows the same premises as these two papers, in examining the extent to which rank--scale distributions can be reproduced by simple neutral models. `hotspotr` compares rank--scale distributions not only of the data of interest, but of corresponding local autocorrelation statistics.

Analysis involves first fitting a model using the function `fit_hotspot_model`, and then testing the significance of that using the function `p-values`.

The remainder of this README documents various exploratory and development phases ...

------------------------------------------------------------------------

Contents
========

[1. Brown vs Ives](#1-brown-ives)

[2. A Spatial Version of Brown et al (1995)?](#2-brown-spatial)

[3. Parallel and Rcpp Tests](#3-parallel)

[4. Tests (currently rubbish)](#4-tests)

------------------------------------------------------------------------

Install
-------

``` r
devtools::install_github ('mpadge/hotspotr')
```

------------------------------------------------------------------------

<a name="1-brown-ives"></a>1. Brown vs Ives
===========================================

The models of Brown and Ives are:

``` r
brown_core <- function (ntests=10, nlayers=10, size=10, sd0=0.1)
{
    y <- lapply (seq (ntests), function (i)
                 {
                     z <- rep (1, size ^ 2)
                     for (j in seq (nlayers)) 
                         z <- z * msm::rtnorm (size ^ 2, mean=1, sd=sd0, 
                                               lower=0, upper=2)
                     z <- sort (log10 (z), decreasing=TRUE)
                     (z - min (z)) / diff (range (z))
                 })
    colMeans (do.call (rbind, y))
}

ives_core <- function (ntests=10, nlayers=10, size=10, sd0=0.1)
{
    # The following 3 parameters are fixed by Ives et al
    a0 <- 0.05 
    s0 <- 0.5
    r0 <- 1.05

    y <- lapply (seq (ntests), function (i)
                 {
                     z <- rep (1, size ^ 2)
                     for (j in seq (nlayers))
                     {
                        svec <- msm::rtnorm (size ^ 2, mean=s0, sd=sd0, 
                                             lower=0, upper=2 * s0)
                        rvec <- msm::rtnorm (size ^ 2, mean=r0, sd=sd0, 
                                             lower=0, upper=2 * r0)
                        z <- svec * z + (1 + rvec / (1 + a0 * z))
                     }
                     z <- sort (log10 (z), decreasing=TRUE)
                     (z - min (z)) / diff (range (z))
                 })
    colMeans (do.call (rbind, y))
}
```

Define a re-usable function to plot the results

``` r
ploty <- function (nt, y)
{
    plot (NULL, NULL, xlim=c (1, size^2), ylim=c(0,1), xlab="rank", ylab="scale")
    cols <- rainbow (length (nt))
    y <- lapply (seq (length (y)), function (i) 
                     lines (seq (size^2), y [[i]], col=cols [i]))
    legend ("topright", lwd=1, col=cols, legend=nt)
}
```

Then compare them both

``` r
size <- 10
nlayers <- c (1, 10, 100)
ntests <- 10
yb <- lapply (nlayers, function (i) 
              brown_core (nlayers=i, ntests=ntests, size=size, sd0=1))
yi <- lapply (nlayers, function (i) 
              ives_core (nlayers=i, ntests=ntests, size=size))

plot.new ()
par (mfrow=c(1,2))
ploty (nlayers, yb)
title (main=paste ("brown"))

ploty (nlayers, yi)
title (main=paste ("ives"))
```

![](fig/ives-vs-brown-plot.png)

And the difference is clearly that Brown is able to respond to differences in numbers of layers, while Ives remains entirely invariant. In other words, the model of Ives et al (1997) may *not* be used as a neutral model because it is does not respond to structural differences!

The only other variable in both cases is `sd0`

``` r
size <- 10
nlayers <- 1
sd0 <- c (0.1, 0.2, 0.5)
ntests <- 10
yb <- lapply (sd0, function (i) 
              brown_core (nlayers=nlayers, ntests=ntests, size=size, sd0=i))
yi <- lapply (sd0, function (i) 
              ives_core (nlayers=nlayers, ntests=ntests, size=size, sd0=i))

plot.new ()
par (mfrow=c(1,2))
ploty (sd0, yb)
title (main=paste ("brown"))

ploty (sd0, yi)
title (main=paste ("ives"))
```

![](fig/ives-vs-brown-plot2.png)

And again, Ives does respond, but only marginally compared to Brown. Moreover, since the product of two normal distributions is also a normal distribution, changes in `nlayers` within the model of Brown et al (1995) are the same as changes in the distributional variance, and so only one of these parameters needs to be considered. The following graphs also include a Poisson distribution for comparison.

``` r
size <- 10
ntests <- 100
nlayers <- c (1, 10, 100)
sd0 <- c (0.001, 0.3, 1)
ylayers <- lapply (nlayers, function (i) 
              brown_core (nlayers=i, ntests=ntests, size=size, sd0=0.1))
ysd <- lapply (sd0, function (i) 
              brown_core (nlayers=1, ntests=ntests, size=size, sd0=i))

plot (NULL, NULL, xlim=c (1, size^2), ylim=c(0,1), xlab="rank", ylab="scale")
cols <- rainbow (3)
y <- lapply (seq (length (ylayers)), function (i) 
             {
                 lines (seq (size^2), ylayers [[i]], col=cols [i])
                 lines (seq (size^2), ysd [[i]], col=cols [i], lty=2)
             })

# Poisson distribution 
yp <- dpois (seq (size^2), 1, log=TRUE)
yp <- log10 (1 + yp - min (yp)) # convert to values > 0
yp <- (yp - min (yp)) / diff (range (yp))
lines (seq (size^2), yp)

ltxt <- sapply (nlayers, function (i) paste0 ('nlayers=', i))
ltxt <- c (ltxt, sapply (sd0, function (i) paste0 ('nsd=', i)), "Poisson")
legend ("bottomleft", lwd=1, col=c (rep (cols, 2), "black"), 
        lty=c (1,1,1,2,2,2,1), legend=ltxt)
```

![](fig/brown-plot.png)

It's obviously far more computationally efficient to consider variance rather than `nlayers`, and this also seems to allow a greater range of possible forms of response. Moreover, the response with increasing variance clearly approaches a Poisson distribution.

Note that although increasing the variance always decreases the relative probability of hotspots (that is, increases the width of the peak for low ranks), increasing the number of layers always decreases it. The graph nevertheless reveals that the response for a very large number of layers (here, 100) is accurately approximated by a very low variance (here, `sd=0.001`).

------------------------------------------------------------------------

<a name="2-brown-spatial"></a>2. A Spatial Version of Brown et al (1995)?
=========================================================================

There is no 'model' to speak of in *Brown et al (1995)*, rather they merely examine the rank-scale properties of normal distributions. Spatial relationships will act in this case simply by smoothing the normal distribution, which will do nothing other than again increase the distributional variance, as now demonstrated.

First define a function to generate spatial autocorrelation on a grid.

``` r
brown_space <- function (sd0=0.5, alpha=0.1, size=10, nt=100)
{
    xy <- cbind (rep (seq (size), each=size), rep (seq (size), size))
    dhi <- 1 # for rook; dhi=1.5 for queen
    nbs <- spdep::dnearneigh (xy, 0, dhi)
    wts <- lapply (nbs, function (x) rep (1, length (x)) / length (x))
    # then convert nbs to 2 vectors of origins and destinations
    nbso <- unlist (lapply (seq (nbs), function (i) rep (i, length (nbs [[i]]))))
    nbs <- unlist (nbs)
    wts <- unlist (wts)

    y <- lapply (seq (nt), function (i) 
                 {
                     z <- msm::rtnorm (size ^ 2, mean=1, sd=sd0, lower=0, upper=2)
                     z [nbso] <- (1 - alpha) * z [nbso] + alpha * z [nbs] * wts [nbs]
                     z <- sort (log10 (z), decreasing=TRUE)
                     (z - min (z)) / diff (range (z))
                 })
    colMeans (do.call (rbind, y))
}
```

Then compare different values of `alpha`

``` r
sd0 <- c (0.00001, 0.3, 1)
size <- 10
ysd <- lapply (sd0, function (i) brown_space (sd0=i, alpha=0, size=size))
alpha <- c (0.1, 0.5, 0.9)
yalpha <- lapply (alpha, function (i) brown_space (sd0=0.1, alpha=i, size=size))

plot (NULL, NULL, xlim=c (1, size^2), ylim=c(0,1), xlab="rank", ylab="scale")
cols <- rainbow (length (sd0))
for (i in 1:length (sd0)) 
{
    lines (seq (size^2), yalpha [[i]], col=cols [i])
    lines (seq (size^2), ysd [[i]], col=cols [i], lty=2)
}

ltxt <- sapply (alpha, function (i) paste0 ('alpha=', i))
ltxt <- c (ltxt, sapply (sd0, function (i) paste0 ('sd=', i)))
legend ("bottomleft", lwd=1, col=rep (cols, 2), lty=c (1,1,1,2,2,2),
        legend=ltxt)
```

![](fig/brown-space.png)

And in this case the introduction of spatial autocorrelation does enable the generation of rank--scale distributions that are notably more peaked than any non-spatially autocorrelation versions. This suggests the neutral models can most flexibly be based simply on the models of *Brown et al (1995)*---that is, simply normal distributions---with the additional aspect of spatial autocorrelation. The following two parameters therefore suffice:

1.  Variance of the (truncated) normal distribution; and

2.  Strength of spatial autocorrelation.

Temporal autocorrelation is formally equivalent to increasing the value of the first parameter, and may be merely **implicitly** rather than explicitly modelled.

------------------------------------------------------------------------

<a name="3-parallel"></a>3. Parallel and Rcpp Tests
===================================================

The current, non-parallel `Rcpp` version:

``` r
ntests <- 1000
dat <- ives (size=10, seed=1)
wts <- lapply (dat$nbs, function (x) rep (1, length (x)) / length (x))
nbs <- dat$nbs
```

``` r
set.seed (1)
system.time (
test1 <- rcpp_neutral_hotspots_ntests (nbs=nbs, wts=wts, alpha_t=0.1,
                  alpha_s=0.1, sd0=0.1, nt=10, ntests=ntests, ac_type='moran')
)
```

    ##    user  system elapsed 
    ##   0.508   0.004   0.513

Then write an equivalent `R` `lapply` version

``` r
rloop <- function (ntests=1000)
{
    z <- lapply (seq (ntests), function (i) {
                 z1 <- rcpp_neutral_hotspots (nbs, wts, alpha_t=0.1, 
                                              alpha_s=0.1, sd0=0.1, nt=10)
                 ac1 <- rcpp_ac_stats (z1, nbs, wts, "moran")
                 z1 <- (sort (z1, decreasing=TRUE) - min (z1)) / diff (range (z1))
                 rbind (z1, ac1)
                })
    ac <- colMeans (do.call (rbind, lapply (z, function (i) i [2,])))
    z <- colMeans (do.call (rbind, lapply (z, function (i) i [1,])))
    cbind (z, ac)
}
set.seed (1)
system.time ( test2 <- rloop (ntests=ntests))
```

    ##    user  system elapsed 
    ##   0.488   0.004   0.493

And then an `R`-internal parallel version. First the slightly different function definition:

``` r
require (parallel)
rParloop <- function (ntests=1000)
{
    z <- parLapply (clust, seq (ntests), function (i) 
                    {
                        z1 <- rcpp_neutral_hotspots (nbs, wts, alpha_t=0.1, 
                                                     alpha_s=0.1, sd0=0.1,
                                                     nt=10)
                        ac1 <- rcpp_ac_stats (z1, nbs, wts, "moran")
                        z1 <- (sort (z1, decreasing=TRUE) - min (z1)) / 
                                diff (range (z1)) 
                        rbind (z1, ac1)
                    })
    ac <- colMeans (do.call (rbind, lapply (z, function (i) i [2,])))
    z <- colMeans (do.call (rbind, lapply (z, function (i) i [1,])))
    cbind (z, ac)
}
```

Note that `makeCluster` with `type="FORK"` automatically attaches all environment variables, but is not portable, as detailed here: <http://www.r-bloggers.com/how-to-go-parallel-in-r-basics-tips/>. The `clusterExport` and `clusterCall` lines explicitly attach only the required bits.

``` r
#clust <- makeCluster (parallel::detectCores () - 1, type="FORK")
clust <- makeCluster (parallel::detectCores () - 1)
clusterExport (clust, "dat")
clusterExport (clust, "wts")
clusterExport (clust, "nbs")
invisible (clusterCall (clust, function () {
                        while (length (grep ('hotspotr', getwd ())) > 0) 
                            setwd ("..")
                        devtools::load_all ("hotspotr")
                        setwd ("./hotspotr")
                                         }))
```

``` r
system.time (test3 <- rParloop (ntests=ntests))
```

    ##    user  system elapsed 
    ##   0.012   0.000   0.196

``` r
stopCluster (clust)
```

And the results look like this ...

``` r
plot (1:length (nbs), test3 [,1], "l", xlab="rank", ylab="scale")
lines (1:length (nbs), test3 [,2], col="gray")
legend ("topright", lwd=1, col=c("black", "gray"), bty="n", legend=c("z", "ac"))
```

![](fig/plot-parallel.png)

The parallel version does not of course generate identical results, because each core starts with its own random seed, but nevertheless after

``` r
ntests
```

    ## [1] 1000

the differences are very small:

``` r
max (abs (test1 - test2)); max (abs (test1 - test3))
```

    ## [1] 1.665335e-15

    ## [1] 0.004418125

The first of these is mere machine rounding tolerance; the second is a measure of convergence of randomised mean profiles.

------------------------------------------------------------------------

<a name="4-tests"></a>4. Tests
==============================

Test\#1
-------

First a demonstration that the model of Ives & Klopfer (Ecology 1997) can be reproduced with a simpler neutral model. This uses the function `ives` which generates random values on a square grid of dimensions `(size, size)`. The resultant ranks therefore range from 1 to 10× 10=100.

``` r
plot.new ()
dat <- ives (size=10, seed=1)
mod <- fit_hotspot_model (z=dat$dat$z, nbs=dat$nbs)
p <- p_values (z=dat$dat$z, nbs=dat$nbs, alpha=mod$alpha, nt=mod$nt, plot=TRUE)
```

![](fig/demo-moran.png)

The panel on the left shows the rank--scale distribution of the data produced by the model of Ives & Klopfer (`test data`), along with the mean rank--scale distribution of a series of neutral models. The two are highly similar, and the corresponding probability (`p=0.62`) indicates no significant difference between them. The panel on the right performs the same analysis on the local autocorrelation statistics, again revealing no significant difference with a neutral model.

The default spatial autocorrelation statistic is Moran's I, with analyses also possible using Geary's C:

``` r
plot.new ()
mod <- fit_hotspot_model (z=dat$dat$z, nbs=dat$nbs, ac_type='geary')
p_values (z=dat$dat$z, nbs=dat$nbs, alpha=mod$alpha, nt=mod$nt, ac_type='geary', 
          plot=TRUE)
```

![](fig/demo-geary.png)

And the Getis-Ord statistic:

``` r
plot.new ()
mod <- fit_hotspot_model (z=dat$dat$z, nbs=dat$nbs, ac_type='getis')
p_values (z=dat$dat$z, nbs=dat$nbs, alpha=mod$alpha, nt=mod$nt, ac_type='getis', 
          plot=TRUE)
```

![](fig/demo-getis.png)

Test\#2
-------

Then test the more complex case of the `meuse` data from the `sp` package, which contain topsoil heavy metal concentrations near Meuse, NL.

``` r
data (meuse, package='sp')
names (meuse)
```

    ##  [1] "x"       "y"       "cadmium" "copper"  "lead"    "zinc"    "elev"   
    ##  [8] "dist"    "om"      "ffreq"   "soil"    "lime"    "landuse" "dist.m"

The function `test_hotspots` requires data to be tested, a list of neighbours (constructed here as `knearneigh`), and a matching list of weights (constructed here as inverse-distance weights):

``` r
xy <- cbind (meuse$x, meuse$y)
nbs <- spdep::knn2nb (spdep::knearneigh (xy, k=4))
dists <- spdep::nbdists (nbs, xy)
d1 <- lapply (dists, function (i) 1/i)
```

Spatial patterns for the different metals can then be statistically compared with neutral models:

``` r
analyse <- function (metal='copper', ntests=1000)
{
    z <- meuse [metal] [,1]
    mod <- fit_hotspot_model (z=z, nbs=nbs)
    p_values (z=z, nbs=nbs, alpha=mod$alpha, nt=mod$nt, ntests=ntests,
              plot=TRUE)
}
```

For demonstration purposes, `ntests=1000` is sufficient, but larger values will generate more reliable estimates. These functions can be quite time-consuming.

``` r
analyse (metal='cadmium', ntests=1000)
```

![](fig/meuse-cadmium.png)

``` r
analyse (metal='copper', ntests=1000)
```

![](fig/meuse-copper.png)

``` r
analyse (metal='lead', ntests=1000)
```

![](fig/meuse-lead.png)

``` r
analyse (metal='zinc', ntests=1000)
```

![](fig/meuse-zinc.png)

These plots demonstrate that in all cases, the observed values themselves (`z` in the figures) can not be reproduced by a neutral model, yet the actual spatial relationships between them can. This indicates that the generating processes can be modelled by assuming nothing other than simple spatial autocorrelation acting on a more complex, non-spatial process.
