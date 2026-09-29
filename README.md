# Hail Loss Projection — Central Europe

#Personal actuarial project: modeling and projecting insured losses 
#from hail events, with a focus on catastrophe natural (cat nat) 
#pricing and risk transfer.

#I am currently in 3rd year of Bachelor in Management and aiming for the Master's in Actuarial Science (MScAS) 
#at HEC Lausanne and building this project to develop hands-on 
#expertise in cat nat modeling and loss distributions.

---

## Level 1 — Data Exploration

#I started with raw NOAA Storm Events data (ncei.noaa.gov) covering 
#all severe weather events across the USA in 2023 (~75k events). 
#I filtered for hail only (~11.7k events), then isolated those with 
#declared insured losses (850 events with real damage figures).

#Loss amounts were stored as text strings ("100.00K", "1.00M") so 
#I wrote a conversion function to parse them into numeric values.

#Plotting the distribution immediately revealed a strongly right-skewed 
#pattern: most events cause small losses, but a handful cause 
#disproportionately large ones. This is the expected signature of 
#cat nat risk, where a small number of extreme events drive the bulk 
#of total losses.

#Worth noting: over half of hail events had no declared damage, either 
#because nothing happened or because it was never reported. A classic 
#reporting bias inherent to the dataset.

### Methodology
#Source: NOAA Storm Events Database 2023 (ncei.noaa.gov)
#Filtering: EVENT_TYPE == 'Hail', DAMAGE_USD > 0
#Conversion: custom parser for K/M/B suffixes into numeric USD
#Visualization: log10-scaled histogram to handle the wide value range

### Key Findings
#850 hail events with real insured losses out of 75,593 total weather events.
#Distribution strongly right-skewed: median ~30K USD, mean ~2.59M USD (86x ratio).
#Max single event: 400M USD.
#Over 57% of hail events reported zero damage (reporting bias).

---

## Level 2 — Loss Distribution Modeling

#With clean data in hand, I moved from description to modeling, fitting 
#statistical distributions to answer actuarial questions: what loss 
#amount is exceeded only 1% of the time? How much should an insurer 
#provision annually?

#Before choosing a distribution, I ran a Mean Excess Plot to explore 
#the tail structure. The curve forms a bell shape (rises then falls), 
#pointing toward a lognormal rather than a pure Pareto.

#I fitted three distributions: lognormal, generalized Pareto, and 
#Weibull. Then I compared them using AIC and a QQ-plot. The lognormal 
#wins on AIC, but the QQ-plot reveals it underestimates extreme events. 
#I kept the Pareto as a stress scenario for capital estimation.

#One notable result: the fitted Pareto has shape ξ=1.98, meaning its 
#theoretical mean is mathematically infinite. In practice this means 
#the empirical mean keeps growing as more data is added, and the risk 
#is unboundable in the classical sense. This is precisely the type 
#of tail risk that cat bonds are designed to absorb.

#For frequency, I used a Poisson distribution (λ=850 events/year) 
#and computed a pure premium = frequency × mean severity. The 
#theoretical pure premium (714M) is 3x lower than the empirical 
#estimate (2.2Bn), not because the model is wrong, but because 
##a single year of data with such a heavy tail is too unstable to 
#calibrate reliably. That instability is itself the main finding.

### Methodology
#Mean Excess Plot to guide distribution selection.
#Fitted distributions: Lognormal, Generalized Pareto, Weibull (scipy.stats).
#Model selection: AIC and QQ-plot.
#Risk metrics: VaR 95%, 99%, 99.5% via inverse CDF (.ppf).
#Frequency model: Poisson (λ = observed annual event count).
#Pure premium: λ × E[X] per distribution.

### Key Findings

| Model | VaR 99% | Pure Premium | AIC |
|---|---|---|---|
| Lognormal ✓ | 11.98M | 714M | 21,346 |
| Pareto (stress) | 69.04M | undefined (ξ ≥ 1) | 21,403 |
| Weibull | 8.18M | 440M | 21,574 |

#3x divergence between theoretical and empirical pure premium confirms 
#that a single year of data is insufficient to calibrate a heavy-tailed 
#distribution reliably.

#Pareto shape ξ=1.98 ≥ 1 implies a theoretically infinite mean, meaning 
#this risk is unboundable without contractual caps or ILS transfer.

---

## Stack
#Python · pandas · numpy · matplotlib · scipy
