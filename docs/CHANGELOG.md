# Methodology Changelog

Every formula version, what changed, and why. Changes driven by measurement that went
against our prior belief are marked **↩ prior belief overturned** — those are the entries
we think are most worth reading.

Version numbers are the `formula_version` stamped on every published score.

---

## v2.0.0 — Weights chosen by measurement; two negative-IC metrics unscored

**The first version whose pillar weights were chosen by an out-of-sample backtest rather than
by judgement, and the first change to move the score for the whole universe at once.**
Published from 2026-08-25.

| | Quality | Valuation | Growth | Fin. Health | Momentum | Ownership |
|---|---:|---:|---:|---:|---:|---:|
| v1.6.0 | 25 | 20 | 15 | 15 | 15 | 10 |
| **v2.0.0** | **22** | **24** | **9** | **9** | **12** | **24** |

Plus `sales_growth_3yr` and `profit_growth_3yr` are **no longer scored** (still extracted,
still visible in the screener). Minimum-metric bars are unchanged.

**What made this possible.** The live feature table has no as-of column and is overwritten
daily, so until this cycle the score could not be backtested at all — the 28 score dates
that existed were the only 28 that could ever exist. A point-in-time fundamentals store
already held what was needed; wiring it into a separate backtest extractor gave 44 monthly
rebalances over 2022–2025 with a 365-day forward horizon. **No published number moved**
during that work: every backtest artifact is a sibling table, and a golden fixture of 191
real production cases pinned the live formula byte-for-byte until the deliberate cutover.

**↩ prior belief overturned — Ownership.** An 8-week live audit measured Ownership at
+0.017 and we had planned to *retire* it. Out of sample it is the **strongest** pillar
(+0.061, positive on 20 of 20 rebalances). Retiring it would have deleted the best signal in
the score. The actual dead weight was Growth (−0.003) and Financial Health (−0.002): 30
points of weight with no measurable signal, diluting the composite below what Ownership and
Valuation achieve alone.

Per-pillar out-of-sample rank IC, 20 monthly rebalances:

| Pillar | v1 weight | Mean IC | Dates positive |
|---|---:|---:|---:|
| Ownership | 10 | **+0.0609** | 20/20 |
| Valuation | 20 | +0.0545 | 20/20 |
| Quality | 25 | +0.0494 | 20/20 |
| Momentum | 15 | +0.0237 | 9/20 |
| Financial Health | 15 | −0.0021 | 8/20 |
| Growth | 15 | −0.0034 | 10/20 |

**Growth and Financial Health were reduced, not removed.** A zero measured IC over 20 dates
in one regime is not proof of uselessness; both are partly risk controls; and cutting them
to zero would make a three-factor score wearing a six-pillar label. A "remove Growth
entirely" arm was backtested and did not beat the shipped formula in both independent
windows.

**Why exclusion rather than IC weighting.** Full IC-proportional weights were deliberately
*not* fitted: twenty weights against 22 observations is overfitting regardless of the
result, and the in/out-of-sample ratio already widened from 1.57× to 1.70× on this
two-parameter change. Excluding the two negative-IC CAGRs rather than sign-flipping them is
the same discipline — the direction registry encodes economics, not a backtest.

**Measured on the live universe before cutover** (4,451 stocks scored under both formulas
through the full production path): mean signed move −0.15, mean absolute move 2.73, max 15;
1,229 stocks (27.6%) change band; **6 lose their score** (all at exactly 61% coverage,
answering 12 of the 21 questions v2.0.0 asks — arithmetic, accepted rather than
special-cased) and 15 gain one. Band floors were **not** moved: on the governed production
distribution the percentile anchors landed at 65/55/45/38 against floors of 66/55/45/37.

**Withdrawn claim.** The first write-up said "positive on 20 of 20 *independent* rebalances,
p ≈ 1 in 1,048,576". Monthly rebalances measuring 365-day returns overlap by ~11 months, so
they are not independent draws and that p-value was false. The textbook corrections
(Newey-West, block bootstrap) were implemented and **both fail toward more confidence** at
this sample size, so they are gated off. What we state instead: v2.0.0 beats v1.6.0 in
**each of the two non-overlapping horizon windows** (+0.0685 and +0.0677 against +0.0477 and
+0.0559), and it is the most stable variant across them (spread 0.0008). No p-value is
asserted anywhere.

**Also corrected in the same cycle:** the per-metric IC table behind the exclusion decision
had been produced ad hoc with no reproducing code. Rebuilt as a committed phase, it turned
out the naive version measured the *raw* feature rather than the value *as scored* —
winsorization clips both tails into ties and reorders the population, so rank IC is not
invariant to it. Three metrics changed sign under the faithful measurement; the two excluded
CAGRs landed *more* negative (−0.0284, −0.0484), so the exclusion is better supported than
the evidence that motivated it. The COVID base-effect objection was tested on a 2015–2019
window with all forward returns completing before February 2020: rejected for
`sales_growth_3yr`, partly supported for `profit_growth_3yr`.

---

## v1.6.0 — Hysteresis measured on the raw delta

**Bug fix that changes published values, hence a version.** The dead band was tested against
the *smoothed* delta. Since `ema − prev = 0.35 × (raw − prev)`, `|ema − prev| < 2.0` really
enforced `|raw − prev| < 5.714` — **2.86× the documented band**. Solved out of production
data: the largest *held* raw move was 5.710 and the smallest *published* one 5.720.

Not merely lag. `prev` is the previously published *integer*, so a stock inside the inflated
band re-anchored to its own frozen value every run and never converged: mean gap between
published and computed score 2.53 points, **3,557 of 4,531 stocks published exactly one
distinct score across nine runs**, and 2,620 were carrying a suppressed move. It also
concealed a real reference-version defect that swung the Quality pillar ~5 points while the
published integer never moved. A freeze that hides a defect is not anti-whipsaw protection.

Fix: one comparison, on the raw delta. The EMA blend and post-EMA cap re-clamp are unchanged.

---

## v1.5.0 — The promoter-pledge penalty finally fires

**A published claim that was false, corrected.** The methodology had listed pledge penalties
(>25% soft, >50% harder) and a >75% hard cap at 40 since v1.0.0 — and the code path that
loads governance data never set the pledge field, so every branch was dead. **18 stocks
above 75% pledged were publishing scores above the cap the methodology said applied to
them.** The most consequential action the engine takes was wired to nothing.

**Source choice is the correctness story.** The obvious current-snapshot pledge column has
no as-of date, so a historical run would score against today's level. The quarterly
disclosure record is used instead, keyed on the NSE **broadcast** date, not the quarter date:
median lag from quarter end to broadcast is 98 days (minimum 14), so keying on the quarter
would be look-ahead measured in months. A disclosure older than 400 days is dropped, not
carried — NSE re-broadcasts ancient quarters (one candidate was 1,325 days stale), and the
400-day line is read off a cleanly bimodal distribution with nothing between 150 and 400.
Dropping is the conservative direction for a penalty.

Shadow-replayed before shipping: 134 stocks flagged, 37 hard-capped, capped drops average
3 points in micro-caps and 8–9.5 in larger names, max 29. **The drop is immediate**, because
the hard cap is re-clamped after smoothing — correct for a hard cap, and stated here rather
than shipped quietly.

---

## v1.4.0 — Coverage measured against what the template can emit

**↩ prior belief overturned — this repository previously argued the opposite.** We had
declined a per-template coverage denominator on the grounds that a bank on 8 of 8 applicable
metrics would report 100% and "outrank" an industrial on 90% while carrying less
information.

Measured: with the flat 23 denominator, `bank` could reach at most 16 of 23 and `nbfc` 18, so
against a flat 60% eligibility bar `general` had 36 points of headroom and `bank` had 5. The
effect was exactly what that predicts — **18.3% of NBFCs ruled ineligible against 3.1% of
general stocks, and 209 of 223 ineligible names failed on coverage alone**, not on pillar
breadth. Their data was not worse; their denominator was. A large NBFC with every applicable
metric present showed "Insufficient data".

Now the denominator is the metrics the stock's template can emit: `nbfc` never emits
`opm_ttm`, `ev_ebitda`, `altman_z`, `fcf_margin`, `cfo_to_np`; `bank` additionally never emits
`roce_5yr` or `debt_to_equity`. Kept as an inapplicability list with the count *derived*, so
adding a metric automatically raises every ceiling that can emit it. An unknown template
deliberately gets the full denominator — understating coverage suppresses a score, which
fails safe. Shadow-replayed: 92 stocks gain eligibility, 0 lose it, `general` untouched.

Two other defects fixed in the same release without changing the formula: the reference
depth gate was keyed without `template`, so `bank` and `nbfc` (both in sector Financial
Services) collided and banks were normalised against a 25-observation cell that should have
fallen back; and a degenerate reference cell (no usable scale) terminated the fallback search
instead of continuing to the next level (120 such cells, all with a usable coarser
counterpart).

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
| **IC-proportional pillar weights** (v1 cycle) | Looked like a decisive win in-sample (0.110 vs 0.060). Fit/holdout **inverted the ranking**: live weights 0.047 → **0.085 (best)**, IC-weighted 0.141 → 0.054. Superseded by the point-in-time backtest that produced v2.0.0, which is a two-parameter tilt, not a fit. |
| **Dropping negative-IC pillars** (Quality, Financial Health) | Best in-sample (0.179), **worst on holdout (0.036)**. |
| **Institutional M−1 momentum skip** | Both NSE and MSCI mandate it. On our data it **cost** signal: 6m 0.074 → 0.068, 12m 0.047 → 0.038. |
| **Small-size factor** | Actively harmful, IC **−0.084**. |
| **"Fixing" accruals to match published India research** | Our data shows the US sign (−0.093), holdout-confirmed negative on 50/50 dates in both windows. We kept our measurement. |
| **Dedicated insurance template** | Only ~7 insurance stocks clear the liquidity gate, so it would calibrate median/MAD from 7 observations. Folded into `nbfc`, which already nulls what insurers cannot support. |
| ~~**Per-template coverage denominator**~~ | **Reversed in v1.4.0.** We rejected it on the "8-of-8 outranks 90%" argument; measurement showed the flat denominator was ruling 18.3% of NBFCs ineligible on their denominator alone. Kept here so the original reasoning stays on record. |
| **IC-proportional weights, fitted** (v2 cycle) | Re-tested against point-in-time features: fitting ~20 metric weights to 22 observations widened the in/out-of-sample ratio on even a two-parameter change. Metric *exclusion* shipped; IC weighting did not. |
| **Raising Financial Health to 2 minimum metrics** | Smallest gain of the three v2 candidates (+0.002 IC) and by far the largest cost: 175 previously scored stocks would have lost their score. Dropped from the ship candidate. |
| **Removing Growth entirely** | Backtested as an arm. Did not beat the shipped formula in both independent horizon windows; its apparent edge sat in one window only. |
| **Ownership holding LEVELS** (in addition to changes) | Measured and deliberately not shipped; ~35% of stocks score exactly 50 on Ownership because their holdings did not move. Open work. |
| **Hysteresis on band thresholds** | Makes classification stateful, breaking the determinism that is the whole defensibility argument. |
| **Bank-specific metrics** (NPA, CAR, PCR, NIM, CASA) | Not rejected — **not available**. Audited: those fields return zero rows across our data estate. Requires a new regulatory-filings source. |
