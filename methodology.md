# Methodology

This document explains *why* the solution is built the way it is. The short version: we treat
time-to-churn as a stack of binary classification problems, one per scored horizon, and glue
their outputs into a valid survival curve. The longer version follows.

## 1. The problem and the metric

We predict, for each customer, a full survival curve over 91 days from the March 31 cutoff and
a single risk score. Phase 1 is auto-scored out of 70 points:

- **Concordance index (40 pts)** — does a higher risk score correspond to shorter survival?
  Uses the `RISK_SCORE` column only.
- **Integrated Brier Score (30 pts)** — are the predicted survival probabilities well
  calibrated across horizons? Uses `SURV_PROB_30D/60D/90D`.

Two facts about the data shape every decision:

1. **The event rate is ~5.2%**, and events are heavily back-loaded — only a few hundred
   customers churn within 30 days, versus ~31k by day 90.
2. **Every censored customer has `DURATION_DAYS == 91`.** There is no early censoring in this
   dataset. That means "alive at horizon *h*" is fully observed for any *h* ≤ 90: a customer
   either has an event with a known day, or is known to be alive the whole way. This is what
   makes the discrete-time reduction clean.

## 2. Why discrete-time classification instead of a hazard model

A standard route is a continuous-time model — Cox proportional hazards, a parametric Weibull
AFT, or a tree-based survival learner. We benchmarked these (see
[`lessons_learned.md`](lessons_learned.md)), but settled on a **discrete-time** formulation for
three reasons:

- **The metric only ever asks about three time points.** We are scored at 30, 60, and 90 days,
  not on the continuous curve. Estimating `F(h)` directly at exactly those horizons targets the
  metric without modelling parts of the curve that are never evaluated.
- **It turns the strongest tool on the problem.** Churn here is overwhelmingly a recency/
  frequency story, and gradient-boosted trees model that nonlinearity far better than a linear
  hazard model. Reducing to classification lets us use LightGBM directly.
- **Calibration is native.** A well-regularised classifier produces probabilities that are
  already close to calibrated, which is exactly what the IBS rewards — no separate baseline-
  hazard fitting step.

## 3. The construction

For each horizon `h ∈ {30, 60, 90}` define the cumulative-incidence target:

```
y_h = 1  if  EVENT_FLAG == 1 and DURATION_DAYS <= h
y_h = 0  otherwise
```

Train a LightGBM classifier per horizon to estimate `F(h) = P(T <= h)`. Then:

1. **Monotone repair.** Independent classifiers can produce `F(30) > F(60)`; we apply a
   cumulative maximum across horizons so `F(30) <= F(60) <= F(90)` for every row. This both
   respects the definition of a cumulative distribution and guarantees the required
   `SURV_PROB_30D >= SURV_PROB_60D >= SURV_PROB_90D`.
2. **Survival probabilities.** `S(h) = 1 - F(h)`.
3. **Risk score.** `RISK_SCORE = F(30) + F(60) + F(90)`. Summing the incidences gives a
   score that ranks customers who churn sooner *and* more certainly above those who don't —
   exactly the ordering the c-index measures. (The metric accepts any monotone risk score.)

## 4. Validation scheme

We use **5-fold stratified cross-validation**, stratifying each horizon's fold split on that
horizon's binary target so the rare early-churn positives are spread evenly. Within each fold
we early-stop on the validation slice. Test predictions are the mean across the five fold
models (bagging), which trims variance on the private split.

The out-of-fold predictions are scored with the same two metrics used by the leaderboard,
including a proper **IPCW Integrated Brier Score** (see [`../src/metrics.py`](../src/metrics.py)).
This is why the OOF and leaderboard numbers track each other closely — we never tune against
the public slice.

## 5. Results

| | OOF (5-fold) | Leaderboard |
|---|---|---|
| c-index | 0.7638 | 0.7623 |
| IBS | 0.0167 (mean horizon Brier) | 0.0133 |

The IBS is structurally easier to score well on than the c-index here: with a 5% event rate, a
curve that is merely well-calibrated near day 90 already sits far below the 0.05 null baseline.
The c-index is the harder 40 points, because ranking precision among the back-loaded events is
where the signal thins out.

## 6. What the model keys on

In order of importance, the dominant signals are **recency of last transaction**, **30-day and
7-day frequency**, **inactivity-gap structure** (max gap, near-threshold gap counts), and
**balance liveness** (days since the balance last moved). The deceleration ratios
(week-over-month, month-over-month frequency) add a second-order "slowing down" signal. The
network and amount-dynamics families contribute modestly. Feature-level hypotheses are
catalogued in [`features.md`](features.md).
