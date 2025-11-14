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
The **Synthetic Control Method (SCM)** builds a “synthetic twin” for a treated unit by combining donor units so that the pre-treatment trajectory matches as closely as possible. Formally, let $Y_{it}(0)$ be the outcome for unit $i$ at time $t$ without treatment. For treated unit $i=1$ and donors $i=2,\dots,J+1$, SCM finds weights $\mathbf{W}=(w_2,\dots,w_{J+1})'$ with $w_j \ge 0$ and $\sum_j w_j=1$ that minimize the pre-period mismatch:

$$
\min_{\mathbf{W}} \ \lVert X_1 - X_0 \mathbf{W} \rVert_V^2,
$$

and uses 

$$
\hat{Y}_{1t}(0)=\sum_j w_j^* Y_{jt}
$$ 

as the counterfactual. 
The effect at $t$ is 

$$
\widehat{\tau}_{1t}=Y_{1t}-\hat{Y}_{1t}(0)
$$


## 3.2 Introducing the Augmented Synthetic Control Method (ASCM)
SCM can struggle when the treated unit isn’t well-approximated by a convex mix of donors or when pre-periods are short. **ASCM** adds a lightweight prediction model to “correct” whatever imbalance remains after re-weighting:

$$
\hat{Y}_{1t}^{\mathrm{ASCM}}(0)
= \underbrace{\sum_j w_j^* Y_{jt}}_{\text{SCM}}
+ \Big[m(X_{1t}) - \sum_j w_j^* m(X_{jt})\Big],
$$

where $m(\cdot)$ is a simple outcome model trained **only on pre-treatment** data (we use ridge regression). This reduces bias without over-fitting noisy daily swings.

### 3.3 Using Covariates in ASCM (what we actually did)
To help the synthetic match the treated units **before** the intervention, we included a small set of stable, pre-treatment covariates per location. Concretely, we compute per-unit summaries **only on dates $t<t_0$**:
- Capacity
- Uptime
- Number of sessions/day
- Baseline off-peak share
- Pre-trend (slope) of off-peak share over time


In `augsynth`, we pass these via the **formula pipe** so the method balances on predictors in the pre period:
```r
outcome ~ treated | cov_capacity + cov_uptime +
                    cov_sess + cov_share_pre + cov_share_trend
```

**Why this matters**: our network is heterogeneous (hardware, locations, load), and daily noise is high. These pre-period summaries give the model a clearer picture of what “similar” means operationally, which improves the pre-fit and the quality of the counterfactual after $t_0$—without leaking post-treatment information.


## 4. Applying ASCM at EVCS

We tested ASCM on five locations that moved to a new time-based pricing structure for Pay-As-You-Go customers. The goal was simple: did the **share of kWh delivered off-peak** go up relative to what we would have expected without the change?

**Data we used (daily, station-level):** off-peak and peak kWh, total sessions, uptime, and available capacity. For each station, we built an outcome series (off-peak share) and computed **pre-treatment covariates**: mean capacity, mean uptime, mean sessions, baseline off-peak share, and a pre-trend. These covariates are fixed per unit (computed only on pre days), so there’s no leakage.

**How we fit the model:** instead of aggregating the five sites into one treated group, we ran ASCM **per treated location** (using all non-treated sites as donors) and then pooled effects across the five. In each single-unit fit we used:
- **Outcome:** daily off-peak share
- **Treatment timing:** the go-live date for pricing
- **Covariates (pre-period only):** the five summaries listed above
- **Augmentation:** ridge regression (keeps the prediction stable)
- **SCM weights:** enabled, with routine donor hygiene (drop donors missing pre dates or with zero pre variance)

In practice, the steps looked like:

1) Build the panel with dates aligned for the treated unit and all donors.  
2) Keep donors with **full** pre coverage and normal variability.  
3) Join **pre-computed covariates** to each unit.  
4) Fit `augsynth(outcome ~ treated | covariates, progfunc="Ridge", scm=TRUE)`.  
5) Extract the post-period effect per treated unit and then **pool** (equal-weight or kWh-weighted).  

This approach keeps the story simple—compare actual vs synthetic for each site—while the **covariates** make the synthetic more comparable to the treated station’s baseline. As a result, the post-treatment difference we see is less likely to be driven by quirks like persistent uptime differences or unusually high capacity at the treated site.

## 5. Implementation Details

From a technical perspective, the setup was fairly straightforward. We pulled daily station-level metrics directly from our Snowflake warehouse using R’s `DBI` and `dplyr` libraries, focusing on energy delivered during peak and off-peak hours, session counts, uptime, and capacity. Each station’s daily data became one row in a panel dataset, where the columns represented the key performance variables and the dates defined a continuous timeline.

A small portion of the raw dataset looked like this:


|date|location_id|treated|post|capacity|uptime|kwh_peak|kwh_offpeak|n_sessions|total_kwh
1|2025-04-01|50|0|0|200|98.54|479.7380|1056.7700|52|1536.5080
2|2025-04-01|51|0|0|200|83.33|18.8570|103.3530|14|122.2100
3|2025-04-01|53|0|0|250|99.97|285.1828|1007.6045|72|1292.7873
4|2025-04-01|54|0|0|200|98.62|405.0850|772.6820|37|1177.7670


Each record corresponds to one station on one calendar day, including whether it belonged to the treatment group (`treated = 1`), whether the intervention had already started (`post = 1`), and basic operational metrics such as uptime, energy delivered, and number of sessions.

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
