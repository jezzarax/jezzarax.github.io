---
layout: post
title: Things to consider while doing a linear regression
---

## Main assumptions

* Validity
* Additivity
* Linearity
* Independence and equal variance of errors (we're talking about OLS)
* Normality of errors (might not be present, but its distribution has to be understood)

## Checklist before modeling

* Distribution of the target variable
* Distributions and correlations between exogenous variables
<!--more-->
* Missing values and outliers
* Interpretation of values (also applies to outlier identification) and data leakage
* Would an intercept make sense from interpretability point of view?
* Should we expect any variable interactions?

## Checklist for transformations

* Where do we need normalization?
* Where do we need one-hot encoding?
  * Do we really need it, can we split some variable values into ordinal bins?
* Any non-linear transformations would make sense here?
* Check correlation and distribution of interactions


## Checklist for fit and validation

* Does the sign of the coefficient make sense? Is it statistically significant?
* Does Rsq make sense? Would you expect it to be big or small and why?
* Is Rsq of validation set lower that of a training set?
* distribution and qqplot of residuals, does it make sense? If not, is there a transformation to make things better
  * Shapiro-Wilk, cook distance?
  * VIF is less then 10 for each variable
  * partial regression plots, do they make sense? Should some variables be kicked out?

## Things for production

* Do we get enough data, is it clean enough?
* How do we see if that data or the process changed enough to make the model invalid?
* Do we get the data quickly enough for the answer to make sense?
* Is the model interpretable enough and accepted by the business?
* How would the model errors affect the business process? And expected feedback loops?