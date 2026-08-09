# Update Cycle

How often the Tapetide Score changes, what a score date means, and what happens when a run
goes wrong.

This document covers **cadence and semantics**. It does not describe our data pipeline —
which feeds we run, how they are collected, or how they are stored. That is deliberate; see
the README.

---

## Cadence

**Once per trading day.**

Scores are recomputed each evening after the day's market data and fundamentals are
consolidated, typically around **19:45 IST**. The recompute is chained to the completion of
the upstream data consolidation rather than fired on a fixed independent timer — a fixed
offset would race a slow consolidation and score against a half-written input.

There is no intraday recomputation. A score is a daily observation.

---

## What a score date means

Every score is stamped with the **latest completed trading date**, not the wall-clock date
of the run.

Consequences worth understanding:

- **Weekends and market holidays reuse the previous session's score.** We do not create
  rows for non-trading days. A score you read on Sunday is Friday's score, correctly
  labelled Friday.
- **The date is the date of the data**, so a score dated two days ago means the run for
  yesterday did not publish — which is information, not a bug (see below).
- Any surface displaying a score should display its score date alongside. Printing today's
  date over an older score misrepresents it.

Roughly **4,500 stocks** are scored per run.

---

## Partial-run protection

A daily batch can fail partway through. Left unguarded, that produces the worst possible
outcome: a run that looks successful, publishes 40% of the universe, and silently drops
everything else out of every ranking.

This is not hypothetical. It happened: a run wrote **1,798 rows against a prior-day 4,440**,
and the ranking surface served 1,709 stocks instead of 4,256 — **2,547 companies silently
missing, including a top-10 bank** — with no error raised anywhere. Single-stock lookups were
unaffected, because each takes that stock's own most recent row, which is exactly why the
symptom was invisible except on rankings.

Two independent guards now exist, and they are deliberately redundant.

### Writer-side: refuse to publish a collapsed run

Before publishing, a run compares its row count against the **median** of the trailing 5
published runs and refuses to publish below **70%** of that baseline.

- **Median, not mean**, so one already-truncated day inside the window cannot drag the
  floor down and wave the next truncated day through.
- **Ceiling arithmetic, integer**, so the writer's floor can never sit fractionally below a
  true 70% that the reader would then reject.
- **A zero-row run is refused outright**, not excused by an absent baseline.
- Writes are atomic — a delete-then-insert inside one transaction with a per-date advisory
  lock, plus a post-write count verification. An earlier chunked write had no transaction at
  all, so a failure on a late chunk left a partial day that cleared the reader's bar.

Calibration: the truncated run above sat at **40%** of its trailing median, while every
healthy run in the surrounding window sat at **99–101%**. 70% leaves ample room for
legitimate universe drift (delistings, liquidity-gate churn) while catching a collapse of
that magnitude.

### Reader-side: rank only the latest COMPLETE date

Ranking surfaces select the latest date whose row count clears the same 70%-of-trailing-
median bar — not simply `max(date)`.

So if a run is refused or partially lands, **rankings degrade to the previous full picture**
rather than publishing a 40% ranking. The user sees yesterday's complete board with
yesterday's date on it.

Both thresholds come from **one shared constant** used by the writer and the reader. That
matters in both directions: a looser reader would rank a run the writer refused, and a
stricter reader would hide a run the writer published — silently freezing scores on an
older date with no error anywhere.

### A complete input does not imply a complete run

Worth stating because it defeated an earlier round of guards. The truncated run above had
**more complete price data than the previous day** — the inputs were fine; the run itself
died partway. Input completeness and output completeness must be verified independently.

Conversely, a recompute must never be forced when the day's price data is itself
incomplete. Observed: a day sitting at ~53% of the liquid universe versus ~94% the day
before. Recomputing then would have published a half-covered score set and flipped
thousands of stocks to "no score". Leaving yesterday's complete scores live is the correct
holding state — it is not a user-facing outage, because rankings and lookups keep serving.

**We never synthesize or carry forward a fabricated input to fill a gap.** A clean
documented gap is always better than invented data.

---

## Day-to-day stability

Scores are EMA-smoothed with a hysteresis dead band
([METHODOLOGY.md](METHODOLOGY.md) §11). Measured on live consecutive runs:

| Date | Stocks compared | Unchanged | Mean abs. move | Max move |
|---|---:|---:|---:|---:|
| Day 1 | 4,420 | 4,325 | 0.055 | 5 |
| Day 2 | 4,484 | 4,271 | 0.121 | 6 |
| Day 3 | 4,484 | 4,406 | 0.038 | 3 |
| Day 4 | 4,489 | 4,439 | 0.024 | 7 |

So on a typical day **97–98% of stocks publish an unchanged score** and the average
absolute movement is well under a tenth of a point.

**This is the point of smoothing: a move you can see is a move that means something.** If a
score jumps several points, the inputs genuinely changed or a red flag became active — it
is not daily noise.

---

## Reference versioning

The frozen peer distribution ([METHODOLOGY.md](METHODOLOGY.md) §3) has its own lifecycle,
independent of the daily cadence.

- A reference version is **written once and never modified**. Published scores pin the
  version they were scored against, so mutating one would silently change the baseline every
  historical score was measured on, with no way to recover the old statistics.
- Adding or removing a metric therefore requires **cutting a new version**.
- A run resolves its version by asking which version was **effective on or before the score
  date** — never by deriving a name from the date.

That last rule is written in blood. Deriving the name arithmetically (`ref_<year>`) caused a
live regression: a newly activated metric was computed into the daily inputs for ~1,900
stocks every night and then **silently discarded at scoring time**, because the unattended
job kept selecting the previous year-named version, which had no cell for that metric. One
day of the correct baseline sat sandwiched between two days of the old one, with no error
and no log line. Resolving as-of the score date also means a backfill of an older date is
never scored against a version that did not exist then — which would be look-ahead bias in
the calibration itself.

A new metric therefore ships **inert**: present in the inputs, ignored by scoring, until a
reference version containing it is cut and becomes effective. That is intentional
fail-safe behaviour, but it must be *visible* — dropped metrics are reported per run with a
share-of-universe figure, which is what distinguishes "deliberately inert new metric"
(~100% of universe) from "a reference rebuild dropped rows" (a count on a metric that
scored yesterday).

---

## Formula versioning

Any behavioural change bumps the formula version: metric membership, weights, minimum
metric bars, the sigmoid or EMA constants, governance penalties, or hard caps.

This is load-bearing rather than cosmetic. The reproducibility stamp is
`hash(formula_version | ref_version | inputs)`, so leaving the version unchanged after a
behavioural change would let two contradictory scores appear **byte-identical** in the
history, destroying the audit trail that the determinism claim rests on.

The first recompute after a version bump legitimately rewrites the formula version and
input hash for every stock. See [CHANGELOG.md](CHANGELOG.md).
