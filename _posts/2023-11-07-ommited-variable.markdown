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

## Exogenous Variables: The Lead Dancers

In the grand ballroom of econometric analysis, exogenous variables are akin to lead dancers. They command the performance, guiding the outcome with precision and influence, yet they remain impervious to the sway of other variables—the spectators of our model. Their role is critical, and their steps must be unerring, for they set the tempo for the dependent variable, our performance metric, without succumbing to the surrounding movements.

Consider the dance floor of a linear regression model:

$$
Y = \beta_0 + \beta_1 X_1 + \beta_2 X_2 + \ldots + \beta_k X_k + \epsilon
$$

Here, $$ Y $$ is our dependent variable, akin to the final performance score in a dance competition, influenced by a variety of factors, such as an individual's earnings. The exogenous variables $$ X_1,X_2,...,X_kX_1​,X_2​,...,X_k​ $$ represent our lead dancers, with the coefficients $$β_1,β_2,...,β_kβ_1​,β_2​,...,β_k​$$ quantifying their impact on $$ Y $$. The intercept  $$β_0​$$ is comparable to the baseline score when the competition begins, and the error term $$ ϵ $$ encapsulates all the unforeseen elements—the slight missteps, the unexpected rhythms—that the lead dancers' movements (our exogenous variables) do not account for.

1. It has a mean of zero, indicating that the residuals, on average, do not deviate in one direction or another.
2. It is homoscedastic, meaning the variance of the error term is constant across all levels of our explanatory variables.
3.  It is normally distributed, particularly important when we're making inferences about population parameters based on our sample, allowing us to apply various statistical tests.
    
The exogenous variables, as expert lead dancers, maintain their rhythm uncorrelated with the error term. This absence of correlation is crucial; it means that our exogenous variables do not contain any information about the value of the error term. Therefore, the residuals $$ ϵ $$ are independent of our explanatory variables $$ X $$. This independence is what allows our Ordinary Least Squares (OLS) estimators to be BLUE—Best Linear Unbiased Estimators—ensuring that our model's coefficients are estimated as accurately as possible given the data.

In sum, the exogenous variables carry the performance with their predefined influence, while the error term $$ ϵ $$, a conglomeration of all other random effects, dances in the background. Our goal is to minimize these random fluctuations by carefully selecting our lead dancers, ensuring that the variability they cannot explain remains as a background noise, not interfering with the integrity of the performance, our econometric model.
   

## Endogeneity: The Disruptive Dancer

Endogeneity is like a disruptive dancer, causing the lead performers to miss their cues. It arises when an explanatory variable, which should be exogenous, is correlated with the error term. This can occur due to omitted variables, measurement error, or simultaneity. Each of these sources of endogeneity can lead to biased and inconsistent estimates, distorting the true effects of the variables on our dependent variable.

### The Omitted Variable Foxtrot

In the world of econometrics and machine learning, omitting an influential variable from our models can throw off our entire performance. It's akin to forgetting a step in a dance routine, which can throw the entire performance into disarray.

For example, if we fail to include a variable like the level of education in our model that predicts earnings, our coefficients for other variables, like years of experience, may be overstated because they are picking up the effect of the omitted education variable.

### The Simultaneity Shuffle

Simultaneity enters the stage when variables such as supply and demand react to each other in real-time. If this mutual interaction isn't accounted for, models may incorrectly assign causation, leading to a tangled understanding of the relationships at play.

Demand function:

$$
Q_d = \alpha + \beta P + \epsilon_d
$$

Price function:

$$
P = \gamma + \delta Q_d + \epsilon_p
$$

Here, the price $$ P $$ and quantity demanded $$ Q_d $$​ are both dancers in this economic ballet, influencing each other in a continuous pas de deux. In an OLS framework, we would expect the error term $$ ϵ_d $$​ in the demand function to be uncorrelated with the price $$ P $$. Yet, due to the simultaneity of these variables, this is not the case:

$$
Cov(P, \epsilon_d) \neq 0
$$

This non-zero covariance indicates that OLS may naively interpret the positive correlation between high demand and high prices as a direct causal relationship, possibly leading to the misguided belief that higher prices could always induce higher demand. This overlooks the reality that demand might decrease in response to higher prices, an interplay that the OLS does not account for, mistaking the intricate dance for a simple march.

When simultaneity is at play, OLS estimates can become biased and inconsistent, failing to separate the simultaneous influence of price on demand and vice versa. The estimates do not converge to the true values because with each new sample, the dance changes, reflecting the market's ever-evolving rhythm rather than a fixed choreography. This results in a biased estimate of $$ β $$, the parameter showing the impact of price on demand, because OLS mistakenly captures the effect of demand on price as well, thus conflating two separate effects into one measure.

In this scenario, the OLS procedure cannot reliably discern the individual steps of each dancer —price and demand— and instead sees them as a single movement, leading to a distorted view of the economic dance.


## Choreographing Econometric Solutions and Machine Learning Ensembles

### Instrumental Variables Technique: The Routine

The instrumental variables (IV) approach introduces a new dancer to the floor—variables that can lead the endogenous variable without being affected by the error terms in the equation. These instruments must be correlated with the endogenous variable but uncorrelated with the error term, providing the leverage needed to estimate the true effect of the endogenous variable.

Finding a suitable instrument in practice is often akin to casting the perfect lead for a complex ballet performance — it is a significant challenge. The ideal instrument must be strongly correlated with the endogenous explanatory variable but must not be correlated with the error term in the equation, satisfying the conditions of relevance and exogeneity:

Instrument Validity:

Exogeneity:

$$
Cov(Instrument, \epsilon) = 0
$$

$$
Cov(Instrument, Endogenous Variable) \neq 0
$$

These conditions ensure that the instrument is not part of the error term's dance, allowing us to isolate the pure effect of the endogenous variable on the outcome. In reality, the conditions for a valid instrument are stringent and difficult to meet. Instruments that are weakly correlated with the endogenous variable can lead to biased estimates and weaken the power of statistical tests, a scenario known as weak instrument bias.

Furthermore, proving the exogeneity of the instrument is often complex, as it requires demonstrating that the instrument does not capture the effect of omitted variables. The plausibility of the instrument's exogeneity is sometimes based on theoretical assumptions or qualitative knowledge, which can be subject to debate.

Another potential issue is the overidentification problem. When there are more instruments than endogenous variables, the model can be overidentified, and we need to test whether the additional instruments are valid. Overidentification tests can be used to assess the validity of the instruments, but these tests have their limitations and can lack power.

### Adding Difference-in-Differences (DiD) to the Repertoire

DiD offers a way to measure the steps of policy changes over time, providing a natural experiment setting. By observing the dance before and after the policy intervention, and comparing it to a group not affected by the policy, DiD can effectively control for unobserved variables that might be causing both the intervention and the outcomes.

DiD Estimation:

$$
Y_{after} - Y_{before} = (Treatment_{after} - Treatment_{before}) - (Control_{after} - Control_{before})
$$

This equation allows us to subtract out the part of the variation in the outcome (YY) that is not attributable to the treatment, isolating the causal effect.
The DiD approach relies on comparing the changes in outcomes over time between a group that is exposed to a treatment (the treatment group) and a group that is not (the control group). The fundamental assumption here is that, in the absence of the treatment, the difference in outcomes between the two groups would have remained constant over time — an assumption known as the parallel trends assumption:

Parallel Trends Assumption:

$$
E[Y_{\text{treatment}, \text{pre}} - Y_{\text{control}, \text{pre}}] = E[Y_{\text{treatment}, \text{post}} - Y_{\text{control}, \text{post}}]
$$

This assumption is critical and often the hardest to verify, as it requires knowing the counterfactual — what would have happened in the absence of the treatment. While pre-treatment trends can be analyzed, the assumption cannot be tested directly for the post-treatment period. Violations of the parallel trends assumption can lead to biased estimates of the treatment effect.

DiD is also sensitive to the presence of other concurrent events or policy changes that could affect the outcome variable simultaneously with the treatment. These events can create additional differences between the treatment and control groups that are not due to the treatment itself, thereby confounding the estimated treatment effect.

### Watch Your Step: Challenges and Considerations

Choosing the right instruments or designing a DiD study requires a delicate touch. A misstep in instrument selection or in defining the treatment and control groups can lead to a misattribution of causation, akin to a misstep on the dance floor leading to a fall.










