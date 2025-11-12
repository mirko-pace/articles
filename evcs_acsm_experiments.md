---
title: Augmented Synthetic Control for Continuous Field Experiments in EV Charging Networks
---

<!-- Load MathJax on this page -->
<script type="text/javascript"
  src="https://cdn.jsdelivr.net/npm/mathjax@3/es5/tex-mml-chtml.js">
</script>


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

Formally, let \( Y_{it}(0) \) denote the outcome for unit \( i \) at time \( t \) had it never been treated, and \( Y_{it}(1) \) be the outcome under treatment.  
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

