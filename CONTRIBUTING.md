# Contributing

We are opening this methodology because we want it argued with. If you have quant,
accounting, or Indian-market expertise, the most valuable thing you can do here is show us
where we are wrong.

You do **not** need access to our data to contribute meaningfully. Most of what follows can
be done with any reasonable panel of Indian equity data.

---

## What we most want

Roughly in order of how much it would improve the score:

**1. A factor we are missing, with evidence.**
Not "have you considered X" — a measurement. See [the bar](#the-bar-for-a-factor-proposal)
below. A candidate that clears it will be taken seriously and credited.

**2. A reproducible disagreement with one of our measurements.**
[docs/VALIDATION.md](docs/VALIDATION.md) publishes our measured ICs and windows. If you run
the same test and get a materially different answer, that is the single most useful issue
you can open. Include your universe definition, window, label horizon, and method.

**3. A defect in the specification.**
Somewhere the math is wrong, a gate is unsafe, a direction is inverted, or two rules
contradict each other. We have shipped all four of those before and caught them late.

**4. Accounting or sector expertise on the template problem.**
The bank/NBFC template is our weakest area ([LIMITATIONS.md](docs/LIMITATIONS.md) §6). If
you know Indian financial-sector accounting and can tell us which of our `general` metrics
are quietly meaningless for a specific business type, that is high-value.

**5. Work on the open items** in [LIMITATIONS.md](docs/LIMITATIONS.md) §10.

---

## The bar for a factor proposal

We hold proposals to the same standard we hold ourselves, because we have been burned by
each of these:

**(a) Per-date cross-sectional rank IC, not pooled correlation.**
Pooled Pearson on our data says 12-month momentum has IC **−0.011** ("momentum is dead in
India"); per-date rank IC on the same data says **+0.047**. The pooled number is an outlier
artifact.

**(b) A per-year series, not a mean.**
A mean hides sign flips and decay. We rejected low-volatility precisely because its mean of
+0.031 concealed **5 positive and 5 negative years**.

**(c) Independent N, not observation count.**
Monthly observations of a 252-day forward return overlap ~11 neighbours. 42 monthly dates
over 3.5 years is **3–4 independent observations**, not 42. Overlap inflates information
ratio mechanically — we once reported IR 2.01 that way. Divide span by label horizon;
Newey-West for anything load-bearing.

**(d) A published long-short premium is not evidence for a pillar.**
This is the one most likely to trip up a well-read contributor. Betting Against Beta reports
29.1%/yr in India and NSE runs live indices on it — and it still failed here, because BAB is
a **beta-neutral leveraged long-short** whose alpha comes from the leverage and
neutralization. A 0–100 score is an **unlevered long-only cross-sectional ranking**. Only
factors with a monotone cross-sectional IC survive the translation.

**(e) Decile monotonicity on the MEDIAN.**
Our most recently adopted metric has non-monotone decile *means* — decile 1 has the highest
mean forward return with the worst median. A mean-return decile table makes the worst cohort
look best. Use medians.

**(f) Independence from what is already scored.**
Correlate against the existing metrics in the same pillar. And if your candidate correlates
with an existing one, use **within-decile residual IC** to establish which is redundant —
pairwise correlation cannot answer that. It told us to cut the opposite metric from the one
we expected.

**(g) Fit/holdout, if you are proposing a weight change.**
Non-negotiable. Our full-sample weight ranking **inverted completely** on holdout: the
scheme that looked best in-sample came fourth, and deleting "underperforming" pillars was
the worst thing we tested. A full-sample improvement is not evidence.

**(h) Sector and size concentration.**
Check whether your factor's top decile is really a sector bet. Ours was 3× over-weight
financials until we gated it.

If you cannot clear (a)–(f), open an issue anyway and say so — a well-posed hypothesis with
partial evidence is still useful. Just label it as a hypothesis.

---

## How to open a factor proposal

Use this shape. It maps onto how we will evaluate it:

```markdown
## Factor: <name>

**Definition:** exact formula, and what happens when the denominator is zero or negative.
**Pillar:** which one, and why it belongs there and not elsewhere.
**Direction:** higher-better or lower-better.
**Template applicability:** all / general-only / financials-only — and why.

### Measurement
- Universe + liquidity definition:
- Window (the range your query ACTUALLY returned):
- Label horizon:
- Independent N:
- Mean rank IC, range, and per-year table:
- Fit/holdout split:
- Decile medians:
- Correlation vs existing metrics in that pillar:
- Residual IC, if correlated with an existing metric:
- Sector/size concentration of the top decile:
```

---

## Reporting a defect in the spec

Open an issue with:

- The section of [docs/METHODOLOGY.md](docs/METHODOLOGY.md) at issue.
- What the spec says versus what it should say.
- The consequence — which stocks or which conditions get a wrong number, ideally with an
  example.

We are particularly interested in defects of the shape **"this produces a confidently wrong
answer with no error and no coverage gap"**, because those are the ones that survive review.
Every serious bug we have found in this model was invisible from aggregate statistics; two
of them showed *higher* confidence than correctly-handled stocks.

---

## What we will not accept

- **Buy/sell signals, price targets, entry/exit levels, stop losses, or allocation advice.**
  We are not a SEBI-registered research analyst, and none of that is going into this score
  regardless of how well it backtests. Do not propose renaming a band after a transaction
  either — "accumulation zone" is a buy call in costume.
- **A weight change justified by full-sample performance.** See (g).
- **Anything requiring a language model in the scoring path.** The score is deterministic
  data and math. That is a hard product constraint, not a technical limitation.
- **A factor that only works with look-ahead.** If it needs the latest restated figures on a
  historical date, it is not a factor.
- **Requests for our data sources.** We will not disclose our pipeline, and the issue will
  be closed. Every input is defined by *what it is*, which is sufficient to reimplement and
  to check.

---

## Process

1. **Open an issue first** for anything substantive. A factor proposal or a spec defect is a
   discussion before it is a diff — and we may already have measured it (check
   [CHANGELOG.md](docs/CHANGELOG.md) §"Things we tested and did NOT ship").
2. **PRs are welcome for documentation** — corrections, clarifications, worked examples,
   spec ambiguities. This repository is documentation, so a PR here changes the *spec*, not
   the running engine.
3. **A methodology change that we adopt ships with a formula-version bump** and an entry in
   [CHANGELOG.md](docs/CHANGELOG.md) crediting the contributor. Adopted changes go through
   our own blast-radius measurement first: how many stocks lose a pillar, how many cross the
   eligibility bar, and what the score distribution does.
4. **Timeline:** adopting a factor requires cutting a new reference version, which is a
   deliberate, infrequent operation. Expect a good proposal to be acknowledged quickly and
   shipped on the next version cut rather than immediately.

---

## Ground rules

- **Measurements beat arguments.** If we disagree, the resolution is a number, not a longer
  paragraph. We have been wrong against our own strong priors at least four times in this
  model's history and each time a measurement settled it.
- **A negative result is a contribution.** Telling us a factor we use does *not* work is as
  valuable as adding one — and easier to verify.
- **Cite the window.** "Momentum works in India" is not a claim; "IC +0.074, 2016–2025, 114
  monthly dates, liquid universe" is.
- Be civil, be specific, assume the other person read the docs.

Thanks for looking at this properly.
