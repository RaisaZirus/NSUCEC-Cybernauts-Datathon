# Feature catalogue

All 41 features grouped by family, with the hypothesis behind each. Features are derived from
three raw sources: the transaction log (send-side, keyed on `SRC_ACCOUNT`), the day-end balance
table, and KYC. See [`../src/features.py`](../src/features.py) for the exact definitions.

A note on null handling: a missing value almost always means "no activity," so it is filled
with the value that encodes inactivity — frequency and spend go to 0, recency and
days-since-balance-change go to their maximum of 91, drawdown goes to 1.0.

---

## BASE — recency, frequency, balance, tenure, gaps, deceleration (23)

The workhorse family. Churn is defined by inactivity, so direct measures of "how recently and
how much did this customer transact" carry most of the signal.

| Feature | Hypothesis |
|---|---|
| `recency_days` | Days since last transaction. The single strongest predictor — long silence precedes churn. |
| `freq_total` | Total Jan–Mar transactions. Engaged customers churn less. |
| `freq_30d` | March transaction count. Recent activity dominates the churn signal. |
| `freq_7d` | Last-week count. Sharpest recency window. |
| `freq_prev30` | February count. Baseline to compare recent activity against. |
| `total_amt` | Total spend. Scale of engagement. |
| `activity_concentration` | Share of activity in March vs lifetime. Recent-loading of activity. |
| `active_span` | Days between first and last transaction. Breadth of the relationship. |
| `bal_last` | Most recent day-end balance. Empty wallets churn. |
| `bal_max` | Peak balance. Account scale. |
| `bal_std_14d` | Balance volatility over the last 14 days. Movement = liveness. |
| `bal_changes_14d` | Number of days the balance moved in the last 14. Detects *inbound* activity the send-side txn features miss. |
| `days_since_bal_change` | Days since the balance last moved. A balance-side recency analogue. |
| `bal_drawdown` | `1 - last/max`. How far the wallet has been drained from its peak. |
| `tenure_days` | Account age at the cutoff. Older accounts are stickier. |
| `mean_gap_days` | Average inter-transaction gap. Cadence of use. |
| `n_gaps_ge20` | Count of 20+ day silences. Customer has flirted with the churn threshold. |
| `n_gaps_ge25` | Count of 25+ day silences. Even closer to the 30-day churn rule. |
| `true_max_gap` | Max of historical gaps and current recency. The longest silence ever observed, including the open-ended current one. |
| `gap_to_30_ratio` | `true_max_gap / 30`. How close the worst silence is to the churn threshold. |
| `decel_week` | `freq_7d / freq_30d`. Is activity concentrated or fading within the month? |
| `decel_month` | `freq_30d / freq_prev30`. Month-over-month slowdown. |
| `gap_vs_mean` | Current max gap relative to the customer's own typical gap. Personalised anomaly. |

## REG — regularity and rhythm (5)

How *evenly* a customer transacts, independent of how much.

| Feature | Hypothesis |
|---|---|
| `n_active_days` | Distinct days with activity. Habitual users have many. |
| `gap_cv` | Coefficient of variation of inter-transaction gaps. Erratic cadence signals instability. |
| `amt_cv` | Coefficient of variation of amounts. Spending consistency. |
| `spread_ratio` | Active days per transaction. Bursty vs. spread-out usage. |
| `mar_jan_ratio` | March-to-January activity ratio. Trajectory over the quarter. |

## NET — counterparty network structure + type mix (8)

A wallet embedded in a richer counterparty network is harder to abandon.

| Feature | Hypothesis |
|---|---|
| `n_unique_dst` | Distinct counterparties (fan-out). More relationships = stickier. |
| `cp_entropy` | Entropy of the counterparty distribution. Diversified vs. single-purpose use. |
| `txn_per_cp` | Transactions per counterparty. Depth of each relationship. |
| `top_cp_share` | Share going to the single biggest counterparty. Single-purpose accounts churn more easily. |
| `p2p_share` | Fraction of P2P transfers. Use pattern. |
| `merchant_share` | Fraction of merchant/bill payments. Embedded, recurring use is sticky. |
| `cashout_share` | Fraction of cash-outs. Draining behaviour can precede exit. |
| `n_trx_types` | Variety of transaction types used. Breadth of wallet use. |

## AMT — amount dynamics (5)

The trajectory and texture of payment sizes.

| Feature | Hypothesis |
|---|---|
| `amt_trend_mar` | March mean amount vs. earlier. Are payments growing or shrinking? |
| `big_txn_share` | Share of transactions above 3x the customer's mean. Presence of major payments. |
| `round_amt_share` | Share of round-number amounts. Behavioural texture / payment type proxy. |
| `log_amt_std` | Dispersion of log amounts. Consistency of payment size. |
| `amt_max` | Largest single payment. Account scale from a different angle than balance. |

---

## What carried the signal

The BASE family — recency, frequency, gap structure, and balance liveness — does the heavy
lifting. REG and AMT add modest, real lift; NET is the weakest family but contributes a little
through fan-out and merchant share. See [`lessons_learned.md`](lessons_learned.md) for the
saturation analysis and the directions that did not pay off.
