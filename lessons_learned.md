# Lessons learned

The most reusable part of any competition write-up is the reasoning and the dead ends. This is
what we would tell a team starting the same problem from scratch.

## 1. Inspect the data structure before modelling anything

The single most useful early observation was that **every right-censored customer has
`DURATION_DAYS == 91`** — there is no early censoring. Once you see that, the survival problem
collapses into "is the customer alive at horizon *h*?", which is a fully-observed binary label
for any *h* ≤ 90. That insight is what justified the entire discrete-time approach and saved us
from over-engineering a continuous-time hazard model.

The lesson generalises: read the raw schema and the label-generating process directly. We also
found KYC columns (`GENDER`, `REGION`) that a quick summary would have missed. They turned out
flat on churn — but you only learn that by checking the source, not the summary.

## 2. The signal is recency-saturated

Churn is *defined* by 30 days of inactivity, so it is no surprise that recency and recent
frequency dominate. `recency_days`, `freq_30d`, and `freq_7d` together explain the large
majority of model gain. Everything else — gaps, balance liveness, deceleration, network, amount
dynamics — adds incremental lift on top of an already strong base.

Practical consequence: do not expect a clever feature to move the needle the way the obvious
ones do. Budget your time accordingly. We adopted a discipline of only keeping a new feature
family if it cleared a **+0.003 OOF c-index** improvement; this kept us from chasing noise.

## 3. The two metrics are not equally hard

With a ~5% event rate, the **IBS is the easier 30 points**. A survival curve that is merely
well-calibrated near the final horizon already sits well under the 0.05 null baseline, because
most customers genuinely do survive. The **c-index is the harder 40 points**, since ranking
precision among the back-loaded events is exactly where signal is scarce. If you are optimising,
the marginal point is cheaper on calibration but more valuable (and harder) on ranking.

A corollary: ~94% of events fall in the 61–90 day bucket, so IBS is decided almost entirely by
calibration at day 90, and short-horizon predictions barely move it.

## 4. What we tried that did *not* clear the bar

Keeping a map of negative results is as valuable as the positive ones — it tells the next team
where not to spend compute. On this problem, the following did not beat the discrete-time
LightGBM stack by a meaningful margin:

- **Linear Cox proportional hazards** as the scoring model. It cannot bend to the nonlinear
  recency relationship and ranks far worse than trees; useful for *interpretation* (hazard
  ratios) but not for the score.
- **Parametric Weibull AFT.** Clean acceleration-factor story for a presentation, but the
  parametric form is too rigid for the sharp recency effect.
- **Finer time bins** (modelling additional intermediate horizons). The metric only scores
  three points; extra horizons added complexity without improving them.
- **Second-order network features** (counterparty-of-counterparty structure). Expensive to
  compute and flat on churn once first-order fan-out was already in the model.
- **Diverse ensembling** across multiple learner types. The gains were within fold-to-fold
  noise and not worth the added fragility.

## 5. Don't overfit the public slice

The public leaderboard is a fraction of the test set. We deliberately kept our cross-validation
honest — proper stratification, a real IPCW Brier estimate, no tuning against the public score —
and as a result our OOF and leaderboard numbers agreed to within a few thousandths. Teams that
chase the public board's fifth decimal often pay for it on the private split. Trust your CV.

## 6. Reproducibility is a feature

Everything runs from one deterministic command with a fixed seed. When you can regenerate the
exact submission, you can refactor with confidence, debug a metric discrepancy quickly, and hand
the work to a teammate without a verbal ritual. That reliability mattered more to our final
standing than any single feature did.
