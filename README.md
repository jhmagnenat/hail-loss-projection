# Hail Loss Projection with NOAA data

Personal actuarial project: modeling hail losses with a focus on catastrophe natural (cat nat) pricing and risk transfer.

I am currently in the 3rd year of a Bachelor in Management, aiming for the Master's in Actuarial Science (MScAS) at HEC Lausanne. I am building this project to develop hands-on expertise in cat nat modeling and loss distributions. Each level is a separate notebook, and I add the next one only once I understand the previous one.

## How to run

1. Download the NOAA NCEI Storm Events details files from the archive (https://www.ncei.noaa.gov/pub/data/swdi/stormevents/csvfiles/) and place them in a `data/` folder: `StormEvents_details-ftp_v1.0_d2023_c20260323.csv.gz` for Levels 1 and 2, and one file per year from 1996 to 2025 for Level 3 (30 files).
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

## Level 3: Multi-year Loss Modeling

At the end of Level 2 I concluded that the model and the data disagreed by a factor of 3 because I only had one year. Level 3 is where I tested that explanation, and it turned out to be wrong, or at least incomplete. I downloaded every year from 1996 to 2025 and left Level 2 untouched, so the mistakes I made there stay visible. In the Level 3 notebook each correction comes right after the problem that made me notice it.

### Rethinking what an event is

The first thing I questioned was my own unit of analysis. In Level 2 I counted one NOAA row as one hail event, but while looking at the biggest rows I realised a row is a county segment: one storm is spread over many rows, tied together by an `EPISODE_ID`. My lambda of 850 for 2023 was a count of rows. Counted by storm, 2023 has 228. So the frequency I used in Level 2 was wrong by a factor of almost 4, before any statistics.

I rebuilt the dataset with one line per storm, whose loss is the sum of its rows: 105,447 hail storms over 1996-2025. Checking the grouping year by year, I found that 1996 is not grouped at all (1.008 rows per episode), so I dropped it. The number of rows per episode also jumps around 2007 (from about 2.4 to 4.4). I do not know why, and I did not manage to find out.

I also learned that only 8% to 15% of storms carry a damage figure, depending on the year. So the frequency I model is the number of storms *with a reported damage*, not the number of hail storms. This is a limit of the data that I cannot remove.

### Choosing the years

The yearly count of storms with damage falls from about 500 to 260 until 2014, then stays between 146 and 257. I compared the variance/mean ratio of the frequency for windows ending in 2025:

| Window | Years | Variance / mean |
|---|---|---|
| 1996-2025 | 30 | 91.8 |
| 2007-2025 | 19 | 42.7 |
| 2015-2025 | 11 | 8.4 |
| 2018-2025 | 8 | 4.9 |

I hesitated between 2015 and 2018. 2018 is more homogeneous, but 2015-2025 gives 11 years and 2,140 storms to estimate the severity, so I kept it. I did not test the other choice on the final result, so this stays a judgment call.

A Poisson needs variance equal to the mean. With a ratio of 8.4 (lambda = 194.5 storms per year), it clearly does not fit: the busiest year, 2017 with 257 storms, has a probability of 1.1e-5 under the Poisson and 0.072 under the negative binomial that I used instead.

### The same severity models, and the same gap

I repeated the Level 2 work on the 2,140 storms: lognormal, generalized Pareto and Weibull. Losses go from 10 USD to 2.3 billion USD, and the largest storm alone weighs 20% of the total over the window (10 storms weigh 75.8%).

| Model (whole sample) | VaR 99% (per storm) | Pure premium | AIC |
|---|---|---|---|
| Lognormal | 19.25M | 305M | 53,084 |
| Pareto | 261.1M | undefined (infinite mean) | 53,201 |
| Weibull | 12.56M | 142M | 53,669 |

The lognormal wins on AIC again, but its pure premium is still 3.4 times below the observed 1.05 billion USD per year, against about 3.1 in Level 2. Adding 10 more years did not close the gap. That was the moment I understood that the problem was not the sample size but the way I modeled the tail. The Pareto I fitted on the whole sample is the second mistake from Level 2: a generalized Pareto describes the excesses over a threshold, not the full distribution of losses.

### Fitting the tail properly

I fitted the generalized Pareto only on the losses above a threshold u (peaks over threshold):

| Threshold | u (USD) | Excesses | Shape |
|---|---|---|---|
| 80% quantile | 195,000 | 426 | 1.77 |
| 90% quantile (kept) | 810,800 | 214 | 1.87 |
| 95% quantile | 2.5M | 106 | 1.66 |
| 97.5% quantile | 10M | 53 | 1.60 |

I kept the 90% quantile because the shape barely moves around it and 214 points are enough to estimate it. What surprised me is that the shape stays above 1 for every threshold, and a bootstrap gives [1.55, 2.19] as a 95% interval. A shape above 1 means an infinite mean. So fitting on the whole sample was not the only reason for the Level 2 Pareto result: the tail really is that heavy in this data, or at least the data do not allow me to say otherwise. On the same 214 excesses the Pareto also has the lowest AIC, though only 7.5 points ahead of the lognormal. On the probability-of-exceedance plot, the Pareto follows the data well up to about 50M USD of excess, but the four or five largest storms fall below the curve. That is compatible with a bounded tail, but five points are not enough to conclude.

### What the model says, and how much I trust it

An infinite mean means no pure premium, so I had to make a choice. I built a spliced model: the observed storms below u as the body, and above u either a Pareto capped at a maximum loss C per storm, or a lognormal fitted on the excesses.

| Tail model | Pure premium | Observed / model |
|---|---|---|
| Lognormal on excesses | 753M | 1.39 |
| Pareto, cap 2.5 billion USD | 1.28 billion | 0.82 |
| Pareto, cap 5 billion USD | 1.77 billion | 0.59 |
| Pareto, cap 10 billion USD | 2.44 billion | 0.43 |

The ratio now goes from 3.4 to below 1, but I do not read this as a success. The cap is an assumption that the data cannot make for me: 2.5 billion USD is simply the first value I tested above the largest observed loss. The threshold matters much less (between 1.12 and 1.28 billion for the 80% to 95% quantiles). So my honest result is a range, not a number.

The VaR of one storm is more stable, because a quantile stays finite when the mean does not:

| VaR per storm | Pareto tail | Lognormal tail | Lognormal (whole sample) |
|---|---|---|---|
| 95% | 2.56M | 3.08M | 2.57M |
| 99% | 49.2M | 47.8M | 19.3M |
| 99.5% | 179.6M | 112.0M | 40.2M |

At 99%, the two tails agree (about 48-49M) and the whole-sample lognormal I used in Level 2 underestimated it by a factor of 2.5. At 99.5% they diverge, since only about ten storms lie above that level.

### What I take from this level

My main lesson is that more data did not fix the model: understanding what one row represents did more than adding 29 years. I also learned that with a tail this heavy, the average observed over 11 years depends on a handful of storms, so it is a weak target, and a precise-looking pure premium would be misleading.

### Methodology

- Storm-level events built from `EPISODE_ID`, with the loss of a storm equal to the sum of its rows.
- Calibration window chosen from the variance/mean ratio of the yearly frequency.
- Frequency: negative binomial (method of moments), checked against the Poisson.
- Severity: lognormal, generalized Pareto and Weibull on the whole sample (AIC, QQ-plot), then peaks over threshold with a generalized Pareto fitted on the excesses.
- Threshold choice from candidate quantiles and the stability of the shape; bootstrap (1,000 resamples) for the uncertainty of the shape.
- Spliced severity: empirical body, tail either a capped generalized Pareto or a lognormal; VaR from the quantile of the tail.

### Key Findings

- The Level 2 gap was not due to the single year alone: with 11 years it is still 3.4 for the lognormal.
- The tail is very heavy: the Pareto shape is about 1.9 with a 95% interval above 1.
- The pure premium ranges from 0.75 to 1.28 billion USD for the two most prudent tail models (observed mean: 1.05 billion), and up to 2.44 billion with a higher cap.
- The VaR 99% per storm is about 48-49M with two different tails, versus 19.3M for the whole-sample lognormal.

## Limitations

- NOAA damage figures are rough estimates, not insurance claims, and amounts are often rounded.
- The frequency counts only storms with a reported damage figure, and about 8-15% of storms have one. I did not model the storms without a figure.
- 1996 is excluded and the grouping of rows into episodes changes around 2007; I have not identified the cause.
- The calibration window (2015-2025) is a trade-off between homogeneity and sample size.
- The mean observed over 11 years is dominated by a few storms, so it is not a reliable target.
- The pure premium depends strongly on the maximum loss assumed in the tail.
- VaR is still computed per storm, not on the total annual loss, which is what solvency rules use. Aggregating frequency and severity is the next step.
- One country only.

## Stack

Python, pandas, numpy, matplotlib, scipy
