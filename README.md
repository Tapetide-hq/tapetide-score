# Tapetide Score — Open Methodology

The **Tapetide Score** is a deterministic 0–100 rating for Indian (NSE/BSE) listed
stocks. It is computed from published financial data and prices by a fixed mathematical
model — **no language model, no analyst opinion, no discretionary override**.

This repository is the complete, authoritative specification of how that number is
produced: every pillar, every metric, every weight, every threshold, every gate, and the
statistical method we use to decide whether a factor earns its place at all.

**Live scores:** [tapetide.com](https://tapetide.com) ·
**Human-readable overview:** [tapetide.com/score/methodology](https://tapetide.com/score/methodology) ·
**Rankings:** [tapetide.com/score/leaderboard](https://tapetide.com/score/leaderboard)

---

## Why this is public

A rating you cannot audit is a rating you should not trust.

Most stock scores in the Indian market are black boxes: you are shown a number and asked
to take the vendor's word for it. We think that is backwards. If our score is any good,
publishing the method costs us nothing that matters and earns the only thing that does —
the ability for a sceptical reader to check our work and tell us we are wrong.

So this repository contains the **actual constants the production engine runs on**, not a
sanitised summary. If you find an error, a bias, or a factor we have weighted badly, open
an issue or a PR. See [CONTRIBUTING.md](CONTRIBUTING.md) — we are specifically looking
for people who will argue with the numbers.

### What is *not* here

Our **data pipeline**: which providers we buy or scrape from, how collection is
scheduled, and how it is stored. That is operational infrastructure, it is commercially
sensitive, and — importantly — **you do not need it to verify the methodology**. Every
input below is defined in terms of what it *is* (`ROCE 5-year average`, `six-month price
return`), not where we got it.

No code from the production engine is published here either. This is a specification
repository: the spec is the contract, and it is complete enough to reimplement.

---

## Design goals, in priority order

1. **Deterministic.** The same inputs always produce the same output, bit for bit. Every
   published score is stamped with its formula version, reference version, and an input
   hash so any number can be traced and reproduced.
2. **Robust to outliers.** Indian small-caps produce genuinely absurd ratio values
   (a cash-conversion ratio of 245×, a P/E of 4,000). The model uses median/MAD
   statistics and percentile winsorization throughout — **no means, no standard
   deviations** — so a single extreme value cannot move a peer group's calibration.
3. **Sector-relative.** A bank and an FMCG company are not comparable on the same
   metrics. Scoring happens against a peer distribution, using metric templates suited
   to the business type.
4. **Honest about missing data.** A stock with insufficient data is left **unscored**.
   We never zero-fill a missing pillar — a data gap must not masquerade as a bad result.
5. **Red flags can only subtract.** Governance risk is a multiplier and a set of hard
   caps. Strong fundamentals can never average away a serious red flag.

---

## The shape of the score

```
                    ┌─────────────────────────────────────────┐
  23 raw metrics →  │  robust z-score vs peer distribution    │  → 0–100 per metric
                    │  (winsorize → median/MAD → logistic)    │
                    └─────────────────────────────────────────┘
                                      ↓
                    ┌─────────────────────────────────────────┐
                    │  6 additive pillars (equal-mean metrics) │
                    │  Quality 25 · Valuation 20 · Growth 15   │
                    │  FinHealth 15 · Momentum 15 · Ownership 10│
                    └─────────────────────────────────────────┘
                                      ↓
                         weighted sum → value-trap damper
                                      ↓
                    ┌─────────────────────────────────────────┐
                    │  governance multiplier (≤ 1.0, floor .50)│
                    │  then hard caps (lowest wins)            │
                    └─────────────────────────────────────────┘
                                      ↓
                    EMA smoothing + hysteresis → re-clamp caps
                                      ↓
                         eligibility gate → publish 0–100
```

Every step, with exact constants: **[docs/METHODOLOGY.md](docs/METHODOLOGY.md)**

---

## Documentation

| Document | What it covers |
|---|---|
| **[docs/METHODOLOGY.md](docs/METHODOLOGY.md)** | The complete specification. Normalization math, all 23 metrics, pillar weights, min-metric bars, governance penalties and caps, smoothing, eligibility, confidence. Every constant. |
| **[docs/VALIDATION.md](docs/VALIDATION.md)** | How we test whether a factor actually predicts returns: rank IC, fit/holdout splits, residual IC for correlated metrics, and the four-check adoption bar. Includes measured results — **and the factors we rejected**. |
| **[docs/LIMITATIONS.md](docs/LIMITATIONS.md)** | What the score cannot do, where the model is weakest, and which claims we deliberately do not make. Read this before trusting any number. |
| **[docs/UPDATE-CYCLE.md](docs/UPDATE-CYCLE.md)** | Update frequency, trading-date semantics, partial-run protection, day-to-day stability, and reference versioning. |
| **[docs/CHANGELOG.md](docs/CHANGELOG.md)** | Every formula version, what changed, and why — including changes driven by measurement that went against our prior belief. |

---

## Quick reference

**Pillar weights** (additive, sum to 100; renormalized over available pillars)

| Pillar | Weight | Min. metrics | What it measures |
|---|---:|---:|---|
| Quality | 25 | 2 | Capital efficiency, margins, earnings quality, cash conversion |
| Valuation | 20 | 2 | Cheapness vs peers, on earnings/book/cash-flow/yield |
| Growth | 15 | 1 | Multi-year sales and profit trajectory |
| Financial Health | 15 | 1 | Solvency, leverage, interest coverage, distress risk |
| Momentum | 15 | 2 | Medium-term price trend |
| Ownership | 10 | 1 | Direction of promoter and institutional holding changes |
| *Governance risk* | *multiplier* | — | *Surveillance, credit, pledge, insider selling, audit — penalty only* |

**Core constants**

| Constant | Value | Role |
|---|---:|---|
| Winsorization | 2nd / 98th percentile | Outlier clamp before scoring |
| MAD scale factor | 1.4826 | Makes MAD comparable to a standard deviation |
| z clip | ±3.0 | Bounds an extreme after normalization |
| Logistic steepness `k` | 1.5 | Maps z → 0–100 |
| EMA α | 0.35 | Day-to-day smoothing weight on the new value |
| Hysteresis band | 2.0 points | Below this, the published score holds |
| Governance floor | 0.50 | A multiplier can never more than halve the score |
| Min. peer-cell observations | 30 | Below this, fall back to a coarser peer group |
| Eligibility | ≥4 of 6 pillars **and** ≥60% metric coverage | Else "Insufficient data" |
| Liquidity gate | ≥200 traded days in trailing 400 | Universe entry |

Roughly **4,500 stocks** are scored per run, once per trading day.

---

## Reading a score honestly

- **A score is not a recommendation.** It is a summary of past and present evidence, not
  a price forecast. A high score is not a buy signal; a low score is not a sell signal.
- **It is relative, not absolute.** 72 means "strong versus comparable companies", not
  "will go up".
- **The bottom of the range is the more reliable end.** The score is built to surface
  weakness and red flags; empirically the lowest deciles are its most consistent signal.
- **Check the confidence value.** It reflects how much data the stock actually had. A
  score built on thin coverage deserves less weight, and we publish that alongside it.

Tapetide is not a SEBI-registered research analyst or investment adviser. Nothing here or
on tapetide.com is investment advice.

---

## Licence

Documentation in this repository is released under
[CC BY 4.0](LICENSE) — use it, quote it, build on it, but attribute it.

You are welcome to implement this methodology yourself. If you publish a score derived
from it, please say so and link back, so readers can tell your calibration from ours.
