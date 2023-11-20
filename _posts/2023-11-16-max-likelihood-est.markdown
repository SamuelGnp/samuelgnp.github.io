---
title: "Maximum Likelihood Estimation and implementation in a regression"
layout: post
date: 2023-11-16 15:51
image: /assets/images/markdown.jpg
headerImage: false
tag:
- markdown
- components
- extra
category: blog
author: samuelgnap
description: maximum likelihood estimation
---

## Maximum Likelihood Estimation and implementation in a regression 
Maximum Likelihood Estimation (MLE) is a cornerstone method in statistics, particularly within the frequentist framework. It's a powerful approach used in a wide range of applications, from simple linear regression to complex machine learning models like neural networks. MLE's versatility and robustness make it a preferred method in both academic and practical settings.
### Understanding MLE in Everyday Terms
Imagine you're a chef trying to perfect a recipe. You have certain ingredients (parameters) like salt, sugar, and flour, and you want to find the right mix that will make your dish (data) taste the best (most likely). Maximum Likelihood Estimation (MLE) is like this process of tweaking the ingredients until you find the best combination that makes your dish taste exactly how you want.
•	The "Taste Test": Think of the likelihood function in MLE as a taste test. Each time you change the amount of an ingredient, you're checking to see how likely it is that this new mix will give you the perfect taste.
•	Finding the Perfect Recipe: The goal of MLE is to adjust the ingredients until you find the recipe where the dish tastes the best (maximizing the likelihood). It's a bit like trial and error, but guided by the feedback from each taste test.
## Conceptual Framework of MLE
At its core, MLE seeks to find the set of parameters ($ θ $) of a statistical model that maximizes the likelihood function. This function measures the probability of observing the given data ($ X $) under specific model parameters.
•	Likelihood Function: The likelihood $ L(X;θ)L(X;θ) $ is conceptually the conditional probability of observing the data given specific values of the parameters. Mathematically, for a series of independent observations $ x_1,x_2,...,xn $ it is expressed as the joint probability:
$ P(x_1,x_1,...,x_n;θ) $
•	Log-Likelihood: Due to the multiplication of probabilities, which can lead to numerical underflow, we often use the natural logarithm of the likelihood. This transforms the product into a summation, resulting in the log-likelihood:
$ ∑_i=1 nlog⁡(P(x_i;θ)) $
•	Negative Log-Likelihood (NLL): Optimization routines commonly focus on minimization. Hence, the negative of the log-likelihood is used, forming the NLL function:
$ min−∑i=1nlog⁡(P(xi;θ)) $ 
MLE in Linear Regression
In linear regression, MLE can be applied to estimate the coefficients (ββ) and variance (σ2σ2) of the model. Assuming a normal distribution of errors, the log-likelihood model can be represented as:
$ ln⁡L(β,σ2)=−n2ln⁡(2π)−n2ln⁡(σ2)−12σ2∑i=1n(yi−Xiβ)2 $
•	Derivation: This formulation comes from the probability density function of the normal distribution. It accounts for the mean of the predicted values based on the linear combination of inputs and coefficients and the variance of the errors.
## MLE for Curve Fitting and Beyond
•	Generalization: MLE isn't limited to linear models. It can be adapted for curve fitting and complex models like artificial neural networks. The principle remains the same: find the parameter values that maximize the likelihood of observing the given data.
Least Squares and MLE
•	Least Squares: In the context of linear regression with normally distributed errors, least squares estimation is mathematically equivalent to MLE. The least squares method minimizes the sum of the squares of the residuals (differences between observed and predicted values).
•	Mathematical Formulation: The least squares objective function is:
$ minimize∑i=1n(yi−Xiβ)2 $
•	Analytical Solution: Unlike many other models where MLE requires numerical optimization, linear regression with least squares has an analytical solution:
$ β=(XTX)−1XTYβ=(XTX)^−1XTY $

## Implementing MLE and Least Squares From Scratch in Python

{% highlight python %}
import pandas as pd
import numpy as np
from scipy.optimize import minimize
from math import log, sqrt, pi

# data from https://www.kaggle.com/datasets/abrambeyer/openintro-possum/data
df = pd.read_csv("possum.csv")

df = df.dropna()
y = df['age'].values
X = df[['chest','belly']].values
#adding a constant
X = np.c_[np.ones((len(X), 1)), X]

def log_likelihood(X, y, beta):
    n = len(y)
    #finding the error 
    # since we are using 
    b1x = np.sum(X[:,1].dot(beta[0]))
    b2x = np.sum(X[:,2].dot(beta[1]))
    epsilon = y - (1+b1x+b2x)
    sigma = np.mean(epsilon)
    variance = sigma**2
    return -n/2 * log(2 * pi) - n/2 * log(variance) - (1 / (2 * variance)) * np.sum(epsilon ** 2)
    
beta_guess = (1,1)

# Minimizing the negative log-likelihood
result = minimize(lambda beta: -log_likelihood(X, y, beta), beta_guess)

beta_estimated = result.x
print(beta_estimated)

{% endhighlight %}

## Frequentist vs. Bayesian Approach 
In the realm of statistical modeling, the distinction between the Frequentist and Bayesian approaches is fundamental. Maximum Likelihood Estimation (MLE) is a hallmark of Frequentist statistics. In contrast, the Bayesian approach uses a different methodology, centered around Bayes' Theorem, for parameter estimation.
Frequentist Approach: Maximum Likelihood Estimation (MLE)
•	Core Principle: In the Frequentist paradigm, probabilities are interpreted as long-run frequencies. The MLE method focuses on finding parameter values that maximize the likelihood of observing the given data, without prior beliefs about these parameters.

•	MLE Characteristics:
o	Considers parameters as fixed but unknown quantities.
o	Estimates these parameters based solely on the observed data.
o	Emphasizes the likelihood of the data under different parameter values.
Bayesian Approach: Bayesian Estimation
•	Core Principle: Bayesian statistics incorporates prior beliefs or knowledge about parameters before observing the data. It then updates these beliefs in light of the observed data using Bayes' Theorem.
•	Bayes' Theorem: This theorem combines the prior distribution (pre-data beliefs about the parameters) with the likelihood of the observed data to form the posterior distribution (updated beliefs after considering the data).
•	Bayesian Characteristics:
o	Treats parameters as random variables with their own distributions.
o	Integrates prior information with the observed data.
o	Results in a posterior distribution for parameters, not a single point estimate.
•	Mathematical Formulation: Bayes' Theorem is stated as:
$ P(θ∣X)=P(X∣θ)P(θ)P(X) $
Here, $ P(θ∣X) $ is the posterior probability of the parameters ($ θ $) given the data ($ X $), $ P(X∣θ) $ is the likelihood, $ P(θ) $ is the prior probability of the parameters, and $ P(X) $ is the probability of the data.
Key Differences and Applications
•	Estimation: While MLE provides a point estimate for parameters, Bayesian estimation offers a probability distribution, reflecting uncertainty in parameter estimates.
•	Prior Information: Bayesian methods can incorporate prior knowledge or beliefs, which is especially useful when data is scarce or prior knowledge is robust.
•	Complexity: Bayesian methods can be computationally intensive, especially for complex models, due to the need to compute posterior distributions.
•	Interpretation: Bayesian inference provides a more intuitive probabilistic interpretation of model parameters, which can be particularly beneficial in decision-making contexts.

