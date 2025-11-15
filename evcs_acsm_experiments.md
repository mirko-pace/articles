---
title: Augmented Synthetic Control for Continuous Field Experiments in EV Charging Networks
layout: default
---
# Augmented Synthetic Control for Continuous Field Experiments in EV Charging Networks


## 1. Introduction: Experimentation in a Complex, Real-World EV Network
At EVCS, experimentation plays a crucial role in informing our decisions and understanding the value of various paths we can take in developing our product and enhancing the customer and driver experience. 

During the second half of 2025, we've been working on implementing time-based energy rates for our Pay-As-You-Go customers to meet better the demand for a competitive, driver-centric offer for EV drivers on the US West Coast.

The challenges while conducting experiments in this context are numerous. First of all, we would not implement randomized sampling strategies as we favor the consistency of the experience and fairness towards the EV drivers using our network.
As we examine quasi-experimental designs, such as geo-testing, the heterogeneous nature of our different charging sites - with varying equipment, surroundings, and utilization levels and patterns - poses another significant challenge to a solid and trustworthy approach.

In this article, we want to illustrate how we adopted the Augmented Synthetic Control methodology to measure the impact of many of our pricing experiments.

## 2. The Challenge: High Variability & Sparse Units
In a conservative approach, we generally roll out material changes in a limited subset of locations. Typically, a mix of 5 to 10 sites represents a good balance of the key traits of our entire network. 

However, our network has a wide range of different setups, contexts in which they operate, and resulting in different levels of utilization, from the single 50kW charger in a very dense, metropolitan area to very powerful 1MW sites along major interstate corridors. From the downtown mall or parking lot in L.A., to the scenic routes in Washington state and Oregon, the EVCS network (counting more than 300 sites and 1600 charging ports) brings high variability in key metrics when looked at each site.

![Off-Peak share of kWh dispensed](/images/off_peak_kwh.jpg)


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


For every day and location, we then compute the **share of kWh delivered off-peak**:  
`share_offpeak = kwh_offpeak / (kwh_peak + kwh_offpeak)` (when total > 0). This is the metric we're going to use to test our hypothesis.

To help the synthetic control match each treated station **before** the change, we compute unit-level summaries using only pre-treatment days. We will use as covariates in our model:  
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
Taking a slightly different approach to `multi_synth`, we run one ASCM per treated locations where the donors are all non-treated stations, then pool effects across all five. This keeps things interpretable (actual vs synthetic for each site) and lets covariates improve the pre-fit in a targeted way.

Finally, we pass covariates in `augsynth` so the method balances on these predictors in the pre period, and we use Ridge Regression for stability in the augmentation process.

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
  cov_agg  = mean                
)
```

Before fitting, we made sure to have full pre-period coverage for donors, and drop any donor with zero pre-period variance in the outcome.

### Reading the results
From each single-unit fit we extract the daily ATT (treated minus synthetic). We then plot actual vs synthetic off-peak share for each location,
compute a pooled daily ATT across the five treated sites (equal-weight or kWh-weighted using pre-period average kWh), and summarize uncertainty using conformal intervals per unit; for a blended view we plot the pooled mean line and an envelope built from the per-unit conformal bands.

![ATT trend](/images/att_with_intervals.png)

In a nutshell: the covariates give ASCM a clearer picture of each treated location’s “baseline pattern,” the Ridge Regression dampens day-to-day noise, and the pooled view turns five small experiments into one consistent story about off-peak shifting and overall effect of time-of-use pricing.


## 6. Inference & Hypothesis Testing

### 6.1 State the hypothesis
Our goal is to determine whether the intervention lifted the **share of off-peak kWh** for the treated locations, relative to what would have happened without the change. Formally, we treat the **Average Treatment Effect on the Treated (ATT)** as our target: the daily ATT is the treated unit’s outcome minus its synthetic counterfactual, and the **average post-treatment ATT** is the mean of those daily effects after go-live. The null hypothesis asserts no improvement—on average, the post-treatment effect is zero. When we expect an increase, we phrase this as a one-sided test: under **H₀**, the average ATT is less than or equal to zero; under **H₁**, it is strictly positive.

### 6.2 Per-location inference (ASCM built-ins)
For each treated location we rely on ASCM’s **conformal inference**, which is designed for time-series outcomes and provides a valid test of the **average post-treatment effect** along with **pointwise confidence intervals** for the daily effects. In practice, we fit the model and then call `summary()` with conformal inference enabled and a one-sided statistic that matches our directional hypothesis:

```r
# Conformal inference (per fitted asyn object)
sumres <- summary(
  asyn,
  inf_type  = "conformal",
  stat_func = function(x) -sum(x)  # one-sided: positive average effect
)

# Readouts
avg_att <- sumres$avg_att          # average post ATT (treated - synthetic)
avg_ci  <- sumres$avg_ci           # 95% CI for average post ATT
pval    <- sumres$p_val            # joint p-value for the average effect

# Daily CIs (pointwise)
att_ci  <- as.data.frame(sumres$att)  # columns: Time, Estimate, Lower, Upper
```

We then interpret the output in the usual way. If the conformal p-value is below our threshold (e.g., 0.05) and the 95% interval for the average effect excludes zero, we conclude that the site exhibits a statistically detectable increase in off-peak share. If not, we retain the null for that location.

### 6.3 Pooled inference across treated locations
Because we fit one ASCM per treated location, each model delivers its own conformal pointwise intervals for the daily ATT. To present a single “blended” view we do two things:
- compute the **pooled mean ATT** across treated units for each date and plot it as the main line;
- build a **conservative envelope** by taking, at each date, the **minimum of the unit-level conformal lower bounds** and the **maximum of the unit-level conformal upper bounds**. 

```r
# 1) Pull per-unit conformal ATT bands
att_ci_all <- map_dfr(fits, function(f) {
  s <- summary(f$asyn, inf_type = "conformal")
  df <- as.data.frame(s$att)                
  df %>%
    rename(time = Time, est = Estimate, lo = lower_bound, hi = upper_bound) %>%
    inner_join(f$time_map, by = "time") %>% # adds 'date'
    transmute(unit_id = f$unit_id, date, est, lo, hi)
})

# 2) Pooled mean + pointwise envelope
pooled <- att_ci_all %>%
  group_by(date) %>%
  summarise(
    mean_att = mean(est, na.rm = TRUE),
    env_lo   = min(lo, na.rm = TRUE),
    env_hi   = max(hi, na.rm = TRUE),
    .groups  = "drop"
  )
```

### 6.4 Multiple views to avoid false comfort
Finally, we complement the formal tests with visual diagnostics. Daily ATT curves with conformal pointwise bands reveal how effects evolve and whether they persist; the average post ATT with its p-value and interval provides the primary decision anchor. We also translate percentage-point effects into operational terms (e.g., additional MWh moved off-peak per month) to connect statistical significance with business significance.


## References

- **Abadie, A., Diamond, A., & Hainmueller, J. (2010).** Synthetic Control Methods for Comparative Case Studies: Estimating the Effect of California’s Tobacco Control Program. *Journal of the American Statistical Association* https://doi.org/10.1198/jasa.2009.ap08746. :contentReference[oaicite:0]{index=0}

- **Ben-Michael, E., Feller, A., & Rothstein, J. (2021).** The Augmented Synthetic Control Method. *Journal of the American Statistical Association* (JASA); https://doi.org/10.1080/01621459.2021.1929245 :contentReference[oaicite:1]{index=1}

- **augsynth R package** — Implementation of ASCM with vignettes (single and staggered adoption) and examples. https://github.com/ebenmichael/augsynth :contentReference[oaicite:2]{index=2}

- **Chernozhukov, V., Wüthrich, K., & Zhu, Y. (2021/2022).** Exact and Robust Conformal Inference for Counterfactual and Synthetic Controls. (JASA article / arXiv preprint). https://doi.org/10.1080/01621459.2021.1920957 :contentReference[oaicite:3]{index=3}

- **Sun, L., Ben-Michael, E., & Feller, A. (2025).** Using Multiple Outcomes to Improve the Synthetic Control Method. https://arxiv.org/pdf/2311.16260. :contentReference[oaicite:4]{index=4}

- **Practical tutorial:** Conformal Inference for Synthetic Control (walk-through with code and intuition). https://matheusfacure.github.io/python-causality-handbook/Conformal-Inference-for-Synthetic-Control.html?utm_source=chatgpt.com :contentReference[oaicite:5]{index=5}

