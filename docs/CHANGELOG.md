# Methodology Changelog

Every formula version, what changed, and why. Changes driven by measurement that went
against our prior belief are marked **↩ prior belief overturned** — those are the entries
we think are most worth reading.

Version numbers are the `formula_version` stamped on every published score.

---

## v1.3.0 — Cash conversion added

**Added `cfo_to_np` to Quality** — operating cash flow ÷ net profit.

The strongest single factor we have measured on our own liquid universe: rank IC
**+0.091**, positive on **50 of 50** rebalance dates, holdout-confirmed (+0.104 fit /
+0.069 holdout). Near-orthogonal to everything already in the score (−0.116 against
earnings yield), so it adds information rather than re-weighting an existing signal.

Two gates, both load-bearing and both measured rather than assumed:

- **`net_profit > 0`** — the ratio inverts sign on a negative denominator, so a loss-maker
  with negative operating cash flow would score as *good* cash conversion. Restricting to
  profitable firms **raised** IC from +0.067 to +0.091, making this the better factor, not
  just the safer one.
- **`general` template only** — bank and NBFC operating cash flow is dominated by deposit
  and borrowing flows. Financials were **3× over-represented** in the top decile (32.3% vs
  ~11% baseline). Artifact, not quality.

Passed all four adoption checks ([VALIDATION.md](VALIDATION.md) §6), including the one that
matters most here: its decile **means** are non-monotone — decile 1 has the *highest* mean
forward return alongside the worst median — so a mean-return decile table would have made
the worst cohort look best. Medians are monotone. We publish medians.

Shipped **inert** until a new reference version was cut for it, per
[UPDATE-CYCLE.md](UPDATE-CYCLE.md).

**↩ prior belief overturned (about our own change):** we pre-simulated the coverage impact
as a ~3.5 percentage point *loss*, reasoning that the denominator would grow while the
metric was still unreferenced. That was wrong — the metric was absent from the inputs
entirely, so activation added numerator and denominator together and measured coverage went
**up**, 76.76% → 78.48%. Movement on activation: 4,271 of 4,484 stocks unchanged, 213
moved, range ±6, mean absolute 0.12. **A pre-run simulation is not a result; re-measure
after the run.**

---

## v1.2.0 — 12-month momentum removed

**Removed `mom_12m` from Momentum**, leaving `mom_6m` and `dist_200dma`.

**↩ prior belief overturned.** The plan said `dist_200dma` was the derived copy of
`mom_12m` and should be the one cut. Measurement said the opposite.

The two are 0.725 rank-correlated, and **pairwise correlation cannot tell you which of two
correlated metrics is the redundant one**. Residual IC can — rank each signal within the
other's deciles:

| Test | Fit | Holdout |
|---|---:|---:|
| `dist_200dma` holding `mom_12m` fixed | +0.027 | **+0.062** (46/54 dates positive) |
| `mom_12m` holding `dist_200dma` fixed | +0.013 | **−0.0005** (30/54 — coin flip) |

`mom_12m` retains nothing once `dist_200dma` is controlled for. Blends confirm it actively
diluted (holdout): `mom_6m + dist_200dma` **0.0825** > all three 0.0775 >
`mom_6m + mom_12m` 0.0701.

Verified before shipping: 4,444 stocks scored Momentum both before and after, **0 lost the
pillar, 0 lost eligibility**. Momentum's minimum bar is 2 with two metrics remaining — zero
slack, accepted knowingly.

Honest caveat recorded at the time: `dist_200dma` standalone swung 0.041 → 0.085 between
windows, and the holdout *is* a drawdown period where a 200DMA trend filter naturally
shines. We do not bet the pillar on it alone — but the **removal** is supported by both
windows.

*Reusable technique: for any two correlated candidates, within-decile residual IC decides
which to cut.*

---

## v1.1.0 — RSI removed

**Removed `rsi_14` from Momentum.**

Weakest of the four momentum signals by measured cross-sectional rank IC (**+0.039** versus
`mom_6m` **+0.074**), and absent from every institutional momentum construction we
compared against. Holdout-confirmed as weakest in both windows.

Blast radius measured before shipping: of 4,534 scored stocks, 4,444 kept ≥2 momentum
metrics; **90 fell below Momentum's minimum bar and lost the pillar** (66 of them
previously eligible); **0 breached the ≥4-of-6 eligibility gate**. A lost pillar is
**absent**, never zero-filled — zero-filling would read as worst-possible momentum.

`rsi_14` remains available as a user-facing screener filter. That is a different feature and
was not touched.

---

## v1.0.0 — Initial release

Six additive pillars (Quality 25, Valuation 20, Growth 15, Financial Health 15, Momentum
15, Ownership 10) plus a multiplicative governance overlay with hard caps. Robust
median/MAD normalization, logistic mapping, versioned sector-relative reference
distribution, EMA smoothing with hysteresis, eligibility gating.

Sanity check on the first full run: clean bell centred near 55; 4,425 stocks scored; 191
low-coverage names correctly suppressed. Cross-checked against an independent internal
quant model — ROCE +0.29, Piotroski +0.39, Spearman +0.23 — and the score's own bands
ordered correctly against that model's categories.

---

## Non-versioned corrections

Changes that fixed defects or documentation without altering the formula.

### Reference version resolved as-of the score date

**A live regression, and the reason [UPDATE-CYCLE.md](UPDATE-CYCLE.md) states this rule so
emphatically.** The reference version was derived arithmetically from the score date's year
(`ref_<year>`). When a new metric was activated under a newly cut version, the unattended
nightly job kept selecting the old year-named version — which had **no cell for that
metric** — so the metric was computed into the inputs for ~1,900 stocks every night and
then **silently discarded at scoring time**. One day of the correct baseline sat sandwiched
between two days of the old one, with no error and no log line anywhere.

Now resolved by looking up which version was **effective on or before the score date**.
Reading the table makes a newly cut version self-healing; resolving *as-of* rather than
"newest" means a backfill is never scored against a version that did not exist then, which
would be look-ahead bias in the calibration itself.

### Financial-sector template assignment

83 live companies were being scored on industrial metrics because template assignment
matched an explicit list of industry strings, so any Financial Services company whose
industry string was not listed fell through to `general`. Insurers reported **negative
operating margins** (−75.0 and −35.4 for two large life insurers) feeding directly into
Quality, and one showed EV/EBITDA of 239.8.

**Invisible from aggregates**: the mis-templated cohort reported the *highest* coverage and
confidence of any group (86.3% / 92.1 vs ~71% / ~78 for correctly-classified financials),
because correct classification NULLs inapplicable metrics while misclassification populates
them with garbage. Now sector-first, with the industry list only choosing *which* financial
template.

### Insider-selling classification fixed

The disposal predicate now matches `invoke` — a pledge invocation is a lender liquidating
collateral, economically a disposal. One company scored 0.000 on insider selling while 21 of
its 22 disclosures were collateral liquidations. Verified that `invoke` does **not** match
`Pledge Revoke`, which is routine and correctly excluded; the two differ by one letter and
matching the wrong one inverts the signal.

### Hard caps re-clamped after smoothing

EMA blends toward the previous published score, so a stock at 90 acquiring a GSM cap of 40
computed `0.35 × 40 + 0.65 × 90 ≈ 72` — **publishing above a "hard" cap** until the EMA
decayed. Caps are now re-applied after smoothing and after the hysteresis-hold branch.
Covered by a dedicated regression test.

### Dropped metrics reported instead of silently discarded

A metric with no reference cell was silently skipped while still counting against the
coverage denominator. Now reported per run with a share-of-universe figure, which is what
distinguishes a deliberately inert new metric (~100% of universe) from a reference rebuild
that dropped rows (a count on a metric that scored yesterday).

The sibling case is reported **separately**: a cell that exists but carries no usable scale.
Merging the two would tell an operator a cell is absent when it exists, sending them to
rebuild the wrong artifact.

### Partial-run publication guard

See [UPDATE-CYCLE.md](UPDATE-CYCLE.md). A run wrote 1,798 rows against a prior-day 4,440 and
rankings served 1,709 stocks instead of 4,256 — 2,547 silently missing, no error. Writer now
refuses to publish below 70% of the trailing median; readers rank only the latest complete
date. Both use one shared constant.

### Documentation corrections

Four public claims were found to be untrue of the code and were retracted rather than
quietly reworded:

| Claim | Status |
|---|---|
| "Point-in-time… no look-ahead bias… the score history you see is the score you would have seen" | **False.** Retracted; stated as a limitation instead. |
| "A 72 today means the same as a 72 last year" | **Overstated.** Narrowed to within a reference version. |
| Momentum is "volatility-adjusted" | **False.** No volatility term exists. Corrected to what the two metrics are. |
| Lookahead-free fundamental history reaches ~2015 | **Wrong.** Availability floor is ~2021; period depth is not availability depth. |

Also corrected: earlier copy promised to keep weights, thresholds and calibration
confidential. This repository supersedes that — the methodology is public.

---

## Things we tested and did NOT ship

Recorded because a rejected candidate is as informative as an accepted one, and because
these will otherwise be re-proposed.

| Candidate | Why rejected |
|---|---|
| **Low volatility / BAB pillar** | Published Indian research reports 1.08%/mo alpha and 29.1%/yr, beating momentum and value. On our universe: mean IC +0.031 with the sign flipping — **5 positive years, 5 negative**. A regime bet, not a factor. BAB's alpha comes from leverage and beta-neutralization, neither of which survives translation into an unlevered long-only ranking. |
| **IC-proportional pillar weights** | Looked like a decisive win in-sample (0.110 vs 0.060). Fit/holdout **inverted the ranking**: live weights 0.047 → **0.085 (best)**, IC-weighted 0.141 → 0.054. |
| **Dropping negative-IC pillars** (Quality, Financial Health) | Best in-sample (0.179), **worst on holdout (0.036)**. |
| **Institutional M−1 momentum skip** | Both NSE and MSCI mandate it. On our data it **cost** signal: 6m 0.074 → 0.068, 12m 0.047 → 0.038. |
| **Small-size factor** | Actively harmful, IC **−0.084**. |
| **"Fixing" accruals to match published India research** | Our data shows the US sign (−0.093), holdout-confirmed negative on 50/50 dates in both windows. We kept our measurement. |
| **Dedicated insurance template** | Only ~7 insurance stocks clear the liquidity gate, so it would calibrate median/MAD from 7 observations. Folded into `nbfc`, which already nulls what insurers cannot support. |
| **Per-template coverage denominator** | Would make a bank on 8-of-8 applicable metrics report 100% and outrank an industrial at 90% while carrying strictly less information. |
| **Hysteresis on band thresholds** | Makes classification stateful, breaking the determinism that is the whole defensibility argument. |
| **Bank-specific metrics** (NPA, CAR, PCR, NIM, CASA) | Not rejected — **not available**. Audited: those fields return zero rows across our data estate. Requires a new regulatory-filings source. |
