# Tapetide Score — Complete Methodology

**Formula version: v1.3.0**

This is the full specification of the Tapetide Score. Every constant here is the value the
production engine runs on. Nothing is rounded off, generalized, or withheld.

Sections:

1. [Universe and eligibility](#1-universe-and-eligibility)
2. [Metric templates](#2-metric-templates)
3. [The peer reference distribution](#3-the-peer-reference-distribution)
4. [Normalizing one metric to 0–100](#4-normalizing-one-metric-to-0100)
5. [The 23 metrics](#5-the-23-metrics)
6. [Pillar sub-scores](#6-pillar-sub-scores)
7. [The composite](#7-the-composite)
8. [The value-trap damper](#8-the-value-trap-damper)
9. [The governance overlay](#9-the-governance-overlay)
10. [Hard caps](#10-hard-caps)
11. [Smoothing and hysteresis](#11-smoothing-and-hysteresis)
12. [Coverage, confidence, and the publish gate](#12-coverage-confidence-and-the-publish-gate)
13. [Reproducibility](#13-reproducibility)
14. [Worked example](#14-worked-example)

---

## 1. Universe and eligibility

A stock enters the scoring universe for a given date only if **all** of the following
hold:

| Gate | Rule | Why |
|---|---|---|
| Active listing | Marked active in the fundamentals snapshot | Delisted/suspended names must not rank |
| Market cap | `> 0` | A missing cap makes size bucketing impossible |
| Liquidity | **≥ 200 traded days in the trailing 400 calendar days** | Price factors are meaningless on a stock that barely trades; a stale last price fakes both momentum and valuation |
| Size bucket | Must resolve to a real bucket (not `unknown`) | Peer comparison requires a peer group |

**Size buckets**, by market capitalization in ₹ crore:

| Bucket | Range |
|---|---|
| `large` | ≥ 20,000 |
| `mid` | 5,000 – 20,000 |
| `small` | 500 – 5,000 |
| `micro` | > 0 and < 500 |

The liquidity gate is the binding constraint. Of roughly 8,300 active listed companies,
about **4,500** clear it. The rest are genuinely thinly traded: most trade fewer than 120
days a year, and only a small minority have both an active fundamentals record and a
market cap above ₹500 crore. **We leave them unscored rather than score them on sparse
data** — see [LIMITATIONS.md](LIMITATIONS.md).

---

## 2. Metric templates

Some metrics are meaningless for some business types. "Operating profit margin" is
undefined for an insurer; EV/EBITDA and Altman Z are not interpretable for a bank. A
model that computes them anyway does not produce a slightly-worse number, it produces a
**confidently wrong** one.

Three templates:

| Template | Assigned to | Excluded metrics |
|---|---|---|
| `bank` | Industry is Private Bank, Public Sector Bank, or Small Finance Bank | `opm_ttm`, `ev_ebitda`, `fcf_margin`, `altman_z`, `cfo_to_np` |
| `nbfc` | **Any other stock whose sector is Financial Services** — NBFCs, housing finance, AMCs, insurers, brokers, fintech, holding companies | same as `bank` |
| `general` | Everything else | none |

**Two rules here are load-bearing and both were bought with a real defect.**

**(a) Financial-template assignment is SECTOR-first, not industry-list-first.** An earlier
version matched an explicit list of industry strings, so any Financial Services company
whose industry string was not on the list fell through to `general` and was scored on
industrial metrics. That silently mis-scored 83 live companies. Insurers reported
*negative* operating margins (−75.0 for one large life insurer, −35.4 for another) because
operating profit is not a defined concept for insurance — and those negative values fed
straight into the Quality pillar. Two of the misses were pure spelling variants
("Financial Technology (Fintech)" vs "Fintech").

The defect was **invisible from aggregate statistics**: the mis-templated cohort reported
the *highest* data coverage and confidence of any group (86.3% / 92.1 versus ~71% / ~78
for correctly-classified financials), precisely because correct classification NULLs the
inapplicable metrics while misclassification populates them with garbage. A confidently
computed wrong answer produces no error and no coverage gap.

Now: any Financial Services stock defaults to a financial template, and the industry list
only chooses *which* one. A new or renamed financial industry degrades to `nbfc` (metrics
nulled — safe) rather than `general` (metrics wrong — unsafe).

**(b) Insurers deliberately share the `nbfc` template** rather than getting their own. Only
about 7 insurance stocks clear the liquidity gate, and the reference distribution is keyed
by template at every fallback level — so a 7-stock template would calibrate its own
median/MAD from 7 observations. `nbfc` already nulls every metric an insurer cannot
support, which is the actual correctness fix. **Adding a template means adding a peer
group; count its population first.**

Note that the `bank` branch is intentionally industry-only with no sector guard. Every
bank-industry stock is already in Financial Services, so the guard would be a no-op today
— and it would fail *unsafe* later, since a bank arriving with a blank sector would be
pushed out of `bank` into `general`.

---

## 3. The peer reference distribution

Each metric is scored against a **frozen, versioned peer distribution** rather than
against today's live cross-section. This is what makes a score reproducible: re-running an
old date against the same reference version reproduces the same number.

A reference is a table of cells, each holding four statistics for one
`(template, sector, size bucket, metric)` combination:

- `median`
- `mad` — median absolute deviation, `median(|x − median|)`
- `winsor_lo` — 2nd percentile
- `winsor_hi` — 98th percentile
- `n_obs` — how many observations the cell was built from
- `higher_better` — direction

**All four statistics are percentile-based. There is no mean and no standard deviation
anywhere in the reference builder.** This is not stylistic. It is the only reason
unbounded ratios are safe here: cash conversion reaches 245× on live data (the smallest
positive profit denominator is ₹0.10 crore), but the p2/p98 winsor bounds for that metric
are −7.39 and +22.05, so every extreme collapses onto the bound. Swap in `avg`/`stddev`
and unbounded ratios break first and loudest.

### Three fallback levels

A stock resolves its reference cell by walking from most specific to least:

| Level | Cell key | Guard |
|---|---|---|
| **L1** | `(template, sector, size)` | Used only if `n_obs ≥ 30` |
| **L2** | `(template, sector, ALL)` | No minimum-observation guard |
| **L3** | `(template, ALL, ALL)` | No minimum-observation guard |

**Known weakness, stated plainly:** the `n_obs ≥ 30` guard applies to L1 only. On the
current live reference, L2 has 523 cells of which **253 are built from fewer than 30
observations and 225 from fewer than 10**. So a metric can legitimately fall back to a
cell whose median and MAD came from under ten companies. Extending the guard to L2 is open
work — see [LIMITATIONS.md](LIMITATIONS.md).

### Reference versions are immutable

A reference version is written once and never modified. Published scores pin the version
they were computed against, so mutating one would silently change the baseline every
historical score was measured on, with no way to recover the old values.

Adding a metric therefore requires **cutting a new version**, and the version a run uses
is resolved by looking up which version was *effective on or before the score date* —
never derived from the date arithmetically. That distinction is not academic: deriving the
name as `ref_<year>` caused a live regression where a newly activated metric was computed
into the inputs every night and then silently discarded at scoring time, because the
nightly job kept selecting the previous year-named version that had no cell for it.

---

## 4. Normalizing one metric to 0–100

Given a raw value `x` and its resolved reference cell:

```
1. Winsorize:   x' = clamp(x, winsor_lo, winsor_hi)

2. Robust scale:
                scale = 1.4826 × mad                        if mad > 1e-9
                scale = (winsor_hi − winsor_lo) / 4         otherwise
                → if both collapse, the metric is DROPPED (not zeroed)

3. Robust z:    z = (x' − median) / scale
                z = −z                                      if lower is better

4. Clip:        z = clamp(z, −3.0, +3.0)

5. Logistic:    score = 100 / (1 + e^(−1.5 · z))
```

Notes on each choice:

- **1.4826** is the standard consistency constant that puts MAD on the same footing as a
  standard deviation for normally distributed data.
- **The fallback scale** exists because MAD legitimately collapses to zero for metrics
  where most companies report the same value — change-in-holding figures cluster hard at
  0. Treating the winsor range as roughly ±2σ recovers a usable spread instead of dividing
  by zero. If the winsor range is *also* degenerate the metric carries no information and
  is dropped.
- **A dropped metric is reported, never silent.** Two separate cases are tracked and
  reported distinctly: `unreferenced` (no cell exists for this metric — how a newly added
  metric ships inert) and `degenerate` (a cell exists but carries no usable scale). They
  look identical in a score but tell an operator to fix completely different things, so
  merging them would send someone to rebuild the wrong artifact.
- **z clip ±3** bounds a survivor of winsorization; combined with `k = 1.5` this puts the
  practical score range at roughly 1–99 rather than a hard 0/100.
- **The logistic curve** is deliberate: it is steepest near the median, so distinctions
  among ordinary companies are amplified, and it flattens at the extremes, so being the
  99th-percentile cheapest stock is worth little more than the 96th. Linear percentile
  ranking would treat those as equally meaningful.

---

## 5. The 23 metrics

Each metric belongs to **exactly one** pillar. This is enforced: a metric appearing in two
pillars would let one underlying quantity dominate the composite through the back door.

`↑` = higher is better · `↓` = lower is better

### Quality — weight 25

| Metric | Dir. | Definition | Template gate |
|---|:--:|---|---|
| `roce_5yr` | ↑ | Return on capital employed, 5-year average | all |
| `roe_5yr` | ↑ | Return on equity, 5-year average | all |
| `piotroski` | ↑ | Piotroski F-Score (0–9) | all |
| `opm_ttm` | ↑ | Operating profit margin, trailing 12 months | `general` only |
| `fcf_margin` | ↑ | Free cash flow ÷ revenue × 100 | `general` only, revenue > 0 |
| `cfo_to_np` | ↑ | Operating cash flow ÷ net profit — **cash conversion** | `general` only, **net profit > 0** |

`cfo_to_np` is the strongest single factor we have measured (rank IC +0.091, positive on
50 of 50 rebalance dates) and it is nearly orthogonal to everything else in the score
(−0.116 against earnings yield). **Both its gates are load-bearing:**

- **`net profit > 0`** — the ratio inverts sign when the denominator is negative, so a
  loss-maker with negative operating cash flow would score as *good* cash conversion.
  Restricting to profitable firms *raised* measured IC from +0.067 to +0.091, so this is
  the better factor, not merely the safer one.
- **`general` only** — for a bank or NBFC, operating cash flow is dominated by deposit and
  borrowing flows, so the ratio is not a quality measure at all. Measured: financials were
  3× over-represented in its top decile (32.3% versus an ~11% baseline). That is an
  artifact, not quality.

### Valuation — weight 20

| Metric | Dir. | Definition | Template gate |
|---|:--:|---|---|
| `earnings_yield` | ↑ | Earnings ÷ price | all |
| `pe_ttm` | ↓ | Price ÷ trailing earnings | all, only when > 0 |
| `pb` | ↓ | Price ÷ book value | all, only when > 0 |
| `ev_ebitda` | ↓ | Enterprise value ÷ EBITDA | `general` only, > 0 |
| `fcf_yield` | ↑ | Free cash flow ÷ market cap | all |
| `dividend_yield` | ↑ | Dividend ÷ price | all |

Negative P/E and P/B are excluded rather than scored. A loss-making company does not have
a "cheap" P/E — it has an undefined one, and scoring −4 as cheaper than +12 is simply
wrong.

Note the deliberate split: **FCF margin sits in Quality, FCF yield in Valuation.** Margin
is about the business, yield is about the price. Putting both in one pillar would
double-count free cash flow.

### Growth — weight 15

| Metric | Dir. | Definition |
|---|:--:|---|
| `sales_growth_3yr` | ↑ | Revenue CAGR, 3 years |
| `profit_growth_3yr` | ↑ | Profit CAGR, 3 years |
| `sales_growth_ttm` | ↑ | Revenue growth, trailing 12 months |

### Financial Health — weight 15

| Metric | Dir. | Definition | Template gate |
|---|:--:|---|---|
| `altman_z` | ↑ | Altman Z-Score (distress) | `general` only |
| `debt_to_equity` | ↓ | Total debt ÷ equity | all |
| `interest_coverage` | ↑ | EBIT ÷ interest expense | all, interest > 0 |

### Momentum — weight 15

| Metric | Dir. | Definition |
|---|:--:|---|
| `mom_6m` | ↑ | Six-month price return |
| `dist_200dma` | ↑ | `(price ÷ 200-day MA − 1) × 100` |

Only two metrics, and Momentum's minimum bar is 2 — **zero slack**, by design and
documented as such. Two metrics were removed here on measured evidence:

- **`rsi_14` removed (v1.1.0)** — weakest of the four momentum signals by cross-sectional
  rank IC (+0.039 versus +0.074 for `mom_6m`), and absent from every institutional
  momentum construction. It remains available as a user-facing screener filter; it is just
  not in the score.
- **`mom_12m` removed (v1.2.0)** — and this one is counter-intuitive enough to be worth
  reading [VALIDATION.md](VALIDATION.md) for. Pairwise correlation cannot tell you which
  of two correlated metrics is redundant; residual IC can. `dist_200dma` ranked *within*
  `mom_12m` deciles retains +0.062 IC in holdout (positive on 46 of 54 dates), while
  `mom_12m` ranked within `dist_200dma` deciles is −0.0005 (30 of 54 — a coin flip). The
  12-month signal carried nothing the 200DMA distance lacked, and blends confirmed it
  actively diluted.

**There is no volatility term in Momentum.** Earlier public copy described it as
"volatility-adjusted"; that was wrong and has been corrected. Low volatility was tested as
a candidate factor and **rejected** — see [VALIDATION.md](VALIDATION.md).

### Ownership — weight 10

| Metric | Dir. | Definition |
|---|:--:|---|
| `chg_promoter` | ↑ | Change in promoter holding |
| `chg_fii` | ↑ | Change in foreign institutional holding |
| `chg_dii` | ↑ | Change in domestic institutional holding |

Direction of change, not level. A high promoter stake is neither good nor bad on its own;
promoters *increasing* their stake is information.

---

## 6. Pillar sub-scores

A pillar's sub-score is the **equal-weighted mean of its available metric scores** — but
only if the pillar clears its minimum-metric bar:

| Pillar | Min. metrics |
|---|---:|
| Quality | 2 |
| Valuation | 2 |
| Growth | 1 |
| Financial Health | 1 |
| Momentum | 2 |
| Ownership | 1 |

**A pillar below its bar is ABSENT, never zero.** This is one of the most important rules
in the model. Zero-filling a missing pillar would read as "worst possible", so a data gap
would present to the user as a damning verdict. Absent means absent.

Metrics within a pillar are equal-weighted deliberately. We have no evidence that would
justify finer weighting inside a pillar, and inventing precision we cannot support is
worse than admitting we do not have it.

Summation is order-stable: values are sorted before summing so floating-point addition
cannot vary with input ordering. Determinism has to hold at the bit level to be worth
claiming.

---

## 7. The composite

The composite is a weight-**renormalized** sum over available pillars:

```
total_w   = Σ weight(p) for each available pillar p
core      = Σ (weight(p) / total_w) × subscore(p)
```

Renormalization matters: a stock missing Ownership (weight 10) is scored on the remaining
90 rescaled to 100, rather than being penalized 10 points for a data gap. The realized
weights are published per stock alongside the score, so you can see exactly what was used.

Pillars are folded in a fixed order so the arithmetic is reproducible.

---

## 8. The value-trap damper

Extreme cheapness on a fragile balance sheet is not a bargain; it is usually the market
correctly pricing distress.

```
if valuation > 70 and financial_health < 40:
    penalty = ((valuation − 70) / 30) × ((40 − financial_health) / 40) × 6.0
    adjusted = core − penalty
```

Maximum effect is 6 points, scaling smoothly with how extreme both conditions are. It
applies only when Valuation is genuinely high *and* Financial Health genuinely weak — it
is a joint condition, not a penalty on cheap stocks.

---

## 9. The governance overlay

Governance is a **multiplier**, never an additive pillar. This is a deliberate structural
choice: an additive governance score would let strong fundamentals *average away* a red
flag. A multiplier can only pull down.

Penalties are subtracted from a 1.0 multiplier and the result is floored at **0.50** — the
overlay can never more than halve a score:

| Red flag | Penalty |
|---|---:|
| Under exchange surveillance (any stage) | −0.10 |
| Credit rating downgrade | −0.10 |
| Promoter pledge > 50% | −0.15 |
| Promoter pledge 25–50% | −0.07 |
| Insider selling ratio > 0.7 | −0.05 |
| Audit qualification / going concern | −0.15 |

```
composite_raw = adjusted × clamp(multiplier, 0.50, 1.0)
```

**Insider-selling classification** deserves a note, because it is subtle. The disposal
predicate matches transaction types containing `sell`, `dispos`, or **`invoke`** — the
last because a pledge *invocation* is a lender liquidating collateral, which is
economically a disposal. One company scored 0.000 on insider selling while 21 of its 22
disclosures were collateral liquidations. Critically, `invoke` does **not** match
`Pledge Revoke`, which is routine and correctly excluded — the two words differ by one
letter and matching the wrong one inverts the signal entirely.

**Honest disclosure about validation:** the overlay is materially active — on the current
run, **807 of 4,496 scored stocks carry a penalty and 130 are hard-capped** — but the
specific penalty magnitudes above are **asserted, not measured**. Every regulatory
disclosure feed we use begins in 2026, while our forward-return labels end mid-2025, so
the two windows do not yet overlap and the penalties cannot be backtested. The earliest
honest test is around March 2027. We would rather say that than imply a calibration we do
not have. See [LIMITATIONS.md](LIMITATIONS.md).

---

## 10. Hard caps

Some conditions override fundamentals entirely. Caps are absolute ceilings, and when
several apply, **the lowest wins**:

| Condition | Cap |
|---|---:|
| Graded Surveillance Measure (GSM) | 40 |
| Additional Surveillance Measure stage 3 or 4 | 40 |
| Audit qualification / going-concern doubt | 40 |
| Promoter pledge > 75% | 40 |
| ASM stage 2 / ESM stage 2 | 50 |
| Stage-1 ESM/ASM | *no cap* (soft penalty only) |

A stock under GSM cannot score above 40 no matter how good its ratios look. That is the
point.

---

## 11. Smoothing and hysteresis

Raw scores move on daily price data. Without damping, a user would see meaningless
one-point wobble every day and lose the ability to notice a real change.

```
if a previous published score exists:
    ema = 0.35 × composite_raw + 0.65 × previous
    ema = min(ema, lowest_active_hard_cap)        ← re-clamp, see below
    published = previous  if |ema − previous| < 2.0     (hysteresis: hold)
    published = ema       otherwise
else:
    published = composite_raw

final = round_half_even(published) clamped to [0, 100]
```

**The post-EMA re-clamp is not redundant, and omitting it was a real bug.** EMA blends
toward the previously published score, so a stock previously at 90 that acquires a GSM cap
of 40 would compute `0.35 × 40 + 0.65 × 90 ≈ 72` — publishing **above a "hard" cap** until
the EMA decayed. The cap must be re-applied after smoothing, and after the hysteresis-hold
branch too, or the hold path leaks the same violation. This is covered by a dedicated
regression test in the engine.

Measured effect of smoothing on live data: on a typical day **4,439 of 4,489** stocks
publish an unchanged score, mean absolute movement is **0.02–0.12 points**, and the
largest single-day move is under 10. Large jumps therefore mean something real changed —
which is the entire purpose of damping.

---

## 12. Coverage, confidence, and the publish gate

```
coverage_pct = used_metrics / 23 × 100
pillar_breadth = available_pillars / 6 × 100
confidence   = round(0.5 × coverage_pct + 0.5 × pillar_breadth)

eligible = (available_pillars ≥ 4) AND (coverage_pct ≥ 60)
```

An ineligible stock is published as **"Insufficient data"** with no composite score. The
number is not computed-then-hidden; it is not shown because it would not be trustworthy.

**Low data lowers CONFIDENCE, never the SCORE.** Conflating the two would make sparse
coverage look like poor performance.

**The coverage denominator is global (23) by design**, and a per-template denominator has
been proposed and deliberately declined. `coverage_pct` means "share of the engine's full
metric set this stock has", which is comparable across every stock. Under a per-template
denominator it would mean "share of what we chose to try" — so a bank with 8 of its 8
applicable metrics would report 100% and outrank an industrial at 90% while carrying
strictly less information. The tracked cost of this choice is that financial templates
sit lower by construction: measured cohort averages are `general` 83.4%, `bank` 70.4%,
`nbfc` 67.6%. Each future `general`-only metric costs financials roughly 4 percentage
points. That is a known, accepted constraint, not a bug.

---

## 13. Reproducibility

Every published score carries:

| Stamp | Meaning |
|---|---|
| `formula_version` | The version of this specification used |
| `ref_version` | Which frozen peer distribution it was scored against |
| `input_hash` | Hash of `formula_version | ref_version | input features` |

The input hash is what makes the determinism claim checkable rather than rhetorical: two
rows with identical hashes must carry identical scores. Consequently, **any behavioural
change requires a formula-version bump** — changing metric membership, weights, minimum
bars, the sigmoid/EMA constants, governance penalties, or caps while leaving the version
unchanged would let two contradictory scores appear byte-identical in the history and
would destroy the audit trail.

---

## 14. Worked example

A `general`-template mid-cap. Suppose Quality resolves to 5 metrics, Valuation 5, Growth
3, Financial Health 3, Momentum 2, Ownership 3 — 21 of 23 metrics, so
`coverage_pct = 91`, all 6 pillars available, `confidence = round(0.5×91 + 0.5×100) = 96`.

Take one metric, `roce_5yr = 22.4`, against a peer cell with
`median = 14.2, mad = 5.1, winsor_lo = −2.0, winsor_hi = 41.0`:

```
x'    = clamp(22.4, −2.0, 41.0)     = 22.4
scale = 1.4826 × 5.1               = 7.561
z     = (22.4 − 14.2) / 7.561      = 1.0845      (higher is better → keep sign)
z     = clamp(1.0845, −3, 3)       = 1.0845
score = 100 / (1 + e^(−1.5 × 1.0845)) = 83.5
```

Repeat per metric; average within each pillar. Say the pillars land at Quality 71,
Valuation 48, Growth 62, Financial Health 66, Momentum 55, Ownership 58. All six are
available so weights are used unrenormalized:

```
core = 0.25×71 + 0.20×48 + 0.15×62 + 0.15×66 + 0.15×55 + 0.10×58
     = 17.75 + 9.60 + 9.30 + 9.90 + 8.25 + 5.80
     = 60.60
```

Valuation is 48, below the 70 damper threshold, so no value-trap penalty. With no red
flags the multiplier is 1.0 and no cap applies, giving `composite_raw = 60.60`.

If the previous published score was 60:

```
ema = 0.35 × 60.60 + 0.65 × 60 = 60.21
|60.21 − 60| = 0.21 < 2.0       → hysteresis holds
published = 60
```

The score stays at **60** — a 0.6-point drift in fundamentals is not news.

---

## Interpreting the bands

| Range | Label | Meaning |
|---|---|---|
| 70–100 | Strong | Broad strength across pillars, clean governance |
| 60–70 | Above average | Meaningful strengths in several pillars |
| 50–60 | Average | Roughly typical for its sector and size |
| 30–50 | Below average | Notable weaknesses or caveats |
| 0–30 | Weak | Multiple weak pillars and/or active red flags |

On live data the distribution is a clean bell centred near 55, with roughly 2 stocks above
70 and about 80 below 30 in a typical run — a genuinely strong or genuinely broken company
is rare, which is what a well-calibrated rating should show.

---

*Continue to [VALIDATION.md](VALIDATION.md) for how factors are tested, or
[LIMITATIONS.md](LIMITATIONS.md) for what this model cannot do.*
