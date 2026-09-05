# Requirements — Binance Trading Bot Optimiser

## Overview

### Purpose

This document specifies **what** the Binance Trading Bot Optimiser must do. It is technology-neutral except where the constitution has already fixed a choice; how these requirements are implemented is the concern of the solution specification.

The system is a **local, non-interactive Python CLI** that ingests Binance public historical market data into a local store, simulates Spot Grid and Futures Grid bot configurations against that history, and identifies configurations that would have performed well on a risk-adjusted basis — while explicitly discounting its own optimism.

### Scope

**In scope:**

- Automatic ingest of 1-minute klines from the Binance public data archive (spot and USD-M futures) and futures funding-rate history.
- Backtesting a single, fully specified grid bot configuration over a 7- or 30-day trailing window.
- Optimising over a derived parameter search space on a 30-day trailing window, using walk-forward validation.
- Reporting results as a human-readable table and as JSON on stdout.
- Inspecting the state of the local data store.

**Out of scope (per constitution — Explicit Non-Goals):**

- Any order execution, authenticated API access, or credential handling.
- Live or paper trading, position monitoring, or portfolio management.
- Charting, HTML reporting, or any graphical output.
- Strategy types other than Spot Grid and Futures Grid.

### Command Surface

Three commands: `backtest`, `search`, and `data status`. Data ingest is **implicit** — it happens automatically as a side effect of `backtest` and `search` when required data is absent. There is no standalone sync command and no results-retrieval command.

### What the Tool Endorses, and What It Merely Reports

The two analysis commands make categorically different claims, and the system must never blur them:

- **`search`** conducts a search and is therefore in a position to endorse — or to decline to endorse — its own output. Its endorsement is meaningful precisely because the tool also knows how large the search was and can discount it.
- **`backtest`** evaluates one configuration the Operator supplied. It measures; it does not select. There is no selection effect to correct for, no deflation to apply, and consequently **no basis on which the tool could endorse anything**. A backtest answers "what would this have done", never "should I run this".

This distinction is carried explicitly through the output contract (FR-023, FR-025) rather than left for a reader to infer, because a machine-readable endorsement attached to an unendorsable result is exactly the kind of false confidence the constitution's first principle forbids.

### Capital Versus Allocation Profile — a distinction this document maintains throughout

Two separate quantities were previously conflated under "capital allocation", and they are kept strictly apart here:

- **Total capital** — the amount of money the Operator intends to commit to the bot. This is always an **Operator-supplied input**. The tool has no business choosing how much money the Operator risks, and never sweeps it.
- **Allocation profile** — how that fixed total capital is distributed across the grid's levels. Binance implements exactly one profile, uniform per level: both the Spot and the Futures grid derive a single Qty/Order from the investment amount and place it at every level, and neither exposes any parameter that tilts capital toward one end of the range. It is therefore **not** a dimension the optimiser sweeps — a weighted profile would describe a configuration no Operator could enter into Binance's interface.

Every requirement below uses these two terms precisely and never interchangeably.

### The Objective Function — stated once, referenced throughout

The product's definition of "best" is fixed here so that every dependent requirement is verifiable:

- **Ranking measure: the annualised Sortino ratio.** Sortino rather than Sharpe because a grid bot's return distribution is deliberately asymmetric — many small realised gains punctuated by adverse inventory revaluation — and penalising upside dispersion would misrepresent it. Sortino rather than Calmar because the drawdown dimension is already governed by a hard gate (FR-018), and using a drawdown-based ratio as well would apply the same constraint twice with different strengths.
- **Return series: hourly mark-to-market equity returns.** Equity is cash plus open inventory valued at the prevailing close. Hourly sampling is chosen because per-bar 1-minute returns are dominated by microstructure noise, while daily sampling yields too few observations for a fold of a 30-day window to support a stable estimate.
- **Because equity is marked to market, periods containing no completed grid cycles are ordinary observations** and are included unchanged. They are not zero-return periods and require no special handling: inventory revaluation is real economic exposure and belongs in the risk denominator.
- **Minimum acceptable return (the downside threshold): zero.** Downside deviation is computed over observations below zero.
- **Risk-free rate: zero**, appropriate to the horizon and to crypto quote assets.
- **Annualisation: by the square root of the number of hourly periods in a year.**
- **Degenerate case:** where a configuration produces no observations below the downside threshold, downside deviation is zero and the ratio is undefined. Such a configuration is reported as having **insufficient downside observations to rank**, and is excluded from the ranking rather than treated as infinitely good. This is a real and common case over short windows and must not be allowed to promote a configuration that simply has not yet met adversity.

### Maximum Drawdown — stated once, referenced throughout

Maximum drawdown is the hard exclusion gate (FR-018), the basis of the derived kill threshold (FR-020), and a required output field (FR-024). Because it does structural work in three places, its computation is fixed here:

- **Series:** the same **hourly mark-to-market equity curve** that underlies the objective function. Intra-hour extremes are not used; the measure is consistent with the series everything else is computed over.
- **Definition:** the largest peak-to-trough decline in that curve, expressed as a **fraction of total capital**.
- **Two distinct measurements, used for two distinct purposes, and never interchanged:**
  - **Selection-gate drawdown** — computed on a candidate's **training-segment** equity curve within a fold. This is the measurement the exclusion gate applies during selection, because selection may only use training data.
  - **Reported drawdown** — computed on the **concatenated out-of-sample** equity curve. This is the figure reported to the Operator, the figure the drawdown kill threshold is derived from, and the figure the risk-ceiling check of FR-018 is applied to. It is generally larger than any single fold's drawdown, because consecutive declining folds compound into one deeper trough — and that compounded figure is the one that reflects what the Operator would actually have experienced.
- Every output presenting a drawdown states which of the two it is.

### The Validation Procedure — stated once, referenced throughout

Walk-forward validation admits two incompatible designs, and this project commits to one. **Selection happens independently within each fold, using training data only.** Candidates are never selected on the strength of their out-of-sample results, because doing so consumes the out-of-sample data as selection input and destroys the very property that makes it informative.

The procedure is:

1. The resolved 30-day window is divided into sequential, non-overlapping train/test fold pairs, in chronological order.
2. Within each fold, candidate configurations are evaluated on the **training** segment only, and the best-scoring candidate by the objective function — subject to the selection-gate drawdown exclusion — is selected as that fold's winner.
3. That fold's winner, and only that winner, is then simulated on the fold's **test** segment. Its hourly return series from the test segment is the fold's out-of-sample contribution.
4. The out-of-sample contributions of all folds are **concatenated in chronological order** into a single out-of-sample return series representing what the selection procedure would actually have earned.
5. **The headline figure is the annualised Sortino ratio of that concatenated series** — not the mean of the per-fold ratios, which would weight short and long folds equally and discard the compounding path.
6. **The reported configuration** is the fold winner selected in the greatest number of folds; ties are broken in favour of the most recent fold's winner, since it was selected on the most recent market conditions.

**No fold is ever discarded.** A fold whose test segment produces few completed cycles is not an invalid measurement — it is a real out-of-sample outcome, and usually the most informative one, because it is what a change of regime looks like. A configuration selected in calm training conditions that then meets a quiet or trending test segment stops cycling and stops earning; that is precisely the failure walk-forward exists to reveal. Excluding such folds would retain only the periods where the strategy happened to keep working and would inflate the headline by construction. Thin folds are therefore **counted and reported, never removed**, and their returns remain in the concatenated series.

**Selection stability is itself a result.** If the folds elect different winners, the procedure has not found a configuration that persists — it has found noise, and the disagreement is stronger evidence of overfitting than any single statistic. The system therefore reports how many distinct configurations won folds and in how many folds the reported configuration was selected, and warns when that falls below the selection consistency threshold.

**The risk ceiling binds the recommendation, not merely the selection.** Selection may only use training data, so a configuration can pass the selection gate and still breach the Operator's stated limit once measured out-of-sample. Where that happens, the tool must not recommend it (FR-018) — a strategy that already exceeded the declared ceiling in the only data that matters is not a candidate, however good its other statistics.

### Recorded Deviation from the Constitution

One requirement below diverges from the approved constitution and is recorded here rather than silently reconciled.

**Deviation D-001 — Result persistence.** Constitution *Integration Points → DuckDB* names DuckDB the system of record for "run metadata, trial counts, rejection counts, search seeds, applied constraints, derived invalidation thresholds and results." This specification instead makes **run results ephemeral** (FR-021): DuckDB stores market data and reference data only, and results exist solely in the command's stdout.

*Impact:* Constitution Principle 3 (trial counting is mandatory bookkeeping) and Principle 9 (reproducibility) are preserved through output rather than storage — every result carries its trial count, seed, applied constraints and reference-data version in both output formats, so a captured stdout is a complete and self-sufficient record. What is lost is automatic run history: a result not saved by the user is not recoverable, and cross-run comparison is manual.

*Required action:* the constitution's Integration Points section should be amended to scope DuckDB's role to market and reference data. Until amended, this specification is the authority on result persistence.

---

## Configuration Constants and Defaults

Every acceptance criterion in this document that refers to a named constant refers to this table. Values are provisional where marked and may be revised once measured against real data, but each has a stated value so that dependent criteria are verifiable today.

| Constant | Default value | Overridable by flag | Notes |
|---|---|---|---|
| **Total capital** | none — required input | `--capital` (required) | The Operator must state it; the system never assumes an amount. |
| **Maximum drawdown limit** | **20%** of total capital | `--max-drawdown` | The risk ceiling of constitution Principle 4. Applied as the selection-gate drawdown during selection, and re-checked against the reported drawdown before any recommendation (FR-018). |
| **Thin-fold cycle threshold** | **20** completed (matched) grid cycles in a fold's test segment | not overridable | A **reporting** threshold, not a discard gate. Folds below it are counted and reported; they are never removed (FR-017). |
| **Aggregate out-of-sample minimum cycles** | **60** completed cycles summed across all test segments | not overridable | Below this, the run reports insufficient out-of-sample evidence. This is the only cycle-count gate that suppresses a headline figure. Provisional pending measurement. |
| **Selection consistency threshold** | reported configuration must win at least **50%** of folds | not overridable | Below this, the run carries a selection-instability warning. |
| **Drawdown kill-threshold margin** | **5 percentage points** above the worst per-fold out-of-sample drawdown | not overridable | The derived threshold remains capped by the maximum drawdown limit. |
| **Default trial budget** | **2,000** configurations per optimisation run | `--max-trials` | Counted across all folds (FR-016). Provisional; to be tuned against measured runtime. |
| **Minimum deflation confidence** | **0.95** | not overridable | A winner must clear this to be presented as meaningful (FR-019). |
| **Equity sampling interval** | **1 hour** | not overridable | The return series underlying both the Sortino ratio and maximum drawdown. |
| **Expected bars per complete day** | **1,440** 1-minute bars | not overridable | The completeness test for a daily archive (FR-001, FR-004). |
| **Top-N results displayed** | **10** configurations | `--top` | Display only; does not affect the search or the trial count. |
| **Analysis window** | none — required input | `--days` (7 or 30 only) | `search` accepts 30 only (FR-015). |
| **Random seed** | system-generated and reported | `--seed` | Recorded in every result for reproducibility. |
| **Archive retry limit** | **3** attempts per file, with backoff | not overridable | Exhaustion is an operational error. |
| **Progress reporting interval** | at most **10 seconds** between updates | not overridable | See NFR-006. |

---

## User Roles

This is a single-user, local-machine tool. It has **no authentication, no authorisation, and no multi-user concerns**, and must not acquire them. Access control is delegated entirely to the operating system's file permissions on the user's data directory.

| Role | Description | Capabilities |
|---|---|---|
| **Operator** | The solo retail trader running Binance grid bots. The only runtime role. Technically capable of using a CLI and reading a results table; not a quantitative researcher. | Invoke all commands; supply flags; read stdout; own and manage the local data directory. |
| **Maintainer** | The developer extending the tool. Not a runtime role; listed because the strategy-registry requirement (FR-011) exists to serve them. | Add strategy types; adjust named constants; extend reference data. |

**RO-001.** The system must operate fully without any account, credential, licence key, or network identity beyond anonymous public HTTP access to the Binance data archive.

**RO-002.** All Operator inputs arrive as command-line flags or documented defaults. No command may prompt, and no command may block awaiting input.

---

## Functional Requirements

### Data Acquisition and Storage

#### FR-001 — Window resolution precedes all other work

Because the archive publishes each day's file with a lag, and may publish a file for a day still in progress, the end of a trailing window is a property of the archive's contents rather than of the local clock. Resolving it is an explicit first step, and it consults the local store before the network.

*Acceptance criteria:*

- The system establishes a **candidate end date** as the most recent date for which a daily archive is known to exist for the requested symbol and market type, determined from the local store's provenance record (DR-005) where available and by probing the archive otherwise.
- **The local store is consulted first.** If the candidate end date is already held locally and its recorded bar count equals the expected bars per complete day, it is accepted as complete and **no download occurs**.
- Only where the candidate end date is absent locally, or is held with a short bar count, does the system fetch it in order to establish completeness.
- If the candidate end date proves incomplete, the system steps the candidate back by one day and repeats the check, applying the same store-first rule at each step.
- The **resolved end date** is the most recent date established complete. The **resolved window** is the requested number of days ending at, and including, it.
- The resolved window's start and end dates are reported to the Operator before analysis begins, and appear in the result output (FR-024), so a captured result is unambiguous about which period it covers.
- A not-yet-published or still-accruing trailing day shortens nothing and triggers no error — it moves the resolved end date earlier.
- If the archive holds fewer complete days than the requested window requires, the system refuses to run and reports how many are available.
- The resolved end date is derived solely from archive availability and content, never from the local system clock.

#### FR-002 — Implicit, incremental data acquisition

Once the window is resolved, the system acquires exactly the data it lacks.

*Acceptance criteria:*

- The system determines the market data required for the resolved window, identifies which of it is absent from the local store, downloads only the absent portions, and ingests them before analysis begins.
- Given an empty store and a resolved 30-day window, the system downloads 30 daily kline archives, plus funding-rate data when the market type is futures.
- **Given a store that already covers the resolved window in full, including its end date, the run performs zero downloads end to end** — window resolution included.
- Given a store containing 25 of the 30 required days, the system downloads exactly the 5 absent days.
- Ingest progress is reported as it proceeds (NFR-006).
- No command exposes ingest as a separate user-invoked step.

#### FR-003 — Archive integrity verification

Every downloaded archive is verified against its published checksum before its contents are ingested.

*Acceptance criteria:*

- A downloaded archive whose computed checksum matches the published value is ingested.
- A mismatch aborts ingest of that file, writes nothing to the store, and reports the symbol, date and both checksum values.
- A checksum failure is an operational error (FR-025), not a research finding.
- Archive contents are validated before extraction: entries with absolute paths, parent-directory traversal, unexpected member names, or implausible uncompressed sizes are rejected without being written to disk.
- Downloads are retried up to the archive retry limit with backoff on transient failure; exhaustion is an operational error.

#### FR-004 — Interior data coverage gates the run

Having resolved the window, the system refuses to analyse it unless it is completely covered.

*Acceptance criteria:*

- Before analysis, the system verifies that every date **within the resolved window** is present in the store and that each contains the expected bars per complete day.
- If any date interior to the resolved window is absent from the archive, or present but incomplete, the system **refuses to run**, lists the specific dates, and exits as an operational error.
- This requirement governs gaps interior to the resolved window only; the trailing boundary is handled by FR-001 and can never present as an interior gap.
- The system never interpolates, forward-fills, or synthesises absent bars.
- The system never silently shortens the resolved window to fit available data.
- There is no flag to override this refusal.

#### FR-005 — Immutable stored market data

Ingested market data is append-only.

*Acceptance criteria:*

- Re-ingesting a date already present is a verified no-op when the source bytes are unchanged.
- Stored bars for a given symbol, market type and date are never modified in place by any command.
- Analysis of an identical window at any later time reads identical input bars.

#### FR-006 — `data status` command

The Operator can inspect what the local store contains without running an analysis.

*Acceptance criteria:*

- `data status` lists, per symbol and market type, the date range held and any gaps within it.
- The command performs no network access and no downloads; it does not resolve a window.
- The command runs successfully against an empty store, reporting that the store is empty.
- Output is available in both human-readable and JSON form (FR-022, FR-023).
- The command produces no performance metrics and no endorsement; its JSON recommendation field is **not applicable** (FR-023).

#### FR-007 — Data retention

Ingested market data is retained indefinitely.

*Acceptance criteria:*

- No command deletes, prunes, expires or archives stored market data.
- Store growth is bounded only by the Operator's own disk management.

### Simulation

#### FR-008 — Backtest a single configuration

`backtest` simulates one fully specified grid bot configuration over the resolved window and reports its performance. It measures; it does not endorse.

*Acceptance criteria:*

- Accepts as flags: symbol; market type (spot or futures); window (`--days`, 7 or 30); **total capital**; maximum drawdown; and the complete strategy parameter set — price range bounds, grid count, spacing mode, **allocation profile**, and for futures leverage and direction.
- Produces the performance metrics of FR-024 for that single configuration.
- Performs no parameter search and no fold division; the trial count of the run is exactly one, and no deflation is applied (FR-019 applies to searches).
- **Emits a recommendation value of "not applicable"** (FR-023), accompanied by a statement that no search was performed and therefore no endorsement is possible. A backtest never emits an affirmative recommendation, whatever its metrics show.
- Drawdown is computed over the full window's hourly equity curve and reported as such; the selection-gate/out-of-sample distinction does not arise, and the output says so.
- Where the computed drawdown exceeds the applied maximum drawdown limit, the output states this plainly. No downgrade applies, because there was no recommendation to downgrade.
- Rejects an incompletely specified configuration before any simulation begins, naming the missing parameters.

#### FR-009 — Spot Grid simulation semantics

The simulator reproduces Binance Spot Grid mechanics.

*Acceptance criteria:*

- Grid levels are derived from the configured lower bound, upper bound, grid count and spacing mode (arithmetic or geometric).
- Buy orders rest below the current price and sell orders above it; a filled buy places a corresponding sell one level up, and vice versa.
- The Operator's total capital is distributed across levels according to the configured allocation profile; the sum of allocated capital never exceeds total capital.
- Simulated fills occur at exactly the grid level price, with no slippage applied.
- No fill occurs at a price outside the configured range.
- When price moves beyond the configured range, the bot holds its inventory and places no orders outside the range.
- Every fill charges the applicable maker or taker fee.

#### FR-010 — Futures Grid simulation semantics

The simulator reproduces Binance Futures Grid mechanics for USD-M perpetuals.

*Acceptance criteria:*

- Supports long, short and neutral grid directions.
- Applies the configured leverage to position sizing and margin calculation, with total capital as the margin base.
- Applies funding payments at each funding interval, using the historical funding rate for that interval and the position held at that time, with the sign reflecting position direction.
- Computes a liquidation price from the position, leverage and margin, and terminates the simulation with a liquidation outcome if price reaches it.
- A liquidated run is reported as liquidated, with the bar at which it occurred, and is never reported as a merely poor-performing run.
- Every fill charges the applicable maker or taker fee.

#### FR-011 — Extensible strategy types

New bot types can be added without modifying the search, output or storage logic.

*Acceptance criteria:*

- Spot Grid and Futures Grid are registered implementations of a common strategy interface.
- The interface exposes: configuration schema, derived search space, order generation, and fill and accounting semantics.
- Adding a third strategy type requires no change to the optimisation loop, the CLI argument surface beyond a new type value, or the market-data schema.

#### FR-012 — No-lookahead simulation

No simulated decision uses information unavailable at the moment of that decision.

*Acceptance criteria:*

- Simulating a series truncated at bar *T* produces results for bars 0..*T* identical to simulating the full series (truncation equivalence).
- This property holds for every registered strategy type.
- No feature, statistic or normalisation used by the simulator is computed over data beyond the current bar.

#### FR-013 — Conservative intrabar fill resolution

Where 1-minute OHLC data cannot determine the order in which prices occurred within a bar, the least favourable feasible ordering is assumed.

*Acceptance criteria:*

- Where a bar's range spans both a buy level and a sell level and the intrabar path is ambiguous, the ordering less favourable to the strategy is applied.
- For a futures position, if a bar's range reaches both a liquidation price and a favourable exit level, liquidation is assumed to occur.
- Each such resolution is deterministic and identical across runs.

### Optimisation

#### FR-014 — Optimise over a derived search space

`search` searches for the best configuration over a resolved 30-day window, deriving the parameter search space from the observed market data.

*Acceptance criteria:*

- Accepts as flags: symbol; market type; window (`--days 30`); **total capital**; maximum drawdown; trial budget; seed; and top-N.
- **Does not** accept per-parameter search bounds; the Operator cannot narrow or widen individual dimensions.
- The search space covers price range bounds, grid count, spacing mode, **allocation profile**, and — for futures — leverage and direction. **Total capital is never a search dimension**; it is held fixed at the Operator-supplied value for every trial.
- The search space is derived from the resolved window's observed price range and realised volatility, computed from **training data only** within each fold, never from the full window.
- The derived search space is reported as part of the result, so the search is inspectable after the fact.
- The derivation is deterministic: identical input data yields an identical search space.

#### FR-015 — Optimisation requires a 30-day window

*Acceptance criteria:*

- `search --days 30` proceeds.
- `search --days 7` is **refused** with an explanation that a 7-day window cannot support viable walk-forward folds, and a suggestion to use `backtest` for that window.
- The refusal never degrades into an in-sample sweep over 7 days.
- The refusal is an input-validation error, distinct from both a research finding and a data error.

#### FR-016 — Bounded search with mandatory trial accounting

Every optimisation run has a declared trial budget and counts every configuration it evaluates.

*Acceptance criteria:*

- A maximum trial budget is established before the search begins, from the default trial budget or the overriding flag.
- The budget governs the **total** number of configuration evaluations across all folds, not the number per fold.
- The search never evaluates more configurations than the budget permits.
- Every evaluated configuration increments the trial counter, including those eliminated in early stages of a staged or coarse-to-fine search, and including the same configuration evaluated again in a different fold.
- Before a long run begins, the system reports the planned trial count and an estimated duration.
- The final trial count is reported with the result, and is the input to deflation under FR-019.

#### FR-017 — Walk-forward validation and out-of-sample sufficiency

Optimisation is validated out-of-sample according to *The Validation Procedure*, and the out-of-sample result is the headline. **No fold is discarded on the basis of its outcome.**

*Acceptance criteria:*

- The resolved 30-day window is divided into sequential, non-overlapping train/test fold pairs in chronological order.
- Within each fold, candidates are scored on the **training** segment only; the fold's winner is the best-scoring candidate on that segment that also passes the selection-gate drawdown exclusion.
- Only the fold's winner is simulated on the fold's **test** segment; no candidate is ever selected using test-segment results.
- **Every fold's out-of-sample contribution is included in the concatenated series, regardless of how many completed cycles its test segment produced.** A fold in which the winner stopped cycling is a valid and informative out-of-sample outcome, not a measurement failure.
- A fold whose test segment yields fewer completed cycles than the thin-fold cycle threshold is marked **thin**. Thin folds are counted and reported (FR-024); they are never removed from the series and never excluded from any statistic.
- The run reports **"insufficient out-of-sample evidence"** and emits **no** headline performance figure only when the **total** completed cycles summed across all test segments falls below the aggregate out-of-sample minimum cycles.
- Where a headline figure is emitted, it is the annualised Sortino ratio of the full concatenated out-of-sample series.
- Training-segment figures, where shown, always appear alongside their out-of-sample counterparts and are never presented as the headline.

#### FR-018 — Selection, the drawdown gate, and the risk-ceiling check

*Acceptance criteria — selection:*

- Within a fold, candidates are ranked by the annualised Sortino ratio as defined in *The Objective Function*, computed on the training segment, never by absolute return.
- A candidate with no observations below the downside threshold is excluded from selection and reported as having insufficient downside observations to rank.
- A candidate whose **selection-gate drawdown** exceeds the applied maximum drawdown limit is **absent from selection entirely** — it cannot win a fold at any position, regardless of its return.
- The applied limit — whether supplied by flag or taken from the default — is reported with every result.
- The **reported configuration** is the fold winner selected in the greatest number of folds, ties broken toward the most recent fold's winner.

*Acceptance criteria — the risk-ceiling check:*

- After the reported configuration is determined, the system compares its **reported (concatenated out-of-sample) drawdown** against the applied maximum drawdown limit.
- Where the reported drawdown **exceeds** the applied limit, the result is **downgraded from a recommended candidate to a research finding**. It is reported in full — with the observed out-of-sample drawdown and the applied limit presented side by side — but the tool **explicitly does not recommend it for a live trial, regardless of its deflated confidence** (FR-019) or any other statistic.
- The downgrade message states plainly that the configuration breached the Operator's stated risk ceiling in the out-of-sample period, and that selection could not have prevented this because selection may only use training data.
- This check exists because the applied limit is the project's single risk ceiling: passing a training-data gate does not entitle a configuration to be recommended when the out-of-sample evidence already shows the ceiling being breached.
- A downgraded result exits as **success**, as all research findings do (FR-025).

*Acceptance criteria — stability reporting:*

- The result reports **selection stability**: the number of distinct configurations that won folds, and the number of folds in which the reported configuration was selected.
- Where the reported configuration wins fewer folds than the selection consistency threshold, the result carries an explicit **selection-instability warning** stating that the procedure did not find a configuration that persisted across the window.

*Acceptance criteria — the empty result:*

- When no configuration survives the gates in any fold, the system reports: the number of trials evaluated; the number rejected by the drawdown gate and the number rejected for insufficient downside observations, separately; the best selection-gate drawdown actually achieved against the applied limit; and a concrete next action, including a specific relaxed drawdown value that would have admitted at least one candidate.
- "No viable configuration" exits as **success** and is distinguishable in JSON output from any operational error.

#### FR-019 — Trial-count-aware scoring

The winner of a search of *N* configurations is not comparable to a single tested configuration, and the system must say so quantitatively rather than leaving the Operator to discount the number themselves.

*Acceptance criteria:*

- For every optimisation run, the system computes a **deflated statistic** for the reported configuration that adjusts the headline out-of-sample ratio for: the total number of trials evaluated across all folds (FR-016), the length of the concatenated out-of-sample series, and the skewness and kurtosis of its return distribution.
- The deflated statistic is expressed as a **confidence that the configuration's true risk-adjusted performance exceeds zero**, given the size of the search that produced it.
- **The deflated value, not the raw ratio, governs how the result is presented.** Where the deflated confidence meets or exceeds the minimum deflation confidence, the result is presented as a candidate worth a minimum-size live trial. Where it does not, the result is presented as **"not distinguishable from the outcome of searching this many configurations"**, and the tool explicitly does not recommend it.
- **Clearing the deflation threshold is necessary but not sufficient for a recommendation.** A result downgraded by the risk-ceiling check of FR-018 is never recommended, whatever its deflated confidence. Where both conditions fail, both are reported.
- Both the raw ratio and the deflated confidence appear in both output formats, adjacent to one another; the raw ratio never appears without its deflated companion.
- A failure to clear the deflation threshold is a **research finding**, not an error, and exits as success (FR-025).
- The trial count used in deflation is the full count of configurations evaluated, including those eliminated early and those re-evaluated in later folds — a search that discards candidates cheaply is not thereby made more credible.
- Deflation does not apply to `backtest`, where exactly one configuration is evaluated and there is no selection effect to correct for. A single backtest's output states that no deflation was applied and why.

#### FR-020 — Derived invalidation thresholds

Every result carrying performance metrics also carries a pre-declared kill criterion, computed by the system.

*Acceptance criteria:*

- Each result includes a **drawdown kill threshold**, computed as the tighter of (a) the worst **per-fold out-of-sample** drawdown plus the drawdown kill-threshold margin and (b) the applied maximum drawdown limit.
- Each result includes a **minimum grid-cycle rate**, below which the configuration should be considered invalidated.
- Both thresholds appear in both output formats, including on results downgraded by the risk-ceiling check.
- No code path prompts the Operator for these values.
- Where flags override the derived values, the result records that the value was supplied rather than derived.
- An override looser than the applied maximum drawdown limit is **rejected with an explanation**, not honoured.
- Where clause (b) was the binding constraint, the output states that the threshold was capped by the risk limit.

### Output

#### FR-021 — Results are ephemeral

Run results are emitted to stdout and are not persisted. *(See Deviation D-001.)*

*Acceptance criteria:*

- No command writes run results, trial counts or rankings to the local store.
- The local store contains market data and reference data only.
- Each result is self-sufficient: captured stdout contains everything needed to interpret and reproduce the run.

#### FR-022 — Human-readable output

*Acceptance criteria:*

- Default output is a plain-text table suitable for a terminal, with no colour or cursor control required for correctness.
- For `search`, the top-N configurations by fold-selection frequency are shown, N taken from the top-N constant or its flag.
- Where a result is downgraded or carries a warning, that statement appears before the metrics table, not buried beneath it.
- A `backtest` result states in plain language that the tool makes no recommendation because no search was performed.
- Output is stable across runs with identical inputs, making it diffable.

#### FR-023 — JSON output and the recommendation contract

*Acceptance criteria:*

- A flag emits the complete result as JSON on stdout.
- The JSON contains every field present in the human-readable output, plus all trust-context and provenance fields.
- When JSON output is selected, no non-JSON text is written to stdout; progress and diagnostics go to stderr.
- The JSON distinguishes the three terminal outcomes, and within research findings distinguishes the specific cause: no viable configuration; insufficient out-of-sample evidence; not distinguishable from search noise; or out-of-sample drawdown exceeding the applied risk ceiling.
- **The recommendation field is tri-state**, and is the single authoritative signal of the tool's stance:

| Value | Meaning | Emitted by |
|---|---|---|
| `recommended` | The search produced a configuration clearing both the deflation threshold and the risk-ceiling check; a minimum-size live trial is warranted. | `search` only |
| `not_recommended` | A search was performed and the tool declines to endorse its outcome. The accompanying cause states why. | `search` only |
| `not_applicable` | No search was performed, so no endorsement is possible. | `backtest`, `data status` |

- A consuming script must be able to determine the tool's stance from this field alone, without inferring it from the presence or absence of warnings.
- `backtest` **never** emits `recommended` or `not_recommended`. Its metrics may be excellent or terrible; neither entitles the tool to an opinion, because the Operator chose the configuration and the tool merely measured it.

#### FR-024 — Required result content

Every result carrying performance metrics contains the following.

*Acceptance criteria — provenance:*

- The resolved window's start and end dates (FR-001), the symbol and the market type.
- The version of the fee and contract reference data applied.
- The random seed, where the search used sampling.

*Acceptance criteria — performance:*

- The out-of-sample annualised Sortino ratio of the concatenated fold series (the headline), with its deflated confidence adjacent (FR-019). For `backtest`, the whole-window ratio with an explicit note that no deflation was applied.
- Net PnL after all modelled costs.
- The maximum drawdown, explicitly labelled as reported (concatenated out-of-sample) or whole-window as applicable, alongside the applied maximum drawdown limit for direct comparison.
- Count of completed (matched) grid cycles.
- Total fees paid, and for futures, total funding paid or received.

*Acceptance criteria — configuration:*

- The configuration's full parameter set — price range bounds, grid count, spacing mode, allocation profile, and for futures leverage and direction — expressed in terms the Operator can enter directly into the Binance grid bot interface.
- The total capital the result was computed against.

*Acceptance criteria — trust context (search only):*

- Number of trials evaluated across all folds.
- The applied maximum drawdown limit and any other applied constraints.
- Selection stability: the number of distinct fold winners and the number of folds won by the reported configuration (FR-018).
- **The number of folds marked thin, out of the total fold count** (FR-017), so the Operator can see how much of the out-of-sample period the strategy spent not cycling.
- The divergence between the training-segment and out-of-sample results for the reported configuration.
- The derived search space that was searched.

*Acceptance criteria — invalidation:*

- The derived drawdown kill threshold and minimum grid-cycle rate (FR-020).

*Acceptance criteria — disclosure:*

- A statement that the simulated fill rate is unverified, that queue position is not modelled, and that live results may therefore fall short.
- A recommendation to run the configuration at minimum size before scaling — present only where the recommendation field is `recommended`.
- A warning that the sample window is short and covers few market regimes.

#### FR-025 — Terminal outcome semantics

The system distinguishes three kinds of terminal outcome, and never conflates them.

*Acceptance criteria:*

| Outcome | Examples | Exit status | Recommendation field | Representation |
|---|---|---|---|---|
| **Result — endorsed** | An search run whose reported configuration clears both the deflation threshold and the risk-ceiling check. | Success | `recommended` | Full result payload. |
| **Result — measured only** | A completed backtest; a `data status` listing. | Success | `not_applicable` | Full payload, with an explicit statement that no endorsement is possible. |
| **Research finding** | No viable configuration; insufficient out-of-sample evidence; reported configuration not distinguishable from search noise; reported configuration breaching the applied risk ceiling out-of-sample. | Success | `not_recommended` | Explained finding with diagnostics, per FR-017, FR-018 and FR-019. |
| **Operational error** | Interior missing archive day; insufficient archive history for the requested window; checksum failure; corrupt archive; unwritable data directory; invalid flags; 7-day search request. | Failure | absent | Actionable error naming what failed, for which symbol and date, and what to do next. |

- A research finding never exits as a failure.
- An operational error never exits as a success.
- A selection-instability warning or a thin-fold count accompanies a result; neither by itself converts an endorsed result into a research finding.
- A breach of the applied risk ceiling by the reported out-of-sample drawdown **does** convert an endorsed result into a research finding (FR-018).
- A backtest is never an endorsed result, regardless of its metrics.
- Exit statuses are documented and stable, so the tool is scriptable.

---

## Non-Functional Requirements

### NFR-001 — Ingest throughput

A first-time 30-day, single-symbol ingest completes within **3 minutes** on a 20 Mbit/s connection, excluding time attributable to archive-side rate limiting.

*Measurement:* timed cold-store ingest of 30 daily 1-minute kline archives for a liquid symbol, on a connection metered at 20 Mbit/s. Measured at least once per release on reference hardware.

### NFR-002 — Peak memory ceiling

Peak resident memory does not exceed **1 GB** for any supported operation, including a default-budget 30-day optimisation run.

*Measurement:* peak RSS sampled during a default-budget `search` run over a 30-day window, and during a cold 30-day ingest. Both must remain under the ceiling on a machine with 8 GB RAM.

### NFR-003 — Disk footprint

Stored market data occupies no more than **30 MB per symbol-month** of 1-minute klines in the local store, and the total footprint including any retained downloaded archives does not exceed **90 MB per symbol-month**.

*Measurement:* store size measured after ingesting one calendar month of 1-minute data for a single symbol into an empty store.

### NFR-004 — Determinism and reproducibility

Identical commands over identical stored data produce byte-identical output.

*Measurement:* the same command executed twice against an unchanged store yields identical JSON output, excluding fields explicitly documented as timing-related. A run re-executed with a recorded seed reproduces the same search, the same fold winners, the same reported configuration, the same deflated statistic and the same recommendation value.

### NFR-005 — Zero-credential operation

The system never requires, accepts, prompts for, or stores any Binance credential.

*Measurement:* the dependency tree contains no authenticated Binance client and no credential-storage library; no code path reads a credential from environment, file or argument; all network destinations resolve to the public data archive.

### NFR-006 — Progress visibility

No operation exceeds the progress reporting interval (**10 seconds**) without reporting progress.

*Measurement:* during a cold 30-day ingest and a default-budget optimisation run, the interval between successive progress updates never exceeds 10 seconds. Progress is written to stderr so it does not contaminate JSON output.

### NFR-007 — Numeric exactness in accounting

No rounding error is introduced into fill, fee, funding or PnL accumulation.

*Measurement:* a scenario producing a large number of fills yields a final balance exactly equal to the independently computed sum of its components, with zero tolerance. Statistical metrics computed downstream of settlement — including the Sortino ratio, maximum drawdown and the deflated statistic — are exempt.

### NFR-008 — Locality and containment

All state written by the system resides within the Operator-specified data directory.

*Measurement:* a full ingest-and-search cycle writes no file outside the configured data root, verified by filesystem monitoring during the run.

### NFR-009 — Portability

The system runs on Linux and macOS with a supported Python version, with no compiled dependency requiring a toolchain the Operator must install separately.

*Measurement:* the full test suite passes on both platforms in CI, from a clean environment created by the project's lockfile.

---

## Data Requirements

### DR-001 — Market data: 1-minute klines

Per symbol and market type (spot, USD-M futures), per 1-minute bar: open time, open, high, low, close, base-asset volume, quote-asset volume, trade count, and close time.

- Prices and volumes are stored at a precision sufficient to represent the source values exactly.
- Bars are uniquely identified by symbol, market type and open time; duplicates are impossible.
- Bars are stored in an ordered, queryable form permitting efficient extraction of a contiguous time range for a single symbol.

### DR-002 — Funding rate history

Per USD-M futures symbol: funding timestamp and funding rate, for every funding interval within any ingested period.

- Required before any Futures Grid simulation may run; its absence within the resolved window is a coverage failure under FR-004.

### DR-003 — Fee and contract reference data

Maker and taker fee rates, funding intervals, contract specifications, tick sizes and minimum order sizes for every symbol a run is priced on.

- For the packaged symbol set: versioned within the project and not fetched at runtime.
- For any other symbol the venue lists: resolved once from the venue's public metadata at first use, validated exactly as a packaged row is, cached beside the data store, and read from that cache on every later run. The fee schedule is never fetched (the venue publishes none); it is packaged and applied to both tiers.
- The applicable version is recorded in every result (FR-024) — the packaged version, or `live:<date>` naming the day the venue served the constraints, re-read from the cache so a rerun reproduces the fetching run. A captured result therefore remains interpretable after the schedules change.
- Where a symbol is priced under a promotional rather than a standard fee schedule, the run states that in its own output. A promotion that runs "until further notice" can be withdrawn, and a result produced under one does not reproduce at standard rates; the recorded version alone identifies the schedule but does not tell the Operator it was temporary.

### DR-004 — Supported symbol scope

The system supports a defined packaged set of popular symbols across spot and USD-M futures markets, plus any other symbol the venue lists on either market.

- The packaged set spans both USDT- and USDC-quoted pairs for each packaged base asset. The two are distinct instruments with distinct venue constraints and fee schedules, not aliases of one another.
- The packaged set is explicit and discoverable by the Operator without network access; the cached set of resolved symbols is listed beside it; the venue's own listing is discoverable on request.
- A request for a symbol the venue does not list, or lists but whose constraints cannot be modelled faithfully, fails as an operational error naming the venue and the listing command — before any archive fetch is attempted.

### DR-005 — Data provenance

For each ingested day: the source URL, the verified checksum, the observed bar count, and the ingest timestamp are retained.

- The recorded bar count is what allows FR-001 to resolve a window without network access when the data is already held.
- This supports the immutability guarantee (FR-005), the completeness tests of FR-001 and FR-004, and verification that stored data matches the archive.

### DR-006 — No result data

No run results, rankings, trial counts, deflated statistics or derived thresholds are stored. *(See Deviation D-001 and FR-021.)*

### DR-007 — Data classification

All data handled by the system is **public market data**. No personal data, no credentials, no financial account data and no personally identifying information enter the system at any point. There are consequently no retention, erasure or data-subject obligations.

---

## Integration Requirements

### IR-001 — Binance public market data (`data.binance.vision`, `fapi.binance.com`)

Two external systems, both Binance public market-data endpoints, and the only network destinations. Every other host is refused.

**`data.binance.vision`** — bulk historical archives:

- Probe for the existence of daily archives to establish a candidate end date, per symbol and market type, where the local store cannot already answer the question (FR-001).
- Retrieve daily 1-minute spot kline archives per symbol and date.
- Retrieve daily 1-minute USD-M futures kline archives per symbol and date.
- Retrieve the published checksum accompanying each archive.

**`fapi.binance.com`** — funding-rate history for USD-M futures symbols, via `GET /fapi/v1/fundingRate`.

- The archive publishes funding in monthly files that appear only after a month has ended. Sourcing funding from it capped every futures window at the last day of the previous month, discarding up to a month of history the venue was already serving. This endpoint publishes each settlement as it happens, so a futures window is bounded by kline availability exactly as a spot window is.

*Requirements:*

- Access to both is anonymous and read-only; no authentication is presented. Neither endpoint accepts or requires a credential.
- The permitted set is an explicit allowlist matched by equality, never a host-suffix pattern: the authenticated Binance trading routes share the `binance.com` suffix, and a pattern match would admit them.
- Requests are rate-limited and retried up to the archive retry limit with backoff; exhaustion is reported as an operational error.
- Every response is treated as untrusted input and structurally validated before ingest. **Archive** responses are additionally verified against their published checksum and defensively extracted. **API** responses have no published checksum to verify against; integrity for them rests on HTTPS and field-by-field validation, which is a genuinely weaker guarantee and is recorded here rather than left implicit.
- An archive that does not exist for a date interior to the resolved window is reported as a specific missing date (FR-004), never treated as an empty result. Funding that does not span the resolved window is reported the same way.
- Network access occurs during window resolution and data acquisition only; no other phase of any command performs outbound requests.

### IR-002 — Local analytical store

An embedded, file-based store within the Operator's data directory holding market data, funding rates and provenance.

*Requirements:*

- No network listener, no server process, no exposed port.
- Schema changes are explicit and preserve previously ingested data.
- Concurrent invocations must not corrupt the store; either concurrent access is safely supported or a second invocation fails cleanly with an explanatory message.

### IR-003 — Binance grid bot parameter compatibility

Configurations the system reports must be directly usable in the Binance grid bot interface.

*Requirements:*

- Parameter names and semantics correspond to the Binance product: price range bounds, grid count, arithmetic or geometric spacing, allocation profile, and for futures leverage and direction.
- Reported values respect the symbol's tick size, minimum order size and permitted leverage range.
- The reported allocation profile, applied to the Operator's stated total capital, must produce per-level order sizes at or above the symbol's minimum order size; a configuration that would produce sub-minimum orders is never reported as a winner.
- A configuration that could not be entered into the Binance interface is never reported as a winner, regardless of simulated performance.

### IR-004 — Consuming systems

The CLI's JSON output on stdout is a supported integration surface for the Operator's own scripts.

*Requirements:*

- The JSON structure is documented and versioned; a version identifier is present in every payload.
- Exit statuses are stable and documented (FR-025), so callers can distinguish a research finding from an operational failure without parsing text.
- The tri-state recommendation field (FR-023) is the single authoritative signal of whether the tool endorses a configuration, and callers must not infer endorsement from metrics alone.
- Diagnostics and progress never appear on stdout when JSON output is selected.
