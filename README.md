# Hail Loss Projection with NOAA data

Personal actuarial project: modeling hail losses with a focus on catastrophe natural (cat nat) pricing and risk transfer.

I am currently in the 3rd year of a Bachelor in Management, aiming for the Master's in Actuarial Science (MScAS) at HEC Lausanne. I am building this project to develop hands-on expertise in cat nat modeling and loss distributions. Each level is a separate notebook, and I add the next one only once I understand the previous one.

## How to run

1. Download `StormEvents_details-ftp_v1.0_d2023_c20260323.csv.gz` from the NOAA NCEI Storm Events archive (https://www.ncei.noaa.gov/pub/data/swdi/stormevents/csvfiles/) and place it in a `data/` folder.
2. Install the dependencies: `pip install pandas numpy matplotlib scipy jupyterlab`
3. Open the notebooks in order with `jupyter lab`.

## Level 1: Data Exploration

I started with raw NOAA Storm Events data covering all severe weather events across the USA in 2023 (75,593 events). I filtered for hail only (11,761 events), then kept those with a positive property damage figure (850 events).

Damage amounts were stored as text ("100.00K", "1.00M"), so I wrote a conversion function to parse them into numeric values.

Plotting the distribution on a log scale shows a strongly right-skewed pattern: most events cause small losses, a handful cause very large ones. This is the typical signature of cat nat risk, where a few extreme events drive most of the total loss.

Damage amounts are not always filled in or positive. Out of the 75,593 events in the file, I kept the hail events and then looked at how many carry a usable damage figure:

| Step | Events | Share of hail events |
|---|---|---|
| All NOAA events (2023) | 75,593 | |
| Hail events | 11,761 | 100% |
| Damage field filled in (including 0) | 4,968 | 42.2% |
| Strictly positive damage | 850 | 7.2% |

The remaining 6,793 hail events have no damage figure at all. Either nothing was damaged or the damage was never reported, which is a reporting bias inherent to the dataset.


### Key Findings

- 850 hail events with a positive damage figure.
- Median loss about 30K USD, mean about 2.59M USD (ratio of 86).
- Largest single event: 400M USD.

## Level 2: Loss Distribution Modeling

With clean data, I moved from description to modeling. The questions: what loss is exceeded only 1% of the time, and how much should an insurer set aside each year?

I first plotted a Mean Excess Plot to explore the tail. The curve is globally increasing, which signals a heavy tail (lognormal or Pareto type). The drop on the far right is not interpretable, since very few observations lie above the highest thresholds. With only 850 points this plot cannot discriminate between candidate laws, so the choice relies on AIC and a QQ-plot.

I fitted three distributions to the severity of a single event: lognormal, generalized Pareto and Weibull. The lognormal has the lowest AIC. The QQ-plot shows a good fit on the body of the distribution but points drifting above the line in the upper tail, meaning the observed extremes are heavier than the lognormal predicts. I kept the Pareto as a stress scenario.

The fitted Pareto has a shape parameter of 1.98. A generalized Pareto has a finite mean only if the shape is below 1 and a finite variance only if it is below 0.5, so here both are theoretically infinite. This is the kind of tail risk that cat bonds and contractual caps are designed for.

For frequency I used a Poisson distribution with lambda = 850, the number of damaging hail events reported in the USA in 2023. The pure premium is frequency times mean severity. The lognormal gives 714M USD, while the empirical mean gives about 2.2 billion USD. The model and the data disagree by a factor of 3, which tells me a single year of data cannot calibrate a distribution this heavy-tailed.

### Methodology

- Mean Excess Plot for tail exploration.
- Distributions fitted with `scipy.stats` (lognormal, generalized Pareto, Weibull), loc fixed at 0.
- Model selection: AIC and QQ-plot.
- Risk metrics: VaR 95%, 99%, 99.5% on the severity of one event, via the inverse CDF (`.ppf`).
- Frequency: Poisson, lambda equal to the observed annual count.
- Pure premium: lambda times the theoretical mean severity.

Modelling assumption: NOAA property damage is used as a proxy for loss severity. Insurance coverage and policy conditions are not available, so the analysis covers reported economic damage rather than insured losses.

### Key Findings

| Model | VaR 99% (per event) | Pure premium | AIC |
|---|---|---|---|
| Lognormal (selected) | 11.98M | 714M | 21,346 |
| Pareto (stress) | 69.04M | undefined (infinite mean) | 21,403 |
| Weibull | 8.18M | 440M | 21,574 |

The lognormal 99.5% VaR is 23.1M, while the Pareto gives about 270M. A small move in the probability level changes the Pareto estimate by a factor of 4, which shows how sensitive tail estimates are to the chosen law.

## Limitations

- One year of data only, and one country.
- NOAA damage figures are rough estimates, not insurance claims.
- VaR is computed per event, not on the total annual loss, which is what solvency rules use.
- Lambda counts only events with a reported damage figure.


## Stack

Python, pandas, numpy, matplotlib, scipy
