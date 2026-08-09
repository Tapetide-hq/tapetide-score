# How We Validate Factors

A weight you cannot defend with a measurement is a guess. This document is how we decide
whether a metric earns a place in the Tapetide Score — including the cases where the
measurement contradicted our prior belief and we removed the factor.

Every number here was measured on our own universe of liquid Indian stocks. Where a result
disagrees with a published paper, we say so and keep our own result.

---

## 1. Rank IC, per date — never pooled correlation

The core measurement is the **cross-sectional Spearman rank information coefficient**: on
each rebalance date, rank every stock by the candidate metric, rank them by forward return,
and correlate those two rankings. Then look at the series of per-date ICs — mean, spread,
and how often the sign flips.

**Pooled correlation across all dates is actively misleading.** Measured on our data,
pooled Pearson correlation between 12-month momentum and 252-day forward return is
**−0.0112**, which reads as "momentum is dead in India". The same data as mean per-date
cross-sectional rank IC is **+0.047** — positive. The pooled figure is an outlier artifact
produced by mixing regimes and extreme values; it is not a finding.

So: **per-date rank IC, reported with its full per-year series.** Never a single pooled
number.

---

## 2. Report a mean IC only with range, per-year series, and independent N

Three retractions in one research session all traced back to skipping this. The discipline
now:

**(a) Print the date range the query actually returned, not the one you filtered for.** A
point-in-time earnings-yield test was first reported as `mean IC 0.0951, IR 2.012,
hit rate 97.6%` over "42 dates". The query had asked for a 2018 start; it silently returned
**2022-01 to 2025-06**. A truncated window looks identical to a full one in a summary
statistic.

**(b) Publish the per-year series, because a mean hides decay.** That same factor:

| Year | IC |
|---|---:|
| 2022 | 0.142 |
| 2023 | 0.095 |
| 2024 | 0.054 |
| 2025 H1 | 0.084 |

Monotonic decay, invisible in the headline. And extended to the earliest year the data
allows, the factor is **negative in 2021 (−0.058)** — the one year the quarterly test
structurally excluded. Always extend a factor to the earliest year available before
believing its sign.

**(c) Divide the span by the label horizon to get real N.** `IR 2.01` was an artifact of
**overlapping windows**. Monthly observations of a 252-day forward return overlap roughly
11 neighbours, so 42 monthly dates across 3.5 years is about **3–4 independent
observations**, not 42. Overlap crushes IC standard deviation and inflates information
ratio mechanically. A 97.6% "hit rate" on serially dependent draws is not evidence of
anything. Use Newey-West standard errors for anything load-bearing.

Shorter horizons buy independent observations at the cost of signal strength — 63-day IC
ran about half the 252-day IC throughout. That is a diagnostic, not a free upgrade.

**(d) When a result looks unusually clean, suspect the query before the market.** A 0-row
or 0-date result for a year range is a join failure until proven otherwise. Isolate stage
by stage — universe count, then visible count, then overlap, then with-label — rather than
concluding "the data is sparse".

---

## 3. A long-short factor premium is NOT a scoring pillar

This is the single most expensive lesson in this document, and it is why we do not have a
low-volatility pillar.

Published Indian research on Betting Against Beta reports **1.08% monthly four-factor
alpha and 29.1% annualized** (Jan 1998 – Jun 2013), beating both momentum and value, and
the NSE runs two live low-volatility indices on the same idea. On that evidence,
low-volatility was recommended internally as the highest-conviction new pillar.

Measuring plain annualized-volatility rank IC on our own liquid universe across the full
2016–2025 label window **killed it**:

| Year | IC | | Year | IC |
|---|---:|---|---|---:|
| 2016 | −0.139 | | 2022 | +0.129 |
| 2018 | +0.136 | | 2023 | −0.060 |
| 2020 | −0.127 | | 2024 | +0.213 |

Mean **+0.031**, and the sign flips — **5 positive years and 5 negative**. That is a regime
bet, not a factor. The recommendation was withdrawn one turn after it was made.

**Why it does not transfer:** BAB is a beta-neutral, *leveraged long-short* construct
(realized beta ~0.09, low-beta longs levered up, high-beta shorts levered down) whose alpha
comes substantially from the leverage and neutralization machinery. A 0–100 composite is an
**unlevered, long-only, cross-sectional ranking** — no shorting, no leverage, no
beta-neutralization. None of the machinery survives the translation.

Value (+15.3%/yr HML) and momentum (+21.9%/yr WML) headline numbers from the same research
are *also* long-short returns. They transfer only because they independently show positive
cross-sectional IC on our data.

**Rule: never quote a long-short premium as evidence for a pillar weight. Measure the
candidate's own per-date cross-sectional rank IC and require a stable sign across years.**

The same test killed the institutional M−1 momentum skip (skipping the most recent month),
which both NSE and MSCI mandate in their index construction. On our data it **cost** signal:
6-month IC fell 0.074 → 0.068, 12-month 0.047 → 0.038.

---

## 4. A factor IC is not a weight — fit/holdout or it does not ship

Having measured per-factor ICs, the obvious next step is to reweight the pillars in
proportion. We did exactly that, built three alternative weight vectors, scored them on the
same panel, and measured a convincing win: **0.110 versus the live 0.060**.

Splitting into **fit (2021–23)** and **holdout (2024–26)** reversed the ranking completely:

| Weight scheme | Fit IC | Holdout IC |
|---|---:|---:|
| **Live weights (25/20/15/15/15/10)** | 0.047 | **0.085 — best** |
| Equal weight | 0.049 | 0.076 |
| IC-weighted | 0.141 | 0.054 |
| Drop negative-IC pillars | 0.179 | **0.036 — worst** |

The full-sample ranking was **exactly inverted** on unseen data. The IC-weighted scheme
that looked best in-sample came fourth; deleting "underperforming" pillars was the single
worst thing we could have done.

**Consequences we accepted:**

- **Keep the live weights 25/20/15/15/15/10.** They won the holdout.
- **Do not delete Quality or Financial Health** for their negative standalone ICs (−0.007
  and −0.043). Deleting them *was* the worst variant.
- Live weights versus naive equal weight is nearly a tie, which honestly means **the
  weights carry little information**. We therefore anchor them on NSE/MSCI multi-factor
  precedent rather than on our own fitting — a position now supported by measurement rather
  than by deference.

**Any proposal to change pillar weights must include a fit/holdout split.** A full-sample
improvement is not evidence.

---

## 5. Residual IC decides between two correlated metrics

When two candidate metrics are correlated, pairwise correlation tells you they overlap. It
**cannot** tell you which one is the redundant copy. Residual IC can: rank each signal
*within the other's deciles*.

Worked case — `mom_12m` versus `dist_200dma`, rank correlation 0.725. The prior belief was
that `dist_200dma` was the derived copy of `mom_12m`. Measured, it was the reverse:

| Test | Fit | Holdout |
|---|---:|---:|
| `dist_200dma` holding `mom_12m` fixed | +0.027 | **+0.062** (46/54 dates positive) |
| `mom_12m` holding `dist_200dma` fixed | +0.013 | **−0.0005** (30/54 — coin flip) |

`mom_12m` retains *nothing* once `dist_200dma` is controlled for. Blends confirm it
actively dilutes (holdout): `mom_6m + dist_200dma` **0.0825** > all three 0.0775 >
`mom_6m + mom_12m` 0.0701.

So `mom_12m` was removed. Note the honest caveat: `dist_200dma` standalone swung
0.041 → 0.085 between windows, and the holdout *is* a drawdown period where a 200DMA trend
filter naturally shines. We do not bet the pillar on it alone — but the **removal of
`mom_12m`** is supported by both windows.

**Reusable: for any two correlated candidates, within-decile residual IC tells you which to
cut. Pairwise correlation alone cannot.**

---

## 6. The four-check adoption bar

Every check below was applied to `cfo_to_np`, the most recent metric added.

**Check 1 — Decile monotonicity on the MEDIAN, not the mean.**
`cfo_to_np`'s decile *means* are non-monotone: decile 1 has the **highest** mean forward
return (0.283) alongside the worst median. A mean-return decile table would make the worst
cohort look best. **Never publish a mean-return decile table.** Medians are monotone.

**Check 2 — Independence from existing factors.** `cfo_to_np` versus earnings yield:
**−0.116**. Genuinely new information, not a repackaging.

**Check 3 — Size and sector concentration.** This is what surfaced the bank problem:
financials were 3× over-represented in the top decile (32.3% versus ~11% baseline), because
bank operating cash flow is deposit and borrowing flow. Hence the `general`-only gate.

**Check 4 — Sector-neutralized IC.** +0.0884, still positive on 50 of 50 dates. Not a
disguised sector bet.

Only then does a metric ship — and even then it ships **inert** until a new reference
version is cut for it.

---

## 7. Interpret drawdown windows carefully

Liquid-universe **median** 252-day forward return by rebalance year:

| Year | Median fwd return |
|---|---:|
| 2016 | +25.5% |
| 2018 | −22.9% |
| 2020 | +56.9% |
| 2023 | +36.5% |
| 2024 | −12.3% |
| 2025 H1 | −7.9% |

Two things follow.

**"Every decile is negative" in 2024–26 is the market, not a broken score.** The p90 also
compresses (+140% → +40%), which mechanically depresses every IC measured in that window.

**Decile non-monotonicity is not composite-specific.** An inverted-U — worst outcomes at
*both* tails — appears identically in the independently validated `mom_6m` over the same
window, because extreme names at both ends fell hardest. Bottom deciles stay reliably
worst, which is precisely what a red-flag-oriented product needs.

The 2023 → 2024 swing (+36.5% to −12.3%) is also why a three-year fit window overfits: it
fits a bull run.

---

## 8. Measured factor ICs

Mean per-date cross-sectional rank IC against 252-day forward returns, liquid universe.
**Read these with sections 2, 3, and 4 in mind** — a single number is not a mandate.

| Factor | Mean IC | Verdict |
|---|---:|---|
| `cfo_to_np` (profit > 0) | **+0.091** | Adopted v1.3.0 — strongest measured, 50/50 dates positive |
| `dist_200dma` | +0.062 … +0.085 | In Momentum; strongest in holdout |
| `mom_6m` | +0.074 | In Momentum |
| Earnings yield (point-in-time) | +0.057 | In Valuation |
| `mom_12m` | +0.047 | **Removed v1.2.0** — zero residual IC vs `dist_200dma` |
| `rsi_14` | +0.039 | **Removed v1.1.0** — weakest momentum signal |
| Annualized volatility (low-vol) | +0.031 | **Rejected** — sign flips 5 of 10 years |
| ROCE / gross-profits-to-assets | regime-flipping | Retained; negative 2021–23, positive 2024–25 |
| Accruals | −0.093 | Red flag — US sign, *opposite* published India result |
| `size_small` | **−0.084** | **Rejected** — actively harmful |

Two entries deserve emphasis:

**Accruals show the US sign (−0.093), which is the opposite of the published Indian
finding.** Holdout-confirmed: −0.101 fit / −0.078 holdout, negative on 50 of 50 dates in
both windows. We did **not** "fix" this to match the paper. Our data says what it says.

**Small size is actively harmful (−0.084) on our universe.** We do not have a size factor,
and the leaderboard defaults to large-cap partly for this reason.

---

## Reproducing this work

You do not need our data pipeline to reproduce any of it. What you need is:

1. A monthly rebalance panel of liquid Indian stocks.
2. Forward returns at your chosen horizon (we use 21/63/126/252 trading days).
3. A delisting flag, so survivorship bias is addressable rather than ignored.
4. Point-in-time fundamentals if you are testing a fundamental factor — using
   latest-reported figures on a historical date embeds look-ahead bias. See
   [LIMITATIONS.md](LIMITATIONS.md) for the honest state of our own point-in-time coverage.

Then: per-date rank IC, per-year series, fit/holdout split, residual IC for correlated
pairs, and the four-check bar above.

If you run this and get a different answer than we did, **we want to hear about it** —
open an issue with your method and window. A reproducible disagreement is the most useful
contribution this repository can receive.
