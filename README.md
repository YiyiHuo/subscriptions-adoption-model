# Predicting Subscriptions adoption among existing merchants

This project uses simulated data previously provided by a company: a sample of merchants with daily payment volume for three products (Subscriptions, Checkout, Payment Links) and a profile for each merchant (signup date, country, business size, industry). The goal was to build a quantitative approach that picks existing merchants for a sales or marketing campaign to adopt Subscriptions.

The full write-up is [`report/report.pdf`](report/report.pdf).

## What I predicted

For each merchant not yet using Subscriptions on a snapshot date $t$, the outcome is whether it starts using Subscriptions within the next 90 days:

$$Y_{it} = \mathbf{1}\{t \le A_i < t + 90\}, \qquad p_{it} = P(Y_{it} = 1 \mid X_{it}),$$

where $A_i$ is the merchant's first Subscriptions day and $X_{it}$ are its profile and its payment behavior over $[t - 90, t)$. The campaign list is the merchants with the highest estimated $p_{it}$ on the day after the data ends.

## Why this approach

- **Rolling snapshots instead of one row per merchant.** A plain "ever vs never adopted" logistic regression would compute covariates over the whole period, so they would include behavior after adoption (leakage) and lose how volume and activity change over time. Monthly snapshots compute covariates only from the past 90 days and stack the snapshots into one training set.
- **Logistic regression as the main model.** It gives odds ratios with confidence intervals (standard errors clustered by merchant), so each merchant on the list can be explained. I chose no L1 or L2 penalty because there are many more adoption events than coefficients and small classes are already merged.
- **Random forest as a second model.** It captures non-linear effects and interactions the logistic regression cannot, and it is stable with few adopters.
- **Equal-weight rank combination.** The two models pick partly different merchants (only 60% to 67% overlap in their top lists), so I average their rank percentiles. I use ranks because forest scores are not calibrated probabilities.
- **Alternatives considered and dropped:** L1 and L2 logistic regressions, other tree models, a GAM, and hazard models (discrete-time, Cox, time-varying Cox). The hazard models ranked about as well as the logistic regression but added assumptions; see Section 7 of the report.

## How I validated it

- **Temporal hold-out.** One test snapshot 90 days before the end of the data, after all training outcome windows, scored only once.
- **Time-series cross-validation** on the training snapshots to tune the forest (minimum leaf size, covariates tried per split).
- **Metrics:** AUC and lift (adoption rate among the top 1% and the top 100 merchants divided by the base rate of 1.0%).

| Model | Test AUC | Lift, top 1% | Lift, top 100 |
|---|---|---|---|
| Logistic regression | 0.73 | 6.4 | 7.0 |
| Random forest | 0.72 | 6.4 | 7.0 |
| Combined (equal weights) | 0.73 | 8.2 | 9.0 |

The most likely adopters are small merchants in service industries (Business services, Personal services, Education, Software) that already use Payment Links and are active but not every day, which fits a periodic billing pattern. The report also covers how to use the list (channel by merchant size, monthly rescoring, a randomized holdout to measure the campaign's incremental effect) and next steps (uplift modeling, value-based targeting).

## Repository

| File | Description |
|---|---|
| `report/report.pdf` | Write-up: data preparation, models, results, evaluation, recommendations, next steps |
| `data_cleaning.ipynb` | Reads the raw files; checks formats, duplicates and missing values; saves a clean merchant-day table |
| `covariates.ipynb` | Distribution of each variable and covariate decisions (merged classes, log volumes, activity covariates, tenure) |
| `logistic.ipynb` | Rolling-snapshot logistic regression: choice of horizon and minimum history, fit, odds ratios, test evaluation |
| `rf.ipynb` | Random forest with time-series CV, comparison with the logistic regression, combined model, final target list |
| `output/target_list_combined.csv` | Final target list: top 100 merchants by combined score |
| `output/target_list_time.csv`, `output/target_list_time_rf.csv` | Top 100 merchants by each model alone |

## Reproducing

The raw data are not included in this repository. With a `data/` folder next to the notebooks containing `merchants.xlsx`, `merchants.csv` and `payments.xlsx`, run `data_cleaning.ipynb`, `covariates.ipynb`, `logistic.ipynb` and `rf.ipynb` in that order. The intermediate `.pkl` files and figures are regenerated in `output/`.

Python 3.12 with pandas, NumPy, scikit-learn, statsmodels, SciPy, Matplotlib and openpyxl.
