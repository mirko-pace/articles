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

| date       | location_id | treated | post | capacity | uptime | kwh_peak | kwh_offpeak | n_sessions | total_kwh |
| ---------- | ----------- | ------- | ---- | -------- | ------ | -------- | ----------- | ---------- | --------- |
| 2025-04-01 | 50          | 0       | 0    | 200      | 98.54  | 479.7380 | 1056.7700   | 52         | 1536.5080 |
| 2025-04-01 | 51          | 0       | 0    | 200      | 83.33  | 18.8570  | 103.3530    | 14         | 122.2100  |
| 2025-04-01 | 53          | 0       | 0    | 250      | 99.97  | 285.1828 | 1007.6045   | 72         | 1292.7873 |
| 2025-04-01 | 54          | 0       | 0    | 200      | 98.62  | 405.0850 | 772.6820    | 37         | 1177.7670 |


**Outcome.** For every day and station, we compute the **share of kWh delivered off-peak**:  
`share_offpeak = kwh_offpeak / (kwh_peak + kwh_offpeak)` (when total > 0).

**Covariates (pre-period only).** To help the synthetic control match each treated station **before** the change, we compute unit-level summaries **using only pre-treatment days**:  
- capacity  
- uptime  
- number of sessions per day  
- baseline off-peak share (mean of `share_offpeak`)  
- a simple pre-trend (slope of `share_offpeak` over time)

```r
# Pre-treatment covariates per station (no leakage)
pre_cov <- df %>%
  dplyr::filter(date < t_start) %>%
  dplyr::group_by(location_id) %>%
  dplyr::summarise(
    cov_capacity_mean = mean(capacity, na.rm = TRUE),
    cov_uptime_mean   = mean(uptime,   na.rm = TRUE),
    cov_sess_mean     = mean(n_sessions, na.rm = TRUE),
    cov_share_pre     = mean(share_offpeak, na.rm = TRUE),
    cov_share_trend   = {
      tt <- as.numeric(date); yy <- share_offpeak
      if (sum(is.finite(tt) & is.finite(yy)) >= 5) as.numeric(coef(lm(yy ~ tt))[2]) else NA_real_
    },
    .groups = "drop"
  ) %>%
  tidyr::drop_na()
```
**Fit strategy:** Instead of collapsing the five locations into one treated group, we run one ASCM per treated station (donors = all non-treated stations), then pool effects across the five. This keeps things interpretable (actual vs synthetic for each site) and lets covariates improve the pre-fit in a targeted way.

**Model call (per treated location):** We pass covariates through the pipe in augsynth so the method balances on these predictors in the pre period, and we keep ridge augmentation for stability.

```r
asyn <- augsynth(
  outcome ~ treated | cov_capacity_mean + cov_uptime_mean + cov_sess_mean + cov_share_pre + cov_share_trend,
  unit     = unit,
  time     = time,
  t_int    = t_int_id,
  data     = as_data_with_covs,  # panel joined with pre_cov by unit
  progfunc = "Ridge",
  scm      = TRUE,
  fixedeff = FALSE,
  cov_agg  = mean                # harmless since covariates are constant per unit
)
```

Before fitting, we (a) align dates, (b) require full pre-period coverage for donors, and (c) drop any donor with zero pre-period variance in the outcome (these break the QP step and don’t help matching anyway).

Pooling & reading results. From each single-unit fit we extract the daily ATT (treated minus synthetic). We then:
plot actual vs synthetic off-peak share for each station,
compute a pooled daily ATT across the five treated sites (equal-weight or kWh-weighted using pre-period average kWh), and
summarize the average post-treatment ATT with uncertainty (e.g., jackknife across the five units, plus conformal pointwise CIs from summary(asyn) for each unit).

Net effect: the covariates give ASCM a clearer picture of each treated location’s “baseline pattern,” the ridge piece dampens day-to-day noise, and the pooled view turns five small experiments into one consistent story about off-peak shifting and overall effect of time-of-use pricing.

