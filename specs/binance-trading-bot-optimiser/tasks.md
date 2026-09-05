# Implementation Tasks — Binance Trading Bot Optimiser

## How to Read This Plan

**Sequencing.** Tasks are ordered bottom-up by layer: infrastructure → domain foundations → storage → ingest → simulation → analytics → a working `backtest` vertical slice → optimisation → search reporting/CLI → integration → calibration → documentation → release. Each layer is complete and tested before the layer above it begins, so no task depends on an unbuilt abstraction.

**The vertical slice matters.** Milestone 7 delivers a fully working `backtest` command — renderer, JSON envelope, exit codes, disclosures — immediately after the analytics layer and before any optimisation code exists. This exercises the output contract end to end at the earliest point it can be exercised, rather than discovering contract problems after the whole search stack is built. The reporting tasks anchor on the `RunOutcome` union rather than on each other, so the text renderer, JSON envelope and exit-code mapping can proceed in parallel.

**Granularity.** A task is a unit of work that fits within four hours, ending in something demonstrable and independently testable. Complexity markers are calibrated to that scale: Small is under an hour, Medium one to two hours, Large two to four. Nothing in this plan is sized above Large; where a capability could not be expressed in four hours it has been split into subtasks with their own acceptance criteria.

**Definition of done.** A task is done when its code exists and its specified tests pass, including any guardrail test named in the solution design.

**Metadata.** Every task carries dependencies, traceability to requirement identifiers, and a complexity marker. Tasks where a mistake is expensive or hard to detect carry an explicit `_Risk:_` line.

**Milestones** below are organisational groupings only; they carry no separate exit gate.

**Provisional constants.** The trial budget, the fold-viability and aggregate-cycle thresholds, and the deflation gate are all marked provisional in the requirements and solution. Milestone 11 measures them against real data and revises them deliberately, rather than leaving guessed values to harden into product behaviour.

---

## Milestone 1 — Infrastructure and Project Setup

- [x] 1\. Repository and toolchain bootstrap
  - Create the package skeleton with `uv`, a committed lockfile, `pyproject.toml`, and the layer packages as empty modules: `domain`, `storage`, `ingest`, `simulation`, `analytics`, `optimisation`, `reporting`, `orchestration`, `cli`.
  - Configure `ruff` for lint and format, and `mypy` in strict mode for `domain`, `simulation` and `analytics`, standard mode elsewhere.
  - Configure `pytest` with a fixtures directory for archive samples and golden files.
  - Acceptance Criteria:
    - `uv sync` from a clean environment reproduces the locked dependency set exactly.
    - `ruff check` and `ruff format --check` pass on the empty skeleton.
    - `mypy` passes with strict settings active on the three designated packages.
    - `pytest` runs and reports zero tests without error.
  - _Dependencies: none_
  - _Requirements: Constitution — Technology Constraints_
  - _Complexity: Medium_

- [x] 2\. Cross-platform CI pipeline
  - Run lint, type-check and the full test suite on Linux and macOS, each from a clean environment built only from the committed lockfile.
  - Acceptance Criteria:
    - A pipeline run on both platforms passes from a cold cache.
    - No step requires a compiler or a system package the Operator would have to install separately.
    - A deliberately introduced lint, type or test failure fails the pipeline.
  - _Dependencies: 1_
  - _Requirements: NFR-009, SC-10_
  - _Complexity: Medium_

- [x] 3\. Architectural and interaction-model static gates
  - Add an `import-linter` contract encoding the one-way dependency rules: nothing imports `cli` or `orchestration`; `simulation` and `analytics` import only `domain`, the standard library and (for `analytics`) `numpy`/`scipy`; only `ingest` performs network I/O; only `storage` opens the database or takes locks.
  - Add a static gate asserting that no interactive prompt, confirmation or input helper — including Typer's prompt and confirm utilities and the built-in `input()` — is referenced anywhere in `cli` or `orchestration`.
  - Wire both gates into CI as build-breaking checks.
  - Acceptance Criteria:
    - The layering contract passes against the current skeleton.
    - An added import from `simulation` to `storage` fails the build.
    - An added import from `analytics` to `ingest` fails the build.
    - An added reference to a prompt or confirmation helper in `cli` fails the build, naming the offending call site.
  - _Dependencies: 1, 2_
  - _Requirements: FR-012, FR-020, RO-002; Constitution — Architecture Constraints, Interaction Model_
  - _Complexity: Medium_
  - _Risk: these gates are the structural enforcement behind the no-lookahead, purity and non-interactivity guarantees. If they are weak or advisory, those guarantees decay silently as the codebase grows._

- [x] 4\. Dependency prohibition gate
  - Add a CI check asserting that the resolved dependency tree contains no authenticated Binance SDK, no keyring or credential-storage library, and no telemetry or crash-reporting client.
  - Acceptance Criteria:
    - The check passes against the current lockfile.
    - Adding an authenticated exchange client to the dependencies fails the build with a message naming the offending package.
    - The check inspects the full transitive tree, not just direct dependencies.
  - _Dependencies: 1, 2_
  - _Requirements: NFR-005, SC-6; Constitution — Security Constraints_
  - _Complexity: Small_

---

## Milestone 2 — Domain Foundations

- [x] 5\. Constants inventory
  - Implement `domain.constants` holding every named constant from the requirements' constants table and the solution's design-level inventory: maximum drawdown limit, thin-fold cycle threshold, aggregate out-of-sample minimum cycles, selection consistency threshold, drawdown kill-threshold margin, default trial budget, minimum deflation confidence, equity sampling interval, expected bars per complete day, top-N, archive retry limit, progress interval, `FOLD_COUNT`, `TRAIN_INITIAL_HOURS`, `TEST_BLOCK_HOURS`, `EMBARGO_HOURS`, grid-count and leverage search bounds, `MIN_CYCLE_RATE_FRACTION`, `MAX_INFEASIBLE_DRAW_RATIO`, `ANNUALISATION_PERIODS`, `DEFLATION_FIXTURE_TOLERANCE`.
  - Mark provisional constants with a docstring noting they are subject to Milestone 11 calibration.
  - Acceptance Criteria:
    - Every constant named in the requirements or solution resolves from this module.
    - A repository-wide test asserts no numeric literal governing behaviour appears outside this module in `simulation`, `analytics` or `optimisation`.
    - Each constant has a stated unit and a one-line rationale.
  - _Dependencies: 1_
  - _Requirements: Requirements — Configuration Constants; Constitution — Coding Standards_
  - _Complexity: Medium_

- [x] 6\. Error hierarchy and terminal outcome types
  - Implement `OptimiserError` with `OperationalError` (exit 1) and `UsageError` (exit 2) subclasses, each carrying the context needed for an actionable message.
  - Define the `RunOutcome` discriminated union — `EndorsedResult`, `MeasuredResult`, `ResearchFinding(cause)` — with causes `no_viable_configuration`, `insufficient_oos_evidence`, `not_distinguishable_from_search_noise`, `risk_ceiling_breached`, `capital_below_venue_minimum`. This union is the contract every reporting task anchors on.
  - Acceptance Criteria:
    - Research findings are ordinary return values, not exceptions — asserted by a test that no finding type subclasses `Exception`.
    - Each error type carries the fields needed to render what failed, for which symbol and date, and what to do next.
    - The cause enumeration is closed and exhaustively matched.
  - _Dependencies: 5_
  - _Requirements: FR-025_
  - _Complexity: Medium_

- [x] 7\. Exact-arithmetic money layer and the settlement boundary
  - Implement the `Decimal` context, quantisation helpers bound to instrument tick and step size, and `settle()` as the single conversion point from `Decimal` to `float64`.
  - Acceptance Criteria:
    - A scenario accumulating many small fills reconciles exactly against an independently computed sum, with zero tolerance.
    - A repository-wide test asserts `float()` conversion of a monetary `Decimal` occurs only inside `settle()`.
    - Quantisation to tick and step size is exact and direction-explicit.
  - _Dependencies: 5_
  - _Requirements: NFR-007, SC-8; Constitution — Numeric Precision Boundary_
  - _Complexity: Large_
  - _Risk: the exact/float boundary is easy to breach accidentally with an incidental cast; once breached, PnL errors are small, plausible and very hard to spot._

- [x] 8\. Core domain models
  - Implement frozen, slot-based dataclasses: `Bar`, `MarketSlice`, `Fill`, `SimResult`, `SpotGridConfig`, `FuturesGridConfig`, `ResolvedWindow`, `FoldPlan`, `FoldResult`.
  - Validate in `__post_init__` so an invalid configuration cannot be constructed.
  - Acceptance Criteria:
    - Constructing a configuration with an inverted price range, a non-positive grid count, or a futures-only field on a spot config raises `UsageError`.
    - All models are immutable — mutation attempts raise.
    - `MarketSlice` exposes a bounded, chronologically ordered view and offers no method to widen its own range.
    - Every model round-trips through serialisation without loss.
  - _Dependencies: 6, 7_
  - _Requirements: FR-008; Constitution — Typed Immutable Domain Models_
  - _Complexity: Large_

- [x] 9\. Symbol reference data and the supported set
  - Package `SymbolSpec` reference data as a versioned data file: maker/taker fees, tick size, step size, minimum notional, funding interval, maximum leverage, per supported symbol and market type.
  - Implement the loader exposing `supported()` and `get()`, with `get()` raising `OperationalError` naming the supported set.
  - Acceptance Criteria:
    - The supported set is enumerable without a network call.
    - An unsupported symbol produces an error naming the available symbols.
    - `data_version` is exposed for inclusion in every result.
    - The file is loaded once at startup and never fetched at runtime.
  - _Dependencies: 8_
  - _Requirements: DR-003, DR-004; Solution D7_
  - _Complexity: Medium_

---

## Milestone 3 — Storage Layer

- [x] 10\. DuckDB schema and migrations
  - Define tables for `kline`, `funding_rate` and `ingest_day` with the declared decimal scales, and an explicit forward-only migration mechanism that preserves ingested data.
  - Deliberately create no table for results, rankings, trials or thresholds.
  - Acceptance Criteria:
    - `(symbol, market_type, open_time)` is the kline primary key, making duplicates impossible.
    - Monetary values round-trip through storage losslessly at the declared scale.
    - A migration applied to a populated store preserves every existing row.
    - A test asserts the schema contains no results-bearing table.
  - _Dependencies: 8_
  - _Requirements: DR-001, DR-002, DR-005, DR-006, FR-021, IR-002_
  - _Complexity: Large_

- [x] 11\. Append-only repository
  - Implement `append_day()` with verified idempotency, `load_slice()` returning an immutable `MarketSlice` for a bounded range, and `covered_dates()`.
  - Acceptance Criteria:
    - Re-appending a byte-identical day is a verified no-op.
    - Appending a byte-differing day for an existing date is rejected, not silently overwritten.
    - No code path deletes, prunes or expires stored market data.
    - `load_slice()` returns data for exactly the requested range and exposes no query handle.
    - Loading an identical window twice returns identical bars.
  - _Dependencies: 10_
  - _Requirements: FR-005, FR-007, DR-001_
  - _Complexity: Large_

- [x] 12\. Ingest provenance
  - Record source URL, verified checksum, observed bar count and ingest timestamp per ingested day, and expose them for offline window resolution.
  - Acceptance Criteria:
    - Every appended day writes a complete provenance row in the same transaction.
    - The recorded bar count is queryable without network access.
    - Provenance survives schema migration.
  - _Dependencies: 11_
  - _Requirements: DR-005, FR-001, FR-005_
  - _Complexity: Medium_

- [x] 13\. Shared-reader / exclusive-writer locking
  - Implement advisory `flock` context managers on a lock file in the data root: shared for analysis, exclusive for writes, with a short acquisition timeout and a deliberate upgrade ordering.
  - Acceptance Criteria:
    - Two concurrent readers both proceed.
    - A writer excludes readers and other writers for the duration of the write.
    - Failure to acquire within the timeout raises `OperationalError` naming the conflicting operation.
    - A read-then-upgrade-to-write sequence completes without deadlock under a contention test.
    - The lock file is created inside the data root and nowhere else.
  - _Dependencies: 10_
  - _Requirements: IR-002; Solution D5_
  - _Complexity: Large_
  - _Risk: lock-upgrade paths are a classic source of deadlock; the read-then-upgrade sequence in ingest needs deliberate ordering and a test that exercises contention._

---

## Milestone 4 — Ingest Layer

- [x] 14\. Archive HTTP client
  - Implement the `httpx`-based client with explicit connect and read timeouts, bounded retries with exponential backoff and jitter, a modest concurrency cap, and HTTPS-only with certificate verification always on.
  - Acceptance Criteria:
    - Transient failures retry up to the archive retry limit and then raise `OperationalError`.
    - No request is issued to any host other than the public data archive.
    - No authentication header is ever attached.
    - Retry backoff is bounded and does not hammer the archive under sustained failure.
  - _Dependencies: 9_
  - _Requirements: IR-001, NFR-005_
  - _Complexity: Large_

- [x] 15\. Checksum verification
  - Fetch the published checksum for each archive and verify before any content is parsed or written.
  - Acceptance Criteria:
    - A matching checksum permits ingest.
    - A mismatch writes nothing and raises `OperationalError` reporting symbol, date and both checksum values.
    - Verification occurs before extraction, not after.
  - _Dependencies: 14_
  - _Requirements: FR-003, IR-001_
  - _Complexity: Small_

- [x] 16\. Defensive extraction and strict CSV parsing
  - Validate ZIP contents before extraction: reject absolute paths, parent-directory traversal, unexpected member names, member counts above one, and implausible uncompressed sizes. Extract into a temporary directory inside the data root and promote atomically. Delete the archive after successful ingest.
  - Parse the CSV with strict typing and an explicit column count, rejecting malformed rows rather than coercing them.
  - Acceptance Criteria:
    - A crafted archive containing a traversal path is rejected with nothing written to disk.
    - A multi-member or oversized archive is rejected before extraction.
    - A malformed row fails the whole day rather than being skipped.
    - No file is written outside the data root at any point.
    - The downloaded archive is absent from disk after a successful ingest.
  - _Dependencies: 15, 13_
  - _Requirements: FR-003, NFR-008; Constitution — Security Constraints_
  - _Complexity: Large_
  - _Risk: archive content is untrusted input and extraction is the only place this tool writes attacker-influenced paths to disk._

- [x] 17\. Candidate end-date resolution from local provenance
  - Determine the candidate end date from the local provenance record where possible, falling back to an archive existence probe only when the store cannot answer.
  - Acceptance Criteria:
    - A store holding a complete candidate end date yields that date with zero network calls.
    - A store that cannot answer issues exactly one existence probe.
    - The candidate date never derives from the local system clock.
  - _Dependencies: 12_
  - _Requirements: FR-001, IR-001_
  - _Complexity: Medium_

- [x] 18\. Completeness verification and step-back loop
  - Verify the candidate end date holds the expected 1,440 bars; where it is absent locally or short, fetch and verify; step back one day and repeat while incomplete; return the resolved window.
  - Acceptance Criteria:
    - A short or still-accruing trailing day moves the resolved end date earlier and raises no error.
    - Each step-back attempt performs at most one fetch and applies the store-first rule again.
    - Insufficient archive history for the requested window refuses the run and reports how many days are available.
    - The resolved window start and end are returned for reporting before analysis begins.
  - _Dependencies: 17, 16_
  - _Requirements: FR-001_
  - _Complexity: Large_
  - _Risk: the store-first path is what makes the zero-download warm-run guarantee hold; a subtle fallthrough here silently reintroduces a network call on every run._

- [x] 19\. Incremental acquisition
  - Determine required data for the resolved window, identify what is absent locally, and download only that.
  - Acceptance Criteria:
    - An empty store and a 30-day window produce exactly 30 kline downloads, plus funding data for futures.
    - A store already covering the window performs zero downloads end to end, window resolution included.
    - A store missing 5 of 30 days downloads exactly those 5.
    - No command exposes ingest as a separately invocable step.
  - _Dependencies: 18_
  - _Requirements: FR-002, SC-2_
  - _Complexity: Medium_

- [x] 20\. Interior coverage verification
  - Verify every date interior to the resolved window is present with a full bar count; produce an explicit missing-date list.
  - Acceptance Criteria:
    - A window with an interior gap refuses the run and lists the specific dates.
    - A short interior day is treated as missing, not as present.
    - No interpolation, forward-fill or synthesis occurs anywhere in the data path.
    - No flag overrides the refusal.
  - _Dependencies: 19_
  - _Requirements: FR-004_
  - _Complexity: Medium_

- [x] 21\. Funding rate ingest
  - Fetch, verify and store USD-M futures funding rate history covering any ingested period.
  - Acceptance Criteria:
    - Funding rows exist for every funding interval within the resolved window before a futures simulation runs.
    - Absent funding data within the window is reported as a coverage failure.
    - Funding rows carry the same provenance guarantees as klines.
  - _Dependencies: 20_
  - _Requirements: DR-002, FR-004, FR-010_
  - _Complexity: Medium_

---

## Milestone 5 — Simulation Core

- [x] 22\. Strategy registry and interface
  - Define the strategy interface — configuration schema, search space derivation, order generation, fill and accounting semantics — and the registry resolving an identifier to an implementation.
  - Acceptance Criteria:
    - Registering a third strategy requires no change to the optimisation loop, the storage schema, or the CLI beyond a new type value.
    - The registry is populated declaratively and resolves deterministically.
    - The interface exposes no method capable of I/O.
  - _Dependencies: 8_
  - _Requirements: FR-011_
  - _Complexity: Medium_

- [x] 23\. Grid level construction and allocation profiles
  - Build grid levels for arithmetic and geometric spacing from lower bound, upper bound and grid count. Distribute total capital uniformly across levels, which is the only allocation Binance implements.
  - Acceptance Criteria:
    - Arithmetic spacing produces equal price differences; geometric produces equal ratios.
    - Allocated capital sums to at most the Operator's total capital, never more.
    - Level prices are quantised to tick size and order quantities to step size.
    - Level construction is deterministic for identical inputs.
  - _Dependencies: 22, 7_
  - _Requirements: FR-009, IR-003_
  - _Complexity: Large_

- [x] 24\. Vectorised event detection
  - Sort levels once; use `numpy.searchsorted` against each bar's low and high to derive the index range of levels touched; emit the set of event-bearing bars.
  - Acceptance Criteria:
    - A bar touching no level is excluded from the event set.
    - A bar spanning *n* levels reports exactly those *n* level indices.
    - Detection results are identical to a naive per-bar scan over a large randomised fixture.
    - A bar whose low or high equals a level price exactly is included, verified by a boundary fixture.
    - Detection performs no accounting and mutates no state.
  - _Dependencies: 23_
  - _Requirements: FR-009, FR-013; Solution D3_
  - _Complexity: Large_
  - _Risk: an off-by-one in the boundary comparison silently drops or duplicates fills at exactly the grid price — the most common price in the dataset._

- [x] 25\. Dual-path conservative intrabar resolution
  - On any bar touching more than one level, evaluate both monotone paths — open→low→high→close and open→high→low→close — and retain the state and PnL of whichever yields the lower closing equity.
  - Acceptance Criteria:
    - On every event-bearing bar, the retained outcome is never better than the discarded alternative.
    - Resolution is deterministic and identical across runs.
    - Each pessimistic branch carries a comment naming the ambiguity and the choice.
  - _Dependencies: 24_
  - _Requirements: FR-013; Constitution Principle 1; Solution D8_
  - _Complexity: Large_
  - _Risk: this is the concrete expression of the project's pessimism mandate; if it silently favours the strategy, every downstream number is optimistic and no other test will catch it._

- [x] 26\. Spot Grid resting-order state machine
  - Implement order placement and replacement: buys resting below price and sells above, a filled buy placing a corresponding sell one level up and vice versa, and no orders placed beyond the configured range.
  - Acceptance Criteria:
    - No order is ever placed outside the configured range.
    - A filled buy places exactly one sell one level up; a filled sell places exactly one buy one level down.
    - Price leaving the range results in held inventory and no new orders beyond the bounds.
    - The order book state is reconstructible at any bar index.
  - _Dependencies: 25_
  - _Requirements: FR-009_
  - _Complexity: Large_

- [x] 27\. Spot Grid fee and inventory accounting
  - Accumulate cash, inventory, fees and matched-cycle counts in `Decimal` across the fill sequence, starting from a flat position holding full total capital.
  - Acceptance Criteria:
    - Every fill charges the applicable maker or taker fee from reference data.
    - Cash plus inventory value reconciles exactly at every step, zero tolerance.
    - Matched buy/sell pairs are counted as completed grid cycles.
    - Simulation begins from a flat position holding full total capital.
  - _Dependencies: 26_
  - _Requirements: FR-009, NFR-007_
  - _Complexity: Large_

- [x] 28\. Futures directional position and margin under leverage
  - Extend the engine for long, short and neutral directions, applying configured leverage to position sizing and margin with total capital as the margin base.
  - Acceptance Criteria:
    - Neutral direction behaves distinctly from long and short on the same series.
    - Position size scales with leverage against the total-capital margin base.
    - Margin is tracked exactly and never becomes negative without triggering liquidation evaluation.
  - _Dependencies: 27_
  - _Requirements: FR-010_
  - _Complexity: Large_

- [x] 29\. Futures funding application
  - Apply funding at each funding interval using the historical rate for that interval and the position then held, with sign reflecting position direction.
  - Acceptance Criteria:
    - Funding is applied at every interval within the simulated range, not approximated at the boundaries.
    - A long position pays under positive funding and receives under negative; a short position does the reverse.
    - Total funding paid or received is accumulated separately from fees for reporting.
    - A position closed before an interval accrues no funding for it.
  - _Dependencies: 28, 21_
  - _Requirements: FR-010_
  - _Complexity: Large_
  - _Risk: funding is a slow, compounding drag that a 30-day backtest will materially misstate if applied at the wrong cadence or sign._

- [x] 30\. Futures liquidation
  - Compute the liquidation price from position, leverage and margin, and terminate the simulation with a liquidation outcome when price reaches it. Under the dual-path rule, liquidation occurs if either feasible path reaches the level.
  - Acceptance Criteria:
    - A series driving the position to its liquidation price liquidates at the computed level.
    - A bar reaching liquidation on only one of the two feasible paths still liquidates.
    - A liquidated run reports the bar at which it occurred and is never reported as merely poor-performing.
    - No fills are recorded after liquidation.
  - _Dependencies: 29_
  - _Requirements: FR-010, FR-013_
  - _Complexity: Large_
  - _Risk: liquidation is the difference between a bad backtest and a total loss; modelling it approximately produces plausible-looking results that understate ruin._

- [x] 31\. Hourly mark-to-market equity reconstruction
  - Reconstruct the hourly equity series from the fill log after settlement: cash plus open inventory valued at the prevailing close, in `float64`.
  - Acceptance Criteria:
    - Equity is produced at exact hourly boundaries across the simulated range.
    - Periods with no completed cycles are ordinary observations reflecting inventory revaluation, not zero-return periods.
    - Reconstruction occurs strictly after settlement and introduces no rounding into the accounting path.
    - The series length matches the simulated duration in hours.
  - _Dependencies: 27, 30_
  - _Requirements: NFR-007; Requirements — Objective Function_
  - _Complexity: Large_

- [x] 32\. No-lookahead truncation-equivalence suite
  - Implement the truncation-equivalence property test: simulating a series truncated at bar *T* produces results for bars 0..*T* identical to simulating the full series. Parametrise across every registered strategy.
  - Acceptance Criteria:
    - The property holds for spot and futures strategies across randomised fixtures.
    - A deliberately introduced forward reference — for example a full-sample volatility normalisation — fails the test.
    - The test is parametrised over the registry, so a new strategy is covered automatically.
  - _Dependencies: 31_
  - _Requirements: FR-012; Constitution Principle 6_
  - _Complexity: Large_
  - _Risk: this is the only automated defence against the single most damaging class of backtesting defect._

- [x] 33\. Golden-file scenarios
  - Commit hand-verified scenarios with independently derived fills, fees, inventory and PnL, computed by hand or by an independent method rather than from the implementation under test.
  - Acceptance Criteria:
    - Each golden file states how its expected values were derived and by what independent method.
    - Fills, fees, inventory and realised PnL all match exactly.
    - Changing a golden value requires an explicit justification recorded alongside it.
  - _Dependencies: 32_
  - _Requirements: FR-009, FR-010; Constitution — Testing Approaches_
  - _Complexity: Large_
  - _Risk: golden values derived from the implementation rather than independently would make the entire suite self-confirming and worthless._

- [x] 34\. Synthetic spot price-series scenarios
  - Construct analytically derivable series: flat, monotone ramp through the grid, clean oscillation across *n* levels, and price exiting the configured range.
  - Acceptance Criteria:
    - Flat series yields zero cycles, zero fees and zero PnL.
    - Monotone ramp yields the exactly predicted fill count and direction.
    - Oscillation across *n* levels yields the exactly predicted matched-pair count and grid profit.
    - Range exit yields no fills beyond the bounds and held inventory thereafter.
  - _Dependencies: 32_
  - _Requirements: FR-009, FR-012_
  - _Complexity: Large_

- [x] 35\. Synthetic futures liquidation and funding scenarios
  - Construct series that drive a leveraged position to its liquidation price, and series spanning multiple funding intervals with known rates.
  - Acceptance Criteria:
    - The liquidation ramp liquidates at the computed level and at the expected bar.
    - A multi-interval series accrues exactly the hand-computed total funding for each direction.
    - A position closed mid-interval accrues no funding for that interval.
  - _Dependencies: 32_
  - _Requirements: FR-010, FR-013_
  - _Complexity: Large_

---

## Milestone 6 — Analytics

- [x] 36\. Per-period Sortino and the annualisation boundary
  - Implement `sortino_per_period()` over hourly returns with downside threshold zero and risk-free zero, returning an `UNDEFINED` sentinel when downside deviation is zero. Implement `annualise()` as a separate call using `ANNUALISATION_PERIODS`.
  - Acceptance Criteria:
    - Values match hand-computed results on fixed series.
    - A series with no observations below zero returns `UNDEFINED` rather than a large or infinite value.
    - Annualisation is never applied inside the ratio computation.
    - Neither function returns `inf` or silently propagates `nan`.
  - _Dependencies: 31_
  - _Requirements: FR-018; Requirements — Objective Function_
  - _Complexity: Large_

- [x] 37\. Maximum drawdown with three labelled bases
  - Implement largest peak-to-trough decline of an hourly equity curve as a fraction of total capital, always returned with its basis: `training_segment`, `concatenated_out_of_sample` or `whole_window`.
  - Acceptance Criteria:
    - Values match hand-computed results on curves with known peak-to-trough structure.
    - Every returned value carries its basis and no call site can omit it.
    - The concatenated basis produces a deeper trough than any constituent fold when consecutive folds decline.
  - _Dependencies: 31_
  - _Requirements: FR-018, FR-020; Requirements — Maximum Drawdown_
  - _Complexity: Medium_

- [x] 38\. Trial-count-aware deflation
  - Implement the null benchmark `R*` from trial-ratio dispersion and trial count, and the confidence via the moment-adjusted normal CDF, accepting only per-period ratios. Commit both worked examples as fixtures.
  - Acceptance Criteria:
    - Example A reproduces confidence ≈ 0.291 within `DEFLATION_FIXTURE_TOLERANCE`.
    - Example B reproduces confidence ≈ 0.952 within `DEFLATION_FIXTURE_TOLERANCE`.
    - The moment-adjustment denominator is recomputed per observed ratio, verified by a test asserting the two examples yield different denominators.
    - Passing an annualised ratio to the entry point is rejected.
    - Confidence falls monotonically as trial count rises, all else equal.
  - _Dependencies: 36, 5_
  - _Requirements: FR-019; Constitution Principle 3; Solution D9_
  - _Complexity: Large_
  - _Risk: a scaling error here inflates every confidence to near 1.0 and silently disables the endorsement gate, which is the project's central overfitting control._

---

## Milestone 7 — Backtest Vertical Slice

*This milestone proves the entire output contract — renderer, JSON envelope, exit codes, disclosures — against a working single-configuration command, before any optimisation code exists. Tasks 39, 40 and 41 all anchor on the `RunOutcome` union from task 6 and can proceed in parallel.*

- [x] 39\. `RunOutcome` text renderer core
  - Render the fixed table layout for any `RunOutcome`: warnings and downgrade statements first, then resolved window, symbol, market type and capital, then the configuration and its metrics, then invalidation thresholds and disclosures.
  - Acceptance Criteria:
    - Warnings never appear beneath the metrics.
    - Output uses no colour or cursor control and is stable and diffable across identical runs.
    - A `MeasuredResult` states plainly that no recommendation is made because no search was performed.
    - Every `RunOutcome` variant renders without error.
  - _Dependencies: 6_
  - _Requirements: FR-022, FR-024_
  - _Complexity: Large_

- [x] 40\. JSON envelope and recommendation contract
  - Emit the versioned envelope for any `RunOutcome`: schema version, command, outcome, tri-state recommendation, cause, run, configuration, performance, invalidation and disclosures. Omit `recommendation` from operational-error payloads.
  - Acceptance Criteria:
    - `recommendation` is `not_applicable` for `backtest`, `data status` and `data symbols`, and absent for operational errors.
    - Research findings carry a specific cause; results carry `null`.
    - With `--json`, stdout contains valid JSON and nothing else.
    - Every payload validates against a committed schema.
  - _Dependencies: 6_
  - _Requirements: FR-023, FR-025, IR-004_
  - _Complexity: Large_

- [x] 41\. Exit code mapping
  - Map outcomes to exit codes at the single exception boundary: 0 for results and research findings, 1 for operational errors including unsupported symbol, 2 for usage errors.
  - Acceptance Criteria:
    - Every research finding exits 0.
    - No operational error exits 0.
    - Unsupported symbol exits 1; `search --days 7` and loose overrides exit 2.
    - Exit codes are documented and asserted by tests.
  - _Dependencies: 6_
  - _Requirements: FR-025, IR-004_
  - _Complexity: Medium_

- [x] 42\. Disclosure emitter
  - Emit one disclosure entry per row of the constitution's declared assumptions table, plus the design-level disclosures: the deflation approximation, the in-sample dispersion proxy, and the flat-start fold assumption.
  - Acceptance Criteria:
    - A disclosure entry exists for every declared assumption, asserted by a test that fails if the two drift apart.
    - The unverified-fill-rate, queue-position and short-window statements are always present.
    - Disclosures are unconditional and identical across output formats.
  - _Dependencies: 6_
  - _Requirements: FR-024; Constitution Principle 8_
  - _Complexity: Medium_

- [x] 43\. `backtest` command
  - Wire the Typer command accepting symbol, market, days, capital, max drawdown, the full strategy parameter set, and the invalidation override flags; run the single-configuration workflow end to end.
  - Acceptance Criteria:
    - An incompletely specified configuration is rejected before any simulation, naming the missing parameters.
    - Trial count is exactly one and no deflation is applied, with the output saying why.
    - Drawdown is reported on the `whole_window` basis.
    - The command never emits `recommended`.
    - Futures-only flags on a spot run are rejected.
    - The command runs to completion with stdin closed and never prompts.
  - _Dependencies: 39, 40, 41, 42, 27, 30, 36, 37_
  - _Requirements: FR-008, FR-019, RO-002_
  - _Complexity: Large_

- [x] 44\. Backtest slice end-to-end verification
  - Exercise the complete `backtest` path against a fixture store with no network access, in both output formats, across success and every relevant failure mode.
  - Acceptance Criteria:
    - Both output formats are produced and the JSON validates against the committed schema.
    - Missing interior data, checksum failure and unsupported symbol each produce the correct exit code and message.
    - The whole path runs with no network access.
    - Two identical invocations produce byte-identical stdout.
  - _Dependencies: 43_
  - _Requirements: FR-008, FR-025, NFR-004_
  - _Complexity: Large_

---

## Milestone 8 — Optimisation

- [x] 45\. Expanding-window fold planner with embargo
  - Divide the 720-hour window into four folds: nested training windows growing 240 → 600 hours, a 6-hour embargo excluded from both sides, and 114-hour test segments.
  - Acceptance Criteria:
    - Test segments are strictly non-overlapping, chronological and total 456 hours.
    - The embargo hours appear in neither the training nor the test segment of any fold.
    - Fold boundaries derive from the constants module, not from literals.
    - Segment slices are bounded `MarketSlice` instances, not query handles.
  - _Dependencies: 11, 5_
  - _Requirements: FR-017; Solution D1, D-003_
  - _Complexity: Large_

- [x] 46\. Venue feasibility filter
  - Snap price bounds to tick size and quantities to step size; clamp leverage to the symbol's maximum; reject candidates whose smallest per-level order falls below minimum notional for the supplied capital. Bound redraws by `MAX_INFEASIBLE_DRAW_RATIO`.
  - Acceptance Criteria:
    - A reported configuration always respects tick size, step size, minimum notional and maximum leverage.
    - A capital-and-grid-count combination producing sub-minimum orders is rejected before simulation.
    - A `backtest` configuration violating a venue constraint raises `UsageError` naming the parameter and the binding limit.
    - Exhausting the redraw bound produces the `capital_below_venue_minimum` finding with diagnostics naming the binding venue minimum — the minimum notional or, where the ladder's denser rungs floor the per-level quantity out first, the minimum quantity.
    - Infeasible candidates do not increment the trial counter.
  - _Dependencies: 23, 9_
  - _Requirements: IR-003, SC-13_
  - _Complexity: Large_

- [x] 47\. Training-only search space derivation
  - Derive range bounds from training realised volatility and price extent, grid count uniform over the rungs of `GRID_COUNT_LADDER`, spacing mode, allocation profile, and for futures leverage and direction — computed from the training segment only and snapped to a common quantisation grid.
  - Acceptance Criteria:
    - Derivation reads no data outside the training segment, asserted by a truncation test.
    - Identical training data yields an identical space.
    - A configuration selected in one fold is identifiable as the same configuration in another.
    - The derived space is exposed for inclusion in the result.
  - _Dependencies: 45, 46_
  - _Requirements: FR-012, FR-014_
  - _Complexity: Large_
  - _Risk: deriving any part of the space from the full window is a lookahead leak that the truncation test on the simulator alone would not catch._

- [x] 48\. Sampler and budget allocation
  - Implement scrambled Sobol sampling over the unit hypercube mapped through each dimension's distribution, seeded from run seed and fold index, with floor-division budget allocation and the remainder to the earliest folds.
  - Acceptance Criteria:
    - A budget of 2,002 across four folds allocates 501, 501, 500, 500.
    - Identical seed and inputs reproduce an identical draw sequence.
    - Planned per-fold counts are computable and reportable before the run begins.
    - The draw never exceeds its fold's allocation.
  - _Dependencies: 47_
  - _Requirements: FR-016; Solution D2_
  - _Complexity: Large_

- [x] 49\. Trial counter
  - Count every training-segment candidate evaluation; count out-of-sample confirmation runs separately as `confirmation_runs`; expose both.
  - Acceptance Criteria:
    - No training evaluation bypasses the counter, asserted by routing all evaluation through a single entry point.
    - The budget is never exceeded.
    - Confirmation runs are excluded from the trial count fed to deflation.
    - The final count is available for the result payload.
  - _Dependencies: 48_
  - _Requirements: FR-016, FR-019_
  - _Complexity: Medium_

- [x] 50\. Per-fold selection with the drawdown gate
  - Rank candidates by per-period training Sortino; exclude candidates with undefined Sortino; exclude candidates whose training-segment drawdown exceeds the applied limit; record the fold winner.
  - Acceptance Criteria:
    - A candidate breaching the drawdown limit is absent from the ranking entirely, not ranked lower.
    - A candidate with no downside observations is excluded and reported as such.
    - Selection reads no test-segment data, asserted by test.
    - Ranking never uses absolute return.
  - _Dependencies: 49, 36, 37_
  - _Requirements: FR-018_
  - _Complexity: Large_

- [x] 51\. Concatenated out-of-sample series construction
  - Simulate each fold winner on its test segment from a flat position with full capital; chain the hourly return series chronologically and compound from a single common starting equity.
  - Acceptance Criteria:
    - Every fold's contribution is included regardless of cycle count — no fold is discarded.
    - The concatenated curve compounds, so consecutive declining folds produce one deeper trough than any single fold.
    - Each fold's test simulation begins flat with full total capital.
    - The concatenated series length equals the sum of the fold test-segment lengths.
  - _Dependencies: 50_
  - _Requirements: FR-017_
  - _Complexity: Large_

- [x] 52\. Thin-fold marking and the aggregate cycle gate
  - Mark folds whose test segment yields fewer completed cycles than the thin-fold threshold; apply the aggregate out-of-sample minimum cycle gate.
  - Acceptance Criteria:
    - A thin fold is marked and counted but remains in the series and in every statistic.
    - Total cycles below the aggregate minimum produces `insufficient_oos_evidence` and no headline figure.
    - The thin-fold count and total fold count are exposed for the trust context.
    - Thin folds are never excluded from any computation.
  - _Dependencies: 51_
  - _Requirements: FR-017; Solution D-002_
  - _Complexity: Medium_

- [x] 53\. Risk-ceiling check and downgrade
  - Compare the reported configuration's concatenated out-of-sample drawdown against the applied limit; downgrade to a research finding when it exceeds.
  - Acceptance Criteria:
    - A breach downgrades the result regardless of deflated confidence.
    - The downgraded result reports observed drawdown and applied limit side by side, and states that selection could not have prevented it.
    - A downgraded result exits as success.
    - The recommendation becomes `not_recommended`.
  - _Dependencies: 52, 38_
  - _Requirements: FR-018; Constitution Principle 4_
  - _Complexity: Large_

- [x] 54\. Selection stability and in-sample/out-of-sample divergence
  - Compute distinct fold winners and folds won by the reported configuration; apply the consistency threshold; emit the nested-training caveat. Compute the divergence between the reported configuration's per-period training Sortino and its per-period concatenated out-of-sample Sortino.
  - Acceptance Criteria:
    - The reported configuration is the most frequently selected winner, ties broken toward the most recent fold.
    - Winning fewer folds than the threshold produces an explicit selection-instability warning that accompanies a result without converting it into a research finding.
    - The nested-training correlation caveat appears in the trust context.
    - Divergence is computed as the training-segment ratio minus the out-of-sample ratio, both per-period, and is present in both output formats.
    - Where either ratio is `UNDEFINED`, divergence is reported as unavailable with the reason, never as zero or omitted silently.
  - _Dependencies: 52_
  - _Requirements: FR-018, FR-024; Solution D-003_
  - _Complexity: Large_

- [x] 55\. Invalidation thresholds and overrides
  - Derive the drawdown kill threshold as the tighter of (worst basis-appropriate drawdown + margin) and the applied limit, and the minimum cycle rate as `MIN_CYCLE_RATE_FRACTION` × realised cycles per day — for both the `per_fold_out_of_sample` and `whole_window` bases. Honour overrides and reject loose ones.
  - Acceptance Criteria:
    - Both thresholds appear on every result carrying performance metrics, including `backtest` and downgraded results.
    - The basis is labelled and correct for each command.
    - A `--kill-drawdown` looser than the applied limit is rejected as a usage error with an explanation.
    - An honoured override sets the source to `supplied`; otherwise `derived`.
    - Where the applied limit was the binding constraint, the output says so.
    - No code path solicits these values from the Operator — they are derived or supplied by flag only, asserted by test.
  - _Dependencies: 53_
  - _Requirements: FR-020, SC-14, RO-002; Constitution Principle 5_
  - _Complexity: Large_

- [x] 56\. Empty-result diagnostics
  - When no configuration survives, report trials evaluated, per-gate rejection counts, best drawdown achieved against the applied limit, and a specific relaxed drawdown value that would have admitted a candidate.
  - Acceptance Criteria:
    - Rejection counts are reported separately per gate — drawdown exclusion, undefined downside, and feasibility.
    - The suggested relaxed value would demonstrably have admitted at least one candidate.
    - The outcome exits as success and is distinguishable in JSON from any operational error.
    - The diagnostics are produced by the search rather than reconstructed afterwards.
  - _Dependencies: 55_
  - _Requirements: FR-018_
  - _Complexity: Large_

---

## Milestone 9 — Optimise Reporting and CLI

- [x] 57\. Optimise-specific rendering extensions
  - Extend the renderer with the top-N distinct fold winners ordered by selection frequency, the trust-context block, and the deflated confidence adjacent to the headline ratio.
  - Acceptance Criteria:
    - Fewer than N rows appear when fewer distinct winners exist, without error.
    - The raw ratio never appears without its deflated companion.
    - Trust context includes trials evaluated, applied constraints, selection stability, thin-fold count, divergence and the derived search space.
    - Output remains stable and diffable across identical runs.
  - _Dependencies: 39, 54, 55, 56_
  - _Requirements: FR-022, FR-024_
  - _Complexity: Large_

- [x] 58\. Staged-trial guidance conditional field
  - Emit `staged_trial_guidance` if and only if the recommendation is `recommended`, in both output formats.
  - Acceptance Criteria:
    - The field is present exactly when the recommendation is `recommended` and absent otherwise.
    - The guidance states that the configuration should be run at minimum size before scaling and that the simulated fill rate is unverified.
    - A downgraded or search-noise result never carries the field.
  - _Dependencies: 40, 53_
  - _Requirements: FR-024; Constitution Principle 5_
  - _Complexity: Medium_

- [x] 59\. `search` command
  - Wire the command accepting symbol, market, days, capital, max drawdown, trial budget, seed, top-N and override flags; run the full walk-forward search workflow.
  - Acceptance Criteria:
    - `--days 7` is refused with an explanation pointing to `backtest`, and never degrades into an in-sample sweep.
    - Per-parameter search bounds are not accepted.
    - Total capital is held fixed across every trial and never swept.
    - The planned trial count and estimated duration are reported before the search begins.
    - The command runs to completion with stdin closed and never prompts.
  - _Dependencies: 57, 58, 56_
  - _Requirements: FR-014, FR-015, FR-016, RO-002_
  - _Complexity: Large_

- [x] 60\. `data status` and `data symbols` commands
  - Report per-symbol coverage and gaps from the local store without network access; list the supported symbol set with its venue constraints.
  - Acceptance Criteria:
    - `data status` performs no network access and resolves no window.
    - An empty store reports as empty without error.
    - `data symbols` lists market type, tick size, step size, minimum notional and maximum leverage per supported symbol.
    - Both commands emit `not_applicable` as their recommendation.
  - _Dependencies: 40, 12_
  - _Requirements: FR-006, DR-004_
  - _Complexity: Medium_

- [x] 61\. Progress reporting and structured logging
  - Write progress to stderr unconditionally at intervals of at most 10 seconds, varying only rendering style by TTY attachment. Add levelled logging to stderr, default WARNING, with debug detail covering fold winner selection, feasibility rejections and gate rejection counts.
  - Acceptance Criteria:
    - No operation exceeds 10 seconds without a progress update, TTY or not.
    - stdout is never touched by progress or logging under any condition.
    - No timing value appears in the stdout payload in either format.
    - Debug logging is sufficient to reconstruct why a run produced a research finding.
  - _Dependencies: 59_
  - _Requirements: NFR-006, SC-7, FR-023_
  - _Complexity: Medium_

---

## Milestone 10 — Integration and Verification

- [x] 62\. End-to-end command and outcome coverage
  - Exercise every command and every outcome kind against a fixture store with no network access, including the empty-result, insufficient-evidence, search-noise and risk-ceiling paths.
  - Acceptance Criteria:
    - Every outcome kind in the terminal-semantics table is reachable and asserted.
    - Every exit code is asserted against a triggering scenario.
    - No test requires network access.
  - _Dependencies: 61, 44_
  - _Requirements: FR-025_
  - _Complexity: Large_

- [x] 63\. JSON schema validation across all payload shapes
  - Validate every payload shape against the committed schema, including the operational-error shape.
  - Acceptance Criteria:
    - Every payload emitted by every command validates.
    - The operational-error payload asserts `recommendation` is absent and `outcome` is present.
    - A schema-breaking change to a payload fails the suite.
  - _Dependencies: 62, 40_
  - _Requirements: FR-023, IR-004_
  - _Complexity: Medium_

- [x] 64\. Determinism and reproducibility verification
  - Assert byte-identical stdout across repeated identical runs, and that a recorded seed reproduces the same search, fold winners, reported configuration, deflated statistic and recommendation.
  - Acceptance Criteria:
    - Two runs over an unchanged store produce byte-identical JSON with no excluded fields.
    - A re-run with a recorded seed reproduces every derived value.
    - Iteration and collection ordering is explicitly deterministic throughout.
  - _Dependencies: 62_
  - _Requirements: NFR-004, SC-5_
  - _Complexity: Large_

- [x] 65\. Containment and credential verification
  - Verify via filesystem monitoring across a full ingest-and-search cycle that nothing is written outside the configured data root, and that no credential is read from any source.
  - Acceptance Criteria:
    - Zero writes outside the data root, including temporary extraction files.
    - The temporary directory is cleared after successful ingest.
    - No credential is read from environment, file or argument on any path.
  - _Dependencies: 62_
  - _Requirements: NFR-005, NFR-008, SC-9_
  - _Complexity: Medium_

- [x] 66\. Performance and footprint measurement harness
  - Measure cold ingest throughput, peak resident memory during ingest and a default-budget search, stored-data footprint and total data-root footprint.
  - Acceptance Criteria:
    - Cold 30-day single-symbol ingest completes within 3 minutes at 20 Mbit/s.
    - Peak RSS stays under 1 GB for every supported operation on an 8 GB machine.
    - Stored market data stays within 30 MB per symbol-month and the data root within 90 MB per symbol-month.
    - Measurements are repeatable and recorded per release.
  - _Dependencies: 62_
  - _Requirements: NFR-001, NFR-002, NFR-003, SC-1, SC-3, SC-4, SC-4b_
  - _Complexity: Large_

- [x] 67\. Guardrail coverage audit
  - Enumerate every guardrail named in the requirements and solution and assert each has a dedicated passing test.
  - Acceptance Criteria:
    - Each of the following has a named test: aggregate cycle gate suppression; 7-day search refusal; drawdown gate exclusion; risk-ceiling downgrade; thin folds retained; `backtest` never endorsed; trial counter completeness; confirmation runs excluded; infeasible draws excluded; loose override rejection; backtest invalidation basis; conditional staged-trial guidance; empty-result diagnostics; one disclosure per declared assumption; **no interactive prompt anywhere in the CLI and every command completing with stdin closed**.
    - The audit fails if a guardrail is named in a spec but has no corresponding test.
  - _Dependencies: 62_
  - _Requirements: SC-12, RO-002; Constitution — Testing Approaches_
  - _Complexity: Large_

---

## Milestone 11 — Calibration of Provisional Constants

- [x] 68\. Measure runtime and calibrate the trial budget
  - Run default-budget searches across several symbols and market types on reference hardware; record wall-clock time and the distribution of per-simulation cost.
  - Acceptance Criteria:
    - A default-budget 30-day search run completes within a single interactive sitting, with the measured figure published rather than assumed.
    - If it does not, the trial budget is revised and the new value recorded with its measurement.
    - The pre-run duration estimate is compared against actual and its error characterised.
  - _Dependencies: 66_
  - _Requirements: SC-11; Constitution — Performance Targets_
  - _Complexity: Large_

- [x] 69\. Calibrate cycle-count thresholds
  - Measure realised completed-cycle counts per test segment across symbols and market conditions; assess whether the thin-fold threshold of 20 and the aggregate minimum of 60 discriminate usefully.
  - Acceptance Criteria:
    - Threshold values are either confirmed against measured distributions or revised with recorded rationale.
    - The revised values do not cause the aggregate gate to trigger on the great majority of ordinary runs, nor to never trigger.
    - Constants are updated only in the constants module.
  - _Dependencies: 68_
  - _Requirements: FR-017; Requirements — Configuration Constants_
  - _Complexity: Large_

- [x] 70\. Calibrate the deflation gate against real trial dispersion
  - Measure actual trial-ratio dispersion across real searches; compute the resulting null benchmark and the annualised Sortino a configuration must post to clear the 0.95 confidence gate.
  - Acceptance Criteria:
    - Real dispersion is recorded and the implied endorsement bar reported.
    - If the gate proves effectively unclearable, the trial budget is reduced rather than the confidence threshold lowered, per the solution's stated policy.
    - Any revision is recorded with its measurement and rationale.
  - _Dependencies: 69_
  - _Requirements: FR-019; Solution D9_
  - _Complexity: Large_
  - _Risk: the temptation under schedule pressure is to lower the threshold so results look endorsable; doing so silently reinstates the multiple-testing problem the whole design exists to control._

- [x] 71\. Apply revised constants and re-verify
  - Update the constants module with calibrated values and re-run the complete suite.
  - Acceptance Criteria:
    - All golden-file, analytics, guardrail and end-to-end tests pass with revised constants.
    - Any deflation fixture affected by a dispersion change is recomputed and re-derived independently.
    - Provisional markers are removed from calibrated constants and retained on any still unmeasured.
  - _Dependencies: 70_
  - _Requirements: Requirements — Configuration Constants_
  - _Complexity: Large_

---

## Milestone 12 — Documentation

- [x] 72\. Operator usage documentation
  - Document installation, the four commands and every flag, the resolved-window concept, and worked examples of each outcome kind including a research finding and an empty result.
  - Acceptance Criteria:
    - Every flag in the command surface is documented with its default and its source constant.
    - The documentation explains why a research finding is a success and not a failure.
    - Examples are reproducible against a fixture store.
  - _Dependencies: 71_
  - _Requirements: FR-025_
  - _Complexity: Large_

- [x] 73\. JSON contract and versioning policy
  - Publish the payload schema, the field contracts including the tri-state recommendation, the exit-code mapping, and the policy for evolving the schema version.
  - Acceptance Criteria:
    - The published schema matches the schema asserted in tests.
    - The tri-state recommendation and its emission rules are documented for script authors.
    - The policy states what constitutes a breaking change to the contract.
  - _Dependencies: 71_
  - _Requirements: FR-023, IR-004_
  - _Complexity: Medium_

- [x] 74\. Interpretation guide for declared assumptions
  - Document each declared simulation assumption, its direction of bias, and how an Operator should interpret results in light of it — with particular emphasis on queue position, the short window, and why endorsement means "worth a minimum-size trial" rather than "validated".
  - Acceptance Criteria:
    - Every declared assumption in the constitution appears with its bias direction.
    - The staged-decision framing is explained, including the recommendation to run at minimum size first.
    - The deflation approximation and the in-sample dispersion proxy are explained in Operator-comprehensible terms.
  - _Dependencies: 72_
  - _Requirements: FR-024; Constitution Principle 5, Principle 8_
  - _Complexity: Large_

---

## Milestone 13 — Release Preparation

- [x] 75\. Packaging and installation verification
  - Build the wheel and verify installation via `uv tool install` and one-off execution via `uvx` on both supported platforms from a clean machine state.
  - Acceptance Criteria:
    - Installation succeeds on Linux and macOS with no compiler or system package required.
    - The installed entry point runs every command successfully.
    - Reference data ships inside the wheel and loads without network access.
  - _Dependencies: 74_
  - _Requirements: DR-003, NFR-009, SC-10_
  - _Complexity: Medium_

- [x] 76\. Release checklist and reference-data versioning
  - Establish the release procedure: lockfile refresh, reference-data version bump when fee or contract schedules change, full suite green including calibration fixtures, and measurement figures recorded for the performance criteria.
  - Acceptance Criteria:
    - The checklist requires the reference-data version to be bumped whenever the packaged schedules change.
    - A release cannot proceed with a failing guardrail audit.
    - Measured values for the performance success criteria are recorded per release rather than assumed.
  - _Dependencies: 75_
  - _Requirements: DR-003, SC-1, SC-3, SC-4, SC-11, SC-12_
  - _Complexity: Medium_
