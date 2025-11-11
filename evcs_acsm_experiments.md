#Augmented Synthetic Control for Continuous Field Experiments in EV Charging Networks

##1. Introduction: Experimentation in a Complex, Real-World EV Network
At EVCS, experimentation plays a crucial role in informing our decisions and understanding the value of various paths we can take in developing our product and enhancing the customer and driver experience. 

During the second half of 2025, we've been working on implementing time-based energy rates for our Pay-As-You-Go customers to meet better the demand for a competitive, driver-centric offer for EV drivers on the US West Coast.

The challenges while conducting experiments in this context are numerous. First of all, we would not implement randomized sampling strategies as we favor the consistency of the experience and fairness towards the EV drivers using our network.
As we examine quasi-experimental designs, such as geo-testing, the heterogeneous nature of our different charging sites - with varying equipment, surroundings, and utilization levels and patterns - poses another significant challenge to a solid and trustworthy approach.

In this article, we want to illustrate how we adopted the Augmented Synthetic Control methodology to measure the impact of many of our pricing experiments.

##2. The Challenge: High Variability & Sparse Units
In a conservative approach, we generally roll out material changes in a limited subset of locations. Typically, a mix of 5 to 10 sites represents a good balance of the key traits of our entire network. 

However, our network has a wide range of different setups, contexts in which they operate, and resulting in different levels of utilization, from the single 50kW charger in a very dense, metropolitan area to very powerful 1MW sites along major interstate corridors. From the downtown mall or parking lot in L.A., to the scenic routes in Washington state and Oregon, the EVCS network (counting more than 300 sites and 1600 charging ports) brings high variability in key metrics when looked at each site.

Traditional difference-in-difference (diff-in-diff) approaches used in the past proved to be difficult to implement and prone to failure due to a variety of reasons beyond our direct control: it was generally challenging to find a subset of "control" sites with matching characteristics to the treatment group and stationary trends over time. It was even more challenging to account for external factors, such as equipment issues, weather, and vandalism (to name a few), when you have a very limited number of sites in your treatment and control groups.

3. Why Synthetic Control (and its Augmented Variant)
