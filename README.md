# Customer Survival Modelling — FictiPay / bKash NSUCEC Datathon (Round 2)

A reference solution for the **time-to-churn survival** problem from the NSUCEC FictiPay
Datathon, finishing **7th of 30** teams in the onsite final.

The task is not *whether* a mobile-wallet customer churns, but *when*. Given three months
of transaction, balance, and KYC history (Jan–Mar 2024), we predict each customer's full
survival curve over the following 91 days and a single risk score for ranking.

This repository is written to be **read**, not just run. It documents the modelling choices,
the diagnostics that drove them, and — just as importantly — the directions that did not pay
off. If you are entering a churn or survival competition, the `docs/` folder is where the
transferable lessons live.

---

## Headline result

| Metric | Weight | Our score | Points earned |
|---|---|---|---|
| Concordance index (c-index) | 40 pts | **0.7623** | ~21 / 40 |
| Integrated Brier Score (IBS) | 30 pts | **0.0133** | ~22 / 30 |
| **Phase 1 (auto-scored)** | **70 pts** | | **~43 / 70** |
| Phase 2 (judged presentation) | 30 pts | | — |

Cross-validated (5-fold) out-of-fold estimate before submission: **c-index 0.7638**,
mean Brier 0.0167. The leaderboard and OOF numbers agreeing this closely was a deliberate
goal — see [`docs/lessons_learned.md`](docs/lessons_learned.md).


## What you'll learn here

- How to turn a right-censored survival target into a discrete-time classification problem.
- How to engineer behavioural features for churn from raw transaction logs at scale (~200M
  rows) using Polars streaming.
- How concordance and Brier score pull in different directions, and why on a low-event-rate
  problem the IBS is decided almost entirely by calibration at the final horizon.
- A worked, dependency-light implementation of the **IPCW Integrated Brier Score**, which is
  surprisingly hard to find done correctly.
- A map of which feature families carried signal and which did not.

---


## Problem summary

- **Churn definition:** a customer churns on the date marking 30 consecutive days without any
  transaction. Customers active at June 30, 2024 are right-censored.
- **Survival origin:** all durations measured from the March 31, 2024 observation cutoff.
- **Training labels:** `(ACCOUNT_ID, DURATION_DAYS, EVENT_FLAG)` derived from the full Jan–Jun
  history; participants receive only Jan–Mar inputs.
- **Submission:** five columns — `ACCOUNT_ID, RISK_SCORE, SURV_PROB_30D, SURV_PROB_60D,
  SURV_PROB_90D` — with survival probabilities monotone decreasing per row.

Full data schema and label construction are documented in [`data/README.md`](data/README.md).

---

## Reproducing the result

The competition dataset is **not** redistributed here (see licensing note below). With the
data attached locally, `python -m src.run` reproduces the submission deterministically
(`random_state=7`, 5-fold stratified CV). Expect the OOF c-index to land near 0.764 and the
mean horizon Brier near 0.0167.

---

## License & data

Code is released under the [MIT License](LICENSE). The competition data is owned by the
organisers and is **not** included in this repository — obtain it from the official Kaggle
competition page. Please confirm the competition's sharing terms before publishing any
derivative of the dataset itself.

This is a teaching repository built from a top-10 datathon finish. It is meant to make the
reasoning reusable, not to serve as a production survival-analysis library.
