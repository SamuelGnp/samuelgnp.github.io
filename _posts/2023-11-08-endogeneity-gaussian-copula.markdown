---
title: "Navigating the Two-Way Street: Addressing Simultaneity and Price Endogeneity in Demand and Supply Estimation"
layout: post
date: 2023-11-07 19:05
image: /assets/images/markdown.jpg
headerImage: false
tag:
- econometrics
- Simultaneity
- Endogeneity
category: blog
author: samuelgnap
description: Ommited variable problem 
---

The Gaussian copula approach is a statistical tool that is employed to address endogeneity issues within regression models. Endogeneity refers to the problem in statistical analysis where an explanatory variable is correlated with the error term, potentially leading to biased and inconsistent estimates. Here’s an overview of how the Gaussian copula can be used in this context:

1. Identification and Correction: The Gaussian copula can help in identifying and correcting endogeneity problems in regression models, particularly those with an intercept​1​.

2. Instrumental Variables (IV) Alternative: It provides an alternative to the traditional instrumental variables (IV) approach, which is commonly used to deal with endogeneity. The Gaussian copula is particularly useful when appropriate instruments are not available​2​​3​.

3. Marketing Research: In non-experimental data, marketing researchers have started using the Gaussian copula to identify and correct endogeneity without relying on IVs. This approach has been demonstrated through simulation studies to be effective in certain regression contexts​4​.

4. Robustness: Theoretical and simulation evidence suggests that the Gaussian copula is robust against misspecification, particularly when compared with other symmetric copulas​5​.

However, the use of Gaussian copulas comes with its own assumptions and limitations. One significant assumption is that the dependency structure between the endogenous regressor and the error term is accurately described by a Gaussian copula, which is a non-testable assumption. Understanding the implications of this assumption, what causes its violation, and possible remedies is an area with limited clarity​1​. Additionally, while it has promising features, the limitations and overall usefulness of the Gaussian Copula approach, especially in management research contexts, are not yet fully understood​3​.

These insights suggest that while Gaussian copulas present a viable option for addressing endogeneity, especially when traditional instrumental variables are not applicable or available, researchers must proceed with caution due to the underlying assumptions and the current gaps in the understanding of its application and limitations.

{% highlight html %}

import pandas as pd
import numpy as np
import scipy.stats as stats
import statsmodels.api as sm
import matplotlib.pyplot as plt
import seaborn as sns
from statsmodels.graphics.gofplots import qqplot
from sklearn.utils import resample

# PLEASE CITE AS:
# Becker, J. M., Proksch, D., Ringle, C. M. (2021). Revisiting Gaussian copulas
# to handle endogenous regressors. Journal of the Academy of Marketing Science.
# forthcoming.

# Read file
# PLEASE DOWNLOAD THE FILE FROM THE REPOSITORY FIRST
dat = pd.read_csv("endogenous_data_simple.csv")
# The endogenous variable is variable P

# Summarize data
print(dat.describe())

# Show skewness and kurtosis
print(stats.skew(dat['P']))
print(stats.kurtosis(dat['P'], fisher=False))

# Conduct non-normality test for endogenous regressors
# We recommend using the Anderson-Darling and the Cramer-von Mises test
# and both report p-values and test statistics

# Anderson-Darling test
print(stats.anderson(dat['P'], dist='norm'))

# Cramer-von Mises test
# Note: Scipy does not have a direct implementation of the Cramer-von Mises test,
# so we would have to implement it ourselves or use an approximation/alternative test.

# Visually investigate the distribution of P
# Plot the density of P against the normal distribution
sns.kdeplot(dat['P'], shade=True, color="red")
plt.show()

# Check for normality of residuals
# For this we use the OLS regression function
model = sm.OLS(dat['Y'], sm.add_constant(dat['P'])).fit()
print(model.summary())

# Anderson-Darling test
print(stats.anderson(model.resid, dist='norm'))

# Cramer-von Mises test
# Same note as above regarding the implementation.

# Visually investigate the distribution of the residuals
# Plot the density of the residuals against the normal distribution
sns.kdeplot(model.resid, shade=True, color="red")
plt.show()

# Function to create Gaussian Copula
def create_copula(P):
    ecdf = sm.distributions.ECDF(P)
    U_p = ecdf(P)
    U_p = np.where(U_p == 0, 0.0000001, U_p)
    U_p = np.where(U_p == 1, 0.9999999, U_p)
    p_star = stats.norm.ppf(U_p)
    return p_star

dat['Copula'] = create_copula(dat['P'])

copula_model = sm.OLS(dat['Y'], sm.add_constant(dat[['P', 'Copula']])).fit()
print(copula_model.summary())

# Bootstrap significances
def boot_function_sig_copula(data, formula, R=500):
    boot_results = []
    for _ in range(R):
        sample = resample(data)
        model = sm.OLS(formula['Y'], sm.add_constant(formula[['P', 'Copula']])).fit()
        boot_results.append(model.params)
    boot_results = np.array(boot_results)
    return np.std(boot_results, axis=0)

boot_results = boot_function_sig_copula(dat, {'Y': dat['Y'], 'P': dat['P'], 'Copula': dat['Copula']})

# Standard errors
print(boot_results)

# t-values and p-values are calculated similarly using the stats.ttest_1samp function
# or by manually calculating them as in the R code

{% endhighlight %}


