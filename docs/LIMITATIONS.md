# Limitations

What the Tapetide Score cannot do, where the model is weakest, and which claims we
deliberately do not make.

This document exists because every one of these items is something a determined reader
would eventually find. We would rather state them plainly than be caught having implied
otherwise. Several of these are corrections to claims we previously made publicly.

---

## 1. The score is not point-in-time

**This is the most important limitation and the one most likely to mislead you.**

Fundamental inputs are the **latest reported figures**, not the figures as they stood on a
past date. A score dated 18 months ago was computed from whatever the fundamentals record
said at the time the run executed — which incorporates restatements, revisions, and late
filings that were not public then.

Consequences:

- **Score history is a trend, not a replayable backtest.** You cannot treat the published
  history as "what the score would have said on that date".
- Any backtest of the composite carries look-ahead bias in the fundamental pillars. Price
  pillars (Momentum) are unaffected.

We previously published the opposite claim — "point-in-time honest… no look-ahead bias…
the score history you see is the score you would have seen". **That was false and has been
retracted** from both the methodology page and the API.

### Why this is not a quick fix

A point-in-time vintage store *does* exist as a separate dataset — roughly 9 million
vintage records across ~8,300 stocks, 45 KPIs, quarterly back to 2002 — and **the scoring
engine reads none of it**. Wiring it in means teaching feature extraction to read vintages
as-of a date and then re-deriving every historical feature vector. That is a project, not a
patch.

And even then the honest usable depth is narrower than the raw record suggests. **Period
depth is not availability depth.** Grouping vintages by year suggests ~2,900 stocks have
2015–2020 history, which reads like a decade of usable data. But of 920 liquid stocks on a
2018 test date, only **26** have any vintage that was actually *available* by that date —
868 of 919 have their earliest availability stamped **2021**, the point at which vintage
capture began. Quarterly coverage is a source limitation (~12 trailing quarters), giving
261–400 stocks per year for 2013–2020 and then 4,102 in 2021.

So a genuinely lookahead-free fundamental study spans roughly **2021 to mid-2025** — about
4.5 years, or **4–5 independent annual observations**. That is enough to check whether a
factor's *sign* is right. It is nowhere near enough to fit six pillar weights, which is
part of why the weights are anchored on external multi-factor precedent
([VALIDATION.md](VALIDATION.md) §4).

We previously stated this floor as ~2015. That was wrong and was corrected.

---

## 2. "72 = 72" holds within a reference version, not across all time

The peer reference distribution is frozen and immutable, so re-running a date against the
same version reproduces the same number exactly. That part is real.

But each reference version is **calibrated from a single day's cross-section**, not from a
pooled multi-year history. So a score is precisely comparable to other scores computed
against the same version, and only approximately comparable across versions.

Earlier copy claimed "a 72 today means the same as a 72 last year". That overstated it and
has been narrowed.

### A pooled historical reference is not currently buildable

The long-standing plan was "freeze a 5-year pooled distribution". Verified against the
actual data, that cannot be executed as written: the fundamentals snapshot holds **exactly
one row per stock** — it is a latest-state table with no history — and the derived feature
store spans only a handful of recent dates. There are no historical fundamental
cross-sections to pool. The vintage store in §1 has the depth, and connecting it is the
same separate project. Until then, pooled calibration is not one command away, and we will
not describe it as such.

---

## 3. The governance overlay is active but unvalidated

The overlay materially moves scores: on a recent run, **807 of 4,496 scored stocks carry a
penalty and 130 are hard-capped**. It is not decorative. (An earlier internal guess that it
was "near-inert" was wrong.)

But the specific penalty magnitudes — −0.10 for surveillance, −0.15 for heavy pledge, the
40/50 caps — are **asserted, not measured**. Every regulatory disclosure feed we use begins
in 2026, while our forward-return labels end mid-2025. The windows **do not overlap at
all**, so a backtest returns either a single unflagged row or zero rows. The earliest
honest test is around **March 2027**.

Two feeds are additionally thin: quarterly pledge data covers 19 stocks and credit-rating
actions 27. The overlay leans on the broader event-based pledge history instead, but this
is a data-acquisition gap, not a modelling choice.

**So: the governance overlay reflects real disclosed facts, and its severity weighting is
judgement.** We are not going to pretend those penalty numbers came out of a study.

---

## 4. Peer cells can be built from very few companies

The `n_obs ≥ 30` minimum applies only to the finest (L1) reference level. L2 and L3 return
unconditionally.

Measured on the live reference:

| Level | Cells | Under 30 obs | Under 10 obs |
|---|---:|---:|---:|
| L1 `(template, sector, size)` | 1,489 | 910 | 537 |
| L2 `(template, sector, ALL)` | 523 | 253 | 225 |
| L3 `(template, ALL, ALL)` | 59 | 0 | 0 |

So a metric can legitimately resolve to an L2 cell whose median and MAD were computed from
fewer than ten companies. Extending the guard to L2 is open work and should be done before
any future reference version is blessed — otherwise a "properly rebuilt" reference bakes in
the same thin cells.

Additionally, 369 of 2,071 cells in the live reference are fully degenerate (no usable
scale), 124 of them at L2. Metrics hitting those cells are dropped and reported rather than
silently zeroed.

---

## 5. About 3,800 listed companies are not scored at all

Roughly 4,500 of ~8,300 active listed companies clear the gates. The gap is almost entirely
the liquidity requirement (≥200 traded days in the trailing 400), and it is intentional:
price-based metrics computed from a stale last traded price are fiction, and valuation
ratios on a price nobody transacted at are worse than no number.

Of the ~3,800 excluded, most trade fewer than 120 days a year and only about 100 have both
an active fundamentals record and a market cap above ₹500 crore. Relaxing the gate to 120
days would add roughly 490 stocks, of which about 64 are above ₹500 crore.

**We would rather show "not scored" than a confident number built on 60 trading days.**

---

## 6. Bank and NBFC scoring is structurally weaker

Financial templates null out five metrics they cannot support, so financials are scored on
fewer inputs and report lower coverage by construction: measured cohort averages are
`general` **83.4%**, `bank` **70.4%**, `nbfc` **67.6%**. In the NBFC cohort, 53 eligible
stocks sit within 8 percentage points of the 60% eligibility bar.

The proper fix is bank-specific metrics — NPA ratios, capital adequacy, provision coverage,
net interest margin, CASA. **We audited for these and they are not available to us:** those
fields return zero rows across our entire data estate, and the vintage store holds only an
interest line for banks (no deposits, advances, NPAs, or provisions). Sourcing them means a
new regulatory-filings collector, which is a separate project. The classification half of
the fix shipped; the metric-substitution half did not.

Each future `general`-only metric widens this gap by roughly 4 percentage points for
financials. It is a tracked constraint.

---

## 7. Pillar weights carry little information

Stated plainly because [VALIDATION.md](VALIDATION.md) §4 shows it: the live weights beat
naive equal weighting on holdout by 0.085 to 0.076. That is close to a tie.

The honest reading is that the *choice of weights* is not where the score's value comes
from — the normalization, the sector-relative peer comparison, the eligibility gating, and
the governance overlay do more work. We keep 25/20/15/15/15/10 because it won the holdout
and matches established multi-factor precedent, not because we have proven it optimal.

Two individual pillars have negative standalone ICs on our data: Quality −0.007 and
Financial Health −0.043. We keep both, because deleting them was measurably the **worst**
variant tested.

---

## 8. Survivorship bias is addressable but not eliminated

Our validation panel carries a delisting-candidate flag, so survivorship can be handled in
factor studies. But the *live* score universe is by construction currently-listed
companies. Any statement of the form "stocks scoring above 70 returned X%" computed from
today's universe is survivorship-contaminated unless the delisting flag is applied.

We previously described v1 backtesting as "on the currently-listed universe" with full
correction promised in v2. The panel now supports the correction; not every historical
number we have quoted was computed with it applied.

---

## 9. What the score structurally cannot tell you

- **Anything about the future.** It is a summary of past and present evidence. There is no
  forecasting model anywhere in it.
- **Anything not in the numbers.** Management quality, competitive moat, product cycles,
  regulatory shifts, litigation, key-person risk, accounting aggressiveness beyond the
  specific flags we track.
- **Absolute quality.** Everything is relative to a peer group. A 72 in a structurally
  weak sector is a 72 *within that sector*.
- **Timing.** There is no entry point, target price, or holding period, and there will not
  be. We are not a registered research analyst and do not give directional calls.
- **Event risk.** A score computed this evening does not know about tomorrow's news.

---

## 10. Known open work

Ordered roughly by how much we think it would improve the score:

1. **Point-in-time feature extraction** — wire the vintage store into feature building and
   backfill. Removes the §1 caveat and unlocks honest composite backtesting.
2. **Extend the `n_obs` guard to L2** before cutting any new reference version (§4).
3. **Pooled historical reference distribution**, which depends on (1).
4. **Bank/NBFC metric set** — requires a new regulatory-filings source (§6).
5. **Governance overlay calibration** — blocked on time, testable from ~March 2027 (§3).
6. **Liquidity gate relaxation** to ~120 traded days with a lower confidence band (§5).

If you want to contribute to any of these, [CONTRIBUTING.md](../CONTRIBUTING.md) says how.

---

## Our standard for this document

If we find that something we have published about the score is not true of the code, we
correct it here and in the product, and we say what the previous claim was. Four
corrections are recorded above (§1 point-in-time, §1 the 2015 floor, §2 the "72 = 72"
claim, and the removal of "volatility-adjusted" from Momentum in
[METHODOLOGY.md](METHODOLOGY.md)).

A methodology document that only ever gains confident claims is not being checked.
