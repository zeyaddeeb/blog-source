---
date: 2020-02-22
title: "Coronavirus (COVID-19) outbreak analysis"
markup: mmark
---

The [first news article I read this morning](https://www.bloomberg.com/news/articles/2020-02-22/coronavirus-may-be-the-disease-x-health-agency-warned-about) described Coronavirus (COVID-19) as "disease X";

> The World Health Organization cautioned years ago that a mysterious “disease X” could spark an international contagion. The new coronavirus, with its ability to quickly morph from mild to deadly, is emerging as a contender - Bloomberg

While these claims are seriously scary, I doubt they're 100% accurate. So often journalists inflate stories to sell news, and I felt like I could create an objective analysis to fact check the news. I've been working on some models surrounding COVID-19 outbreak, which I'll be sharing today.

John Hopkins: The Center for Systems Science and Engineering (CSSE), is kind enough to share data they collect from multiple sources. You can grab the data from this [link](https://gisanddata.maps.arcgis.com/apps/opsdashboard/index.html#/bda7594740fd40299423467b48e9ecf6) or see my [Github](https://github.com/zeyaddeeb/coronavirus).

Obviously, the first thing to analyze is the trend of cases overtime;

{{% plot "/plots/coronavirus-trend.html" %}}

This chart shows that confirmed cases are on the rise with a whopping total of 78,044 cases, with a recovery rate of 29.28% and a death rate of 3.12% as of Feb 22 2020.

There are two reasons why you might not get concerned about these stats.

1. The recovery rate is higher than the death rate.
2. The majority of confirmed cases are confined to China, which is notoriously good at controlling its population, thus making quarantine easier.

{{% plot "/plots/coronavirus-map.html" %}}

If you are an epidemiologist, you know that modeling of infectious diseases requires the use of Ordinary Differential Equations (ODEs) to estimate the long term impacts of the disease on a population.

I'll be covering the [simple compartmental model (SIR)](https://en.wikipedia.org/wiki/Compartmental_models_in_epidemiology#The_SIR_model)

The standard SIR equation is;

$$
S(t)+I(t)+R(t)=N
$$

which has three compartments that can be described as follows:

+ $$ S $$: the number of individuals susceptible to the disease.
+ $$ I $$: the number of individuals infected with the disease and capable of spreading the disease to those in the susceptible group.
+ $$ R $$: the number of individuals who have been infected and then removed from the infected group, either due to recovery or due to death. Individuals in this group are not capable of contracting the disease again or transmitting the infection to others.

Our free variables are $$t$$ for time and $$N$$ for the population.

There are multiple variations of the SIR model. The most basic version only accounts for the contact rate $$ \beta $$ and the recovery rate $$ \gamma $$, python time:

```python
import numpy as np
from scipy.integrate import odeint

N = 7_000_000_000  # Earth population
S = N - 1  # Everybody except for patient zero
I = 78_044  # Number of infected individuals (from data)
R = 22_858  # Number of recovered individuals (from data)
beta = 0.5  # infection rate (coin toss)
gamma = 0.14  # Recovery rate (from data)


def diff(sir, t):
    dsdt = -(beta * sir[0] * sir[1]) / N
    didt = (beta * sir[0] * sir[1]) / N - gamma * sir[1]
    drdt = gamma * sir[1]
    dsirdt = [dsdt, didt, drdt]
    return dsirdt


sir0 = (S, I, R)  # ODE initial conditions


t = np.linspace(0, 200)  # Let's solve over 200 days

sir = odeint(diff, sir0, t)

```

From this analysis we can conclude that, following this basic model, after 50 days cases of infection will start going down. According to the CDC, the [first case of coronavirus was reported Jan 30th 2020](https://www.cdc.gov/coronavirus/2019-ncov/summary.html).

Here is the plot from running the basic SIR model;

{{% plot "/plots/coronavirus-sir.html" %}}

Counting 50 days past Jan 30th, it seems we will reach peak infection rate around March 15th 2020. Normal disclaimer: by no means am I claiming this is an accurate epidemic modelling (or even that it is substantial). I personally have mathematical qualms with this model because not every one in the susceptible group will be infected and the transmission rate is a latent variable.

Another way to model this outbreak is from a bayesian approach where we look at it from a population density perspective.

We can say that infection rate per country $$c$$ as a set of Poisson random variables:

$$Pr(I_{c}(t) | \lambda_c(t)) = \text{Poisson}(\lambda_c(t)) $$

Where the country specific outbreak intensity at time $$t$$ is modeled as:

$$\lambda_a(t) = S_c(t-1) \frac{I(t-1)\mathbf{B}}{N_c} $$

where $$S_c(t-1)$$ is the number of susceptibles in a specific country $$c$$ in the previous time period, and $$\mathbf{B}$$ a matrix of transmission coefficients ([from Prem et al.](https://journals.plos.org/ploscompbiol/article?id=10.1371/journal.pcbi.1005697#sec020)), and $$N_c$$ an estimate of the population of in the country $$c$$.

This can be modeled in `pymc3` where the number of all the variables take the form of distribution and use MCMC sampling to find the actual rates. See measles reference for more detail on using the bayesian approach.

Remember to always be skeptical of news surrounding an outbreak, we are still in the midst of the battle. It is hard to see people lose loved ones to an outbreak, but with government intervention and adherence to basic hygiene and safety standards, the outbreak will hopefully be over soon enough.

## References

[https://www.cdc.gov/coronavirus/2019-ncov/summary.html](https://www.cdc.gov/coronavirus/2019-ncov/summary.html)
[https://lexparsimon.github.io/coronavirus/](https://lexparsimon.github.io/coronavirus/)
[https://github.com/fonnesbeck/mongolia_measles](https://github.com/fonnesbeck/mongolia_measles)
