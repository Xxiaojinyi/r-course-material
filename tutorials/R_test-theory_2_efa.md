Test Theory: Exploratory Factor Analyses
================
Philipp Masur
2021-11

-   [Introduction](#introduction)
    -   [Goals](#goals)
    -   [Steps](#steps)
-   [Preparation](#preparation)
    -   [Getting some data](#getting-some-data)
    -   [Checking the adequacy of the items for the factor
        analysis](#checking-the-adequacy-of-the-items-for-the-factor-analysis)
-   [Exploratory factor analysis](#exploratory-factor-analysis)
    -   [Deciding on an extraction
        method](#deciding-on-an-extraction-method)
    -   [Determining the number of
        factors](#determining-the-number-of-factors)
    -   [Estimating the model](#estimating-the-model)
    -   [Improving the model](#improving-the-model)
-   [Reliability](#reliability)

# Introduction

In an earlier [tutorial](tutorials/R_test-theory_1_cfa.md), we have
learned about the basics of test theory and an approach called
confirmatory factor analysis (CFA). In most cases, where theory informs
us about how a latent concept should be measured and what dimensionality
we should expect, using a CFA is the best choice. However, sometimes, we
may be unsure about the exact dimensionality hidden in a given item
pool. Generally speaking, we can think of two situation in which this is
the case:

1.  We want to develop a new scale for a certain latent concept (e.g.,
    literacy) and have created a large item pool that includes all
    aspects that we believe measure our latent concept. Now, we want to
    know whether there are any subdimensions that we need to account
    for.
2.  We have tested a unidimensional, confirmatory model, but it does not
    fit the data and we are unsure whether there are more dimensions
    hidden in the item pool.

In such cases, we may use an exploratory factor analysis (EFA) instead.

## Goals

EFA is an approach to detect structure in data. The main goal is to
identify one or several latent variables that explain common variance
between several indicator variables. Often, the goal is to reduce
several items to a number of overarching factors. We thereby transform
the covariance or correlation matrix in a way that it represents a
so-called factor structure.

The factor solution should at best have a “simple structure”, which
means that each indicator (e.g., each item in a questionnaire) loads
onto one factor. Practically, however, there are always cross-loadings
(which, however, hopefully are going to be minimal).

## Steps

Each EFA consists of several steps:

1.  We check whether our items are appropriate for conducting a
    exploratory factor analysis.

2.  We decide on a specific extraction methods and identify the number
    of factors.

3.  We decide on a particular rotation methods and run the factor
    analysis.

4.  We assess the resulting model based on model fit, factor loadings,
    reliability, and validity

To conduct an EFA in R, we can use the package `psych`, which contains
several useful functions.

``` r
library(tidyverse)
library(psych)
```

# Preparation

## Getting some data

For this tutorial, we will again assess the Big Five Personality Model
as assessed in the International Personality Item Pool (ipip.ori.org).
Whereas in the tutorial on [confirmatory factor
analyses](tutorials/R_test-theory_1_cfa.md), we used theory to derive
the dimensionality in the 25 item pool and thus a confirmatory model
that we could pass to lavaan, we now are going to try to detect these
dimensions exploratorily. We again recode items that were negatively
formulated. We also directly remove cases with missing values.

``` r
bfi_items <- bfi %>% 
  as_tibble %>%
  select(A1:O5) %>%
  na.omit %>%
  mutate(A1 = (A1-7)*-1,  
         C4 = (C4-7)*-1,
         C5 = (C5-7)*-1,
         E1 = (E1-7)*-1,
         E2 = (E2-7)*-1,
         O1 = (O1-7)*-1,
         O3 = (O3-7)*-1,
         O4 = (O4-7)*-1) 
```

## Checking the adequacy of the items for the factor analysis

In a first step, we should investigate whether at all the items are
adequate for a exploratory factor analysis. To this end, we can estimate
the so-called Measure of Sampling Adequacy (MSA) that has become know as
the Kaiser-Meyer-Olkin (KMO) index.

In basic terms, the statistic is a measure of the proportion of variance
among variables that might be common variance. The lower the proportion,
the higher the KMO-value, the more suited the data is to factor
analysis.

In his delightfully flamboyant style, Kaiser (1975) suggested that KMO
&gt; .9 were marvelous, in the .80s, mertitourious, in the .70s,
middling, in the .60s, medicore, in the 50s, miserable, and less than
.5, unacceptable.

``` r
KMO(bfi_items)

#'## Wahl der Extraktionsmethode
# Multivariate Normalverteilung
bfi_items %>%
  mardia(plot = FALSE) # Eher PAF als ML EFA 

# Normalverteilung der Einzelnen Items 
bfi_items %>%
  describe 
```

As we can see, we get an KMO/MSA index for all items (overall) and each
item individually. As all items are &gt; .70 (middling) or even &gt; .80
(mertitourious = deserving of reward or praise) and the overall score is
also &gt; .80, we can conclude the item pool is well adequate for
conducting a factor analysis.

Another way to assess the adequacy of the item pool for factor analysis
is simply to look at the bivariate correlations between all the
variables:

``` r
# Correlation matrix for Agreeableness
bfi_items %>%
  select(A1:A5) %>%
  cor %>%
  round(3)


# Correlation plot using the package `corrplot`
library(corrplot)
bfi_items %>% 
  select(A1:A5) %>%
  cor %>%
  corrplot(method = "color", 
           addCoef.col = "black", 
           tl.col = "black")
```

# Exploratory factor analysis

## Deciding on an extraction method

In the literature on EFA, three alternative approaches are usually
discussed:

-   Principal component analysis (PCA)
-   Principal axis factoring (PAF)
-   Maximum likelihood factor analysis (ML)

The PCA has been criticized a lot as it does not assume a reflective
measurement model. It assume that all variance in a manifest indicator
is explainable by a common factor.

The most used approaches to do a true exploratory *factor* analysis are
hence PAF and ML. Both rest on the same assumptions as the confirmatory
factor analysis and hence align with classical test theory. This
approaches thus assume that we can decompose the variance of each
measurement: Every observable measurement *Y* is composed of variance
(true score: *τ*<sub>*a*</sub>) explained by the latent variable,
variance (again a type of true score: *τ*<sub>*b*</sub>) explained by
the specific indicator (e.g., the item), and the measurement error *ϵ*:

*Y*<sub>*i*</sub> = *τ*<sub>*a*, *i*</sub> + *τ*<sub>*b*, *i*</sub> + *ϵ*<sub>*i*</sub>

## Determining the number of factors

Before we fit an EFA, we need to identify the optimal number of factors
that should be extracted. Unfortunately, there is no common approach or
criterion for extracting factors. Our decision should hence be based on
several criteria:

-   Theoretical plausibility
-   Reproducibility of the factors based on the correlation matrix

Approaches include:

-   Analysis of the Eigenvalues
    -   Kaiser criterion
    -   Parallel analysis
-   Very simple structure criterion
-   Empirical BIC
-   Root Mean Residual -…

Many scholars consider the parallel analysis as the best option to
extract the number of factors. Traditionally, we determine the number of
factors in a data matrix or a correlation matrix by examining the
“scree” plot of the successive eigenvalues. Sharp breaks in the plot
suggest the appropriate number of components or factors to extract.

The parallel analyis is an alternative approach that compares the scree
of factors of the observed data with that of a random data matrix of the
same size as the original. We can run this analysis in R using the
function `fa.parallel()`, in which we further specify the extraction
method (`fm = "pa"` to do principal axis factoring) and which eigen
values to show (`fa = "fa"`).

The function `nfactors()` computes some of the mentioned alternative
approaches.

``` r
# Parallel analysis
fa.parallel(bfi_items, fa = "fa", fm = "pa")

# Different approaches at the same time
nfactors(bfi_items)
```

The parallel analysis suggest 6 factors (not 5 as proposed by theory!).
So let us for now proceed with 6 factors and check out the factor
matrix.

## Estimating the model

To run the actual EFA, we can use the function `fa()` in which we
specify the number of factors, the methods (pa = principal axis
factoring) and how we want to rotate our matrix. Rotations minimize the
complexity of the factor loadings to make the structure simpler to
interpret.

Factor loading matrices are not unique, as for any solution involving
two or more factors there are an infinite number of orientations of the
factors that explain the original data equally well. Rotation of the
factor loading matrices attempts to give a solution with the best simple
structure.

There are two types of rotation:

-   Orthogonal rotations constrain the factors to be uncorrelated.
    Although often favored, in many cases it is unrealistic to expect
    the factors to be uncorrelated, and forcing them to be uncorrelated
    makes it less likely that the rotation produces a solution with a
    simple structure.
-   Oblique rotations permit the factors to be correlated with one
    another. Often produces solutions with a simpler structure and
    aligns with the theory.

``` r
# EFA
fit <- bfi_items %>%
  fa(nfactors = 6, 
     fm = "pa", 
     rotate="Promax") 

# Customize output
print(fit, 
      digits = 2,   ## round to 2 digits
      cut = .3,     ## remove loadings below .3
      sort = TRUE)  ## sort the items into the factors
```

The resulting output consists of several elements:

-   The pattern matrix: Which contains the standarized loadings of each
    items onto the respective factors. We omitted any loadings &lt; .30
    to ease interpretation.
    -   The column “h2” represents the amount of variance explained by
        the factors (also known as communality).
    -   The column “u2” represent uniqueness and i is simply 1-h2.  
    -   The columm “com” represents Hoffmann’s index of complexity. It
        equals 1 if an item loads on only one factor.
-   We further get tables for the variance accounted for and the
    inter-factor correlations
-   We further get extended information about the model fit (including
    fit indices such as RMSEA or TLI)

## Improving the model

It is quite clear that the model does not fit the data well. Most
importantly the sixth factor does not really add much to the overall
model. So let us rerun the model with only 5 factor

``` r
fit2 <- bfi_items %>%
  fa(nfactors = 5, 
     fm = "pa", 
     rotate="Promax") 

print(fit2, digits=2, 
      cut=.3, 
      sort=TRUE) 
```

The result looks much better and also more in line with what we would
expect (5 factors). There are still some cross-loadings (items with a
high complexity!). Let’s remove these as well.

``` r
bfi_items2 <- bfi_items %>%
  select(-c(N4,E5,A5,A4,O4))

fit3 <- bfi_items2 %>%
  fa(nfactors = 5, 
     fm = "pa", 
     rotate="Promax") 

print(fit3, digits=2, 
      cut=.3, 
      sort=TRUE) 
```

The model improved a lot. Factor loadings are now acceptable, there are
no considerable cross-loadings and also the model fit indices improved
(albeit the TLI is still not satisfactory).

We nonetheless stop for now and visualize the final model:

``` r
fa.diagram(fit3, digits = 2, cut = .1) 
```

# Reliability

Similar to the steps in a confirmatory factor analyses, we can further
investigate the reliability of the new found factors. We can compute the
same reliability estimates that we learned about in the tutorial on CFA
(e.g., Cronbach’s *α*, McDonald’s *ω*,…). Unfortunately, there is no
convenient function that directly extract these estimates from the fit
object. We hence have to create smaller objects that contain the
respective items and use the function `omega()` from the `psych` package
to get the estimates.

``` r
# Reliability of the first factor "neuroticism"
N.items <- bfi_items %>% select(N1, N2, N3, N5)
rel <- omega(N.items, nfactors = 1, plot = F)
rel$alpha     ## Cronbach's Alpha
rel$omega.tot ## McDonald's Omega
```

We can see that the internal consistency (*α* = .80) and composite
reliability (*ω* = .81) is good.
