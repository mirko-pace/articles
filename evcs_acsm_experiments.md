---
title: Augmented Synthetic Control for Continuous Field Experiments in EV Charging Networks
layout: default
---

## 1. Introduction: Experimentation in a Complex, Real-World EV Network
At EVCS, experimentation plays a crucial role in informing our decisions and understanding the value of various paths we can take in developing our product and enhancing the customer and driver experience. 

During the second half of 2025, we've been working on implementing time-based energy rates for our Pay-As-You-Go customers to meet better the demand for a competitive, driver-centric offer for EV drivers on the US West Coast.

The challenges while conducting experiments in this context are numerous. First of all, we would not implement randomized sampling strategies as we favor the consistency of the experience and fairness towards the EV drivers using our network.
As we examine quasi-experimental designs, such as geo-testing, the heterogeneous nature of our different charging sites - with varying equipment, surroundings, and utilization levels and patterns - poses another significant challenge to a solid and trustworthy approach.

In this article, we want to illustrate how we adopted the Augmented Synthetic Control methodology to measure the impact of many of our pricing experiments.

## 2. The Challenge: High Variability & Sparse Units
In a conservative approach, we generally roll out material changes in a limited subset of locations. Typically, a mix of 5 to 10 sites represents a good balance of the key traits of our entire network. 

However, our network has a wide range of different setups, contexts in which they operate, and resulting in different levels of utilization, from the single 50kW charger in a very dense, metropolitan area to very powerful 1MW sites along major interstate corridors. From the downtown mall or parking lot in L.A., to the scenic routes in Washington state and Oregon, the EVCS network (counting more than 300 sites and 1600 charging ports) brings high variability in key metrics when looked at each site.

Traditional difference-in-difference (diff-in-diff) approaches used in the past proved to be difficult to implement and prone to failure due to a variety of reasons beyond our direct control: it was generally challenging to find a subset of "control" sites with matching characteristics to the treatment group and stationary trends over time. It was even more challenging to account for external factors, such as equipment issues, weather, and vandalism (to name a few), when you have a very limited number of sites in your treatment and control groups.

## 3. Why Synthetic Control (and its Augmented Variant)
## 3.1 The Synthetic Control Method (SCM)

The **Synthetic Control Method (SCM)** provides a transparent, data-driven way to estimate the causal impact of an intervention when only one or a few units are treated and no randomized control exists.  
The core idea is to construct a *synthetic twin* for the treated unit—a weighted average of untreated (donor) units whose pre-intervention outcomes closely reproduce the treated unit’s pre-treatment trajectory.

Formally, let $Y_{it}(0)$ denote the outcome for unit \( i \) at time \( t \) had it never been treated, and \( Y_{it}(1) \) be the outcome under treatment.  
For \( i = 1 \) (the treated unit) and donors \( i = 2,\dots,J+1 \), SCM seeks a vector of non-negative weights

$$
\mathbf{W} = (w_2, w_3, \dots, w_{J+1})', \qquad 
w_j \ge 0,\ \sum_{j=2}^{J+1} w_j = 1,
$$

that minimizes the distance between pre-treatment outcomes of the treated unit and the weighted average of the donors:

$$
\min_{\mathbf{W}} \; \lVert X_1 - X_0 \mathbf{W} \rVert_V^2,
$$

where \( X_1 \) is the vector of pre-treatment outcomes (and possibly covariates) for the treated unit,  
\( X_0 \) is the corresponding matrix for donor units, and \( V \) is a diagonal matrix assigning relative importance to each predictor.  
The optimal \( \mathbf{W}^{*} \) defines the *synthetic control*.  
The counterfactual (no-treatment) trajectory for the treated unit is then

$$
\hat{Y}_{1t}(0) = \sum_{j=2}^{J+1} w_j^{*} Y_{jt}, \qquad t \ge t_0,
$$

and the estimated treatment effect at time \( t \) is

$$
\widehat{\tau}_{1t} = Y_{1t} - \hat{Y}_{1t}(0).
$$

In practice, this framework yields an interpretable composite of existing units that best mimics the treated site’s pre-intervention dynamics, avoiding the need for parametric modeling of complex temporal patterns.  
Within the **EVCS** network, SCM is especially appealing: each charging location has its own usage profile, and aggregating a small group of similar *donor* sites can reproduce the treated group’s baseline load behavior remarkably well.  
By visually comparing the treated trajectory against its synthetic counterpart, we can attribute post-intervention divergences—such as a sustained rise in off-peak charging share—to the experimental treatment rather than to random daily volatility or network-wide shocks.

## 3.2 Introducing the Augmented Synthetic Control Method (ASCM)

While the traditional Synthetic Control Method (SCM) performs well when the treated unit can be closely approximated by a convex combination of donors, its performance deteriorates when the pre-treatment fit is poor or when the number of pre-periods is limited.  
The **Augmented Synthetic Control Method (ASCM)**, proposed by Ben-Michael, Feller and Rothstein (2021), addresses this limitation by combining SCM weighting with a regularized outcome model that *augments* the counterfactual prediction.  
This augmentation corrects small pre-period imbalances and reduces bias, especially when the treated unit lies outside the convex hull of the donors.

Formally, ASCM adds a model-based adjustment term to the SCM estimator.  
Let $Y_{it}(0)$ be the untreated potential outcome as before, and let the set of donor weights $\mathbf{W}$ minimize the SCM objective.  
The pure SCM counterfactual for the treated unit at time $t$ is

$$
\hat{Y}_{1t}^{\mathrm{SCM}}(0)
  = \sum_{j=2}^{J+1} w_j^{*} Y_{jt}.
$$

ASCM augments this with a prediction from an outcome model $m(X_{it})$ (often a ridge regression trained on pre-treatment data):

$$
\hat{Y}_{1t}^{\mathrm{ASCM}}(0)
   = \hat{Y}_{1t}^{\mathrm{SCM}}(0)
     + \bigl[\, m(X_{1t}) - \sum_{j=2}^{J+1} w_j^{*} m(X_{jt}) \,\bigr].
$$

The estimated treatment effect is then

$$
\widehat{\tau}_{1t}^{\mathrm{ASCM}}
   = Y_{1t} - \hat{Y}_{1t}^{\mathrm{ASCM}}(0).
$$

Intuitively, the outcome model $m(\cdot)$ adjusts for systematic differences between the treated and donor units that persist after re-weighting.  
When $m(\cdot)$ is estimated using ridge regression, the regularization parameter $\lambda$ controls how strongly the model smooths idiosyncratic noise across time and donors, balancing bias and variance:

$$
m(X_{it}) = X_{it}' \boldsymbol{\beta}_{\lambda},
\qquad
\boldsymbol{\beta}_{\lambda}
  = \arg\min_{\beta}
    \sum_{i,t \le t_0} (Y_{it} - X_{it}'\beta)^2 + \lambda \|\beta\|^2.
$$

In practice, ASCM yields a bias-corrected version of SCM that performs reliably even when the pre-period fit is imperfect—a common situation in EVCS field experiments where sample sizes are small and daily station-level variability is high.  
By augmenting the synthetic control with a flexible model trained on pre-treatment data, we recover a stable and interpretable estimate of the causal effect of each intervention on charging behavior.

## 4. Applying ASCM at EVCS

To put the Augmented Synthetic Control Method into practice, we started with a simple and concrete experiment involving five locations. These sites were chosen because they represented a good mix of charger types, utilization levels, and geographic coverage—enough diversity to test the methodology, but still small enough to keep the analysis manageable and interpretable.

Each of the five sites received a new time-based pricing structure for Pay-As-You-Go customers, where off-peak energy rates were discounted to encourage drivers to charge outside the 4–9 PM evening window. We wanted to understand whether this change actually shifted charging behavior toward off-peak hours, or if daily and regional variability would overwhelm any measurable effect.

To measure this, we used daily station-level data from our reporting tables—covering kWh delivered during peak and off-peak hours, total sessions, uptime, and available capacity. For every day since the start of the experiment, we calculated the **share of kWh delivered during off-peak hours** at each site. The five treated locations were then aggregated into a single treated group, while all other active stations in the network served as potential donors for the synthetic control.

After aligning the data by date, we filtered out control stations that didn’t have full pre-treatment coverage. This ensured that the model had enough history to learn a consistent baseline trend for each donor. Once the data was cleaned and balanced, we applied the ASCM model using the `augsynth` package in R, with ridge regression as the augmentation method.

The ridge regression works by fitting a simple predictive model to the pre-treatment data while penalizing large coefficients. This appraoch helps smooth out noise and prevents the model from over-reacting to random day-to-day swings in charging behavior, giving ASCM a more stable and reliable baseline to compare against.

The goal was straightforward: compare how the treated group’s off-peak share evolved relative to its synthetic counterpart—an estimated “what-if” trajectory built from the rest of the network. This gave us a clear, data-driven way to isolate the impact of our pricing change from the natural ups and downs of everyday charger activity.

## 5. Implementation Details

From a technical perspective, the setup was fairly straightforward. We pulled daily station-level metrics directly from our Snowflake warehouse using R’s `DBI` and `dplyr` libraries, focusing on energy delivered during peak and off-peak hours, session counts, uptime, and capacity. Each station’s daily data became one row in a panel dataset, where the columns represented the key performance variables and the dates defined a continuous timeline.

A small portion of the raw dataset looked like this:

  date       location_id treated post capacity uptime  kwh_peak  kwh_offpeak n_sessions total_kwh
1 2025-04-01 50 0 0 200 98.54 479.7380 1056.7700 52 1536.5080
2 2025-04-01 51 0 0 200 83.33 18.8570 103.3530 14 122.2100
3 2025-04-01 53 0 0 250 99.97 285.1828 1007.6045 72 1292.7873
4 2025-04-01 54 0 0 200 98.62 405.0850 772.6820 37 1177.7670


Each record corresponds to one station on one calendar day, including whether it belonged to the treatment group (`treated = 1`), whether the intervention had already started (`post = 1`), and basic operational metrics such as uptime, energy delivered, and number of sessions.

After preparing the data, we created a simple aggregate for our five treated locations and kept all other active stations as potential donors. To make sure every control unit had a comparable baseline, we filtered out any stations that were missing dates before the intervention. This step is important because ASCM relies heavily on consistent pre-treatment trends to learn the right synthetic combination.

Once the panel was clean, we used the **`augsynth`** package in R to estimate the augmented synthetic control model. In practice, this meant specifying the treated group, the intervention date, and the outcome variable — the share of off-peak kWh. The ridge regression component acted as a stabilizer, reducing noise in the donor weights and helping the model find a smoother “synthetic twin” for the treated group.

Here’s a simplified version of the core R call we used:

```r
asyn <- augsynth(
  outcome ~ 1,
  unit   = unit,
  time   = time,
  t_int  = t_int_id,
  data   = as_data,
  treat  = "treated_group",
  progfunc = "Ridge",
  scm = TRUE
)
```

The output gave us two main pieces of information:
a reconstructed counterfactual series showing how the treated group would have evolved without the pricing change, and
the difference between the actual and synthetic trajectories — the estimated treatment effect over time.
To make interpretation easier, we visualized these results using simple line charts comparing the treated and synthetic trends, and another plot showing the daily treatment effect with confidence intervals. Together, these views gave us both a visual and statistical sense of how strong and consistent the off-peak shift really was.
