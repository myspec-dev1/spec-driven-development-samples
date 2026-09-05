# Project Constitution — Binance Trading Bot Optimiser

## Project Vision

### Problem

Binance offers hosted Spot Grid and Futures Grid trading bots, but provides no credible way to validate a configuration before committing capital. Operators choose a price range, a grid count and a capital allocation largely by intuition, deploy, and discover only afterwards — with real money — whether the parameters suited the market. The failure is not that good configurations are unavailable; it is that there is no instrument for distinguishing a good one from a bad one in advance.

### Target Users

The primary user is a **solo retail trader running Binance grid bots** on their own machine, with their own capital, and no research infrastructure. They are technically capable enough to use a command-line tool and read a results table. They are not a quantitative researcher, and they will not independently supply the statistical scepticism this domain demands — the tool must supply it for them.

### Long-Term Outcome

A local, reproducible research harness that converts Binance's public historical market data into a defensible answer to one question: *given this symbol and this recent history, which grid configuration would actually have worked, and how much should I believe that answer?*

The second half of that question is the point. A tool that ranks configurations is easy and worthless; the same parameter sweep that finds the best configuration also guarantees that the best configuration flatters itself. The long-term outcome is a tool whose headline number is one the user can act on **because the tool has already discounted its own optimism**.

Crucially, "act on" does not mean "deploy at full size". The output of this tool is the **first stage of evidence** in a staged process that continues with live observation at minimum size (Principle 5). No historical simulation, however rigorous, can substitute for the stages that follow it.

### Explicit Non-Goals

This project does not, and in its current form will not:

- Execute, place, cancel or manage live orders.
- Connect to any authenticated Binance endpoint.
- Custody funds, hold credentials, or touch a user's account in any way.
- Offer financial advice or make forward-looking return claims.

The tool is a simulator and an analysis instrument. Every output is a statement about the past, and must be presented as such.

---

## Core Principles

These are ordered. Where two principles conflict, the earlier one wins.

### 1. A Number That Cannot Be Trusted Is Worse Than No Number

The stated success bar for this project is *fee-accurate simulation the user believes enough to deploy*. That bar cuts both ways: earning belief is the goal, and therefore **misplaced belief is the primary harm this project can cause**. A backtest is a claim about a counterfactual, and almost every way of producing that claim is wrong — with the wrong ways reliably producing better-looking results than the truth.

Consequently, when the implementation faces a choice between an assumption that flatters the result and one that penalises it, and the available data cannot distinguish them, **the pessimistic assumption is mandatory**. This is not conservatism for its own sake; it is the only asymmetry that keeps the error on the safe side of a decision involving the user's capital.

Refusing to answer is always available and is sometimes the correct output. A run that cannot produce a trustworthy figure must say so rather than produce an untrustworthy one — and, per Principle 4, must explain itself when it does.

### 2. The Out-of-Sample Result Is the Only Headline

An optimiser's in-sample winner is, by construction, the configuration that best fit the noise in that particular window. It is a selection artefact, not a finding. Therefore the number presented most prominently to the user is always the **out-of-sample** result produced by walk-forward validation. In-sample figures may be shown, but never as the headline and never without their out-of-sample counterpart adjacent to them.

#### Fold Viability — the condition that makes this principle meaningful

Walk-forward is only informative when each test fold contains enough activity for its result to mean anything. A grid configuration evaluated over a fold in which price barely crossed a grid level produces a number driven by two or three fills — noise wearing the authority of a headline, which would violate Principle 1 in the name of satisfying Principle 2.

Binding rules:

- A walk-forward fold is **reportable only if it contains at least a minimum number of completed grid cycles** (matched buy/sell pairs). This threshold is a named, configurable project constant with a conservative default; it is not a magic number scattered through the code.
- If a material proportion of a run's folds fall below the threshold, the run must report **"insufficient out-of-sample evidence"** and must not present a headline performance figure. Reporting nothing is the correct behaviour here.
- **Optimisation requires the 30-day window.** A 7-day window cannot be split into train and test folds that both satisfy fold viability, and is therefore supported for **single-configuration backtesting only**. Requesting an optimisation run over 7 days must be refused with an explanation, not silently degraded into an in-sample sweep.

### 3. The Size of the Search Is Part of the Result

A performance figure quoted without the number of configurations tried to obtain it is uninterpretable. Search enough of the parameter space and something will look excellent by luck alone; the more thorough the search, the more certain that outcome. **Trial counting is therefore mandatory bookkeeping, not an optional feature.** Every configuration evaluated — including those discarded early — counts toward the multiple-testing burden and must be recorded, because the deflated statistics are uncomputable without it.

This principle has a direct structural consequence: the search must be **bounded and declared in advance** rather than emerging from nested loops (see Architecture Constraints — Bounded Search). A search whose size is knowable before it runs is both computationally tractable and statistically interpretable; an unbounded one is neither.

### 4. "Best" Means Risk-Adjusted, and Drawdown Is a Gate Rather Than a Penalty

Configurations are ranked by **risk-adjusted return**, never by absolute return. A configuration that produced a strong return by holding a large adverse inventory through a deep drawdown is not a better configuration than a modest, stable one — it is a different and worse bet, and absolute-return ranking cannot express that.

The maximum-drawdown constraint is a **hard exclusion, not a soft penalty**. A configuration whose simulated maximum drawdown breaches the user's stated limit is removed from the ranking entirely and cannot appear in results as a winner, regardless of how attractive its return is. This distinction is constitutional because a penalty and a gate select genuinely different winners: a penalty permits a sufficiently high return to buy its way past the risk limit, which is precisely the trade the user has already declined to make by stating a limit at all.

Where drawdown is not explicitly supplied by the user, a conservative default limit applies, and the applied limit is always reported with the result.

**The applied drawdown limit is the project's single risk ceiling.** No other rule in this document may recommend, derive or imply a drawdown tolerance greater than it. Principle 5's derived thresholds are explicitly subordinate to this ceiling.

#### When Nothing Survives — the gates must be legible

Three independent gates now stack: the drawdown exclusion above, fold viability (Principle 2), and the trial budget (Bounded Search). Over a difficult symbol or a choppy window they can plausibly reject every candidate. Principle 1 endorses returning nothing in that situation — but a tool that returns an empty table without explanation will be read as broken rather than as honest, and a user who concludes the tool is broken goes back to guessing, which is the exact outcome this project exists to prevent.

Therefore, **an empty result is a first-class, fully explained outcome, not an error**:

- When no configuration survives, the tool must report **how many trials were evaluated**, and **how many were rejected by each gate** — drawdown exclusion versus failed fold viability — so the user can see which constraint bound.
- It must report the **best maximum drawdown actually achieved** against the applied limit, so the distance between what was asked for and what was available is visible.
- It must state a **concrete next action**: relax the drawdown limit to a specific stated value that would have admitted candidates, choose a different window or symbol, or accept the finding that no viable grid configuration existed for this symbol in this period.
- "No viable configuration" is a **legitimate research finding and exits as success**, not as a failure. It must be distinguishable in both human-readable and JSON output from an operational error such as a failed download or a corrupt archive.

### 5. The Output Is Evidence for a Staged Decision, Not Authorisation to Deploy

Capital exposure should grow only as evidence does. A backtest is the *first* stage of evidence, and it is structurally incapable of producing the last. Two classes of error survive every historical simulation and surface only on live data: lookahead and data-availability defects, which cannot appear in a test where the future is already present in the file; and execution reality — whether resting orders actually fill — which this tool explicitly does not model (see Declared Simulation Assumptions).

Binding rules:

- The tool must present its result as **evidence supporting a trial, not as a validated configuration**.
- Output must state that the **simulated fill rate is unverified** until compared against live fills, and that this is the most likely reason live results will fall short of simulated ones.
- Output must recommend running the selected configuration at **minimum size first**, and scaling only after realised fills and realised PnL track the simulation.

#### The invalidation condition is derived and disclosed, never solicited

A configuration handed over without a pre-declared kill criterion will be argued with rather than retired, so every result must carry one. But this is a **non-interactive, flag-driven CLI** (see Technology Constraints), and the headline journey is a single command that fetches, sweeps and prints. There is no point at which the tool may stop and ask the user a question, and reintroducing a prompt here would contradict the chosen interaction model.

The invalidation condition is therefore **computed by the tool from the winning configuration's own simulated behaviour** and emitted as part of the result:

- A **drawdown kill threshold**. This is the **tighter** of: (a) the worst drawdown the configuration exhibited across its out-of-sample folds plus a stated margin, and (b) the applied maximum-drawdown limit from Principle 4. A live drawdown exceeding this threshold means the configuration is behaving worse than anything the backtest produced, or has reached the risk ceiling the user declared — either way, the signal is to stop.
- A **minimum grid-cycle rate**, below which the market has stopped oscillating enough to suit the configuration, since a grid bot that has stopped cycling has stopped earning while remaining exposed.
- Both values are **always present in output**, in both the human-readable table and the JSON payload, and are **never obtained by prompting**.
- Optional flags may override the derived values. Overrides are recorded with the result so it remains clear whether a threshold was derived or supplied.

**Precedence is absolute and one-directional.** Principle 4's applied drawdown limit always binds. A derived threshold may only ever be *more* conservative than that limit, never less — the tool must never instruct a user to hold a position through a drawdown deeper than the one that would have disqualified the configuration from being recommended in the first place. This holds for user-supplied overrides too: an override looser than the applied limit is rejected with an explanation rather than silently honoured.

When clause (b) is the binding constraint — that is, the derived threshold was capped by the risk ceiling rather than set by observed fold behaviour — **the output must say so**, because it tells the user the configuration is operating with little headroom between its normal behaviour and its kill criterion.

Comparing live behaviour against the *distribution* the backtest produced — rather than against its single point estimate — is what makes these thresholds meaningful.

### 6. Simulate Only From Information That Existed at the Time

No decision at bar *T* may consume information from bar *T+1* or later, directly or indirectly. Lookahead bias is the most common and most damaging defect in backtesting software, and it is rarely introduced deliberately — it arrives through full-sample statistics, careless indexing, and normalisations computed over data the strategy could not have seen. This principle is enforced structurally (see Architecture Constraints) and verified by dedicated tests, not left to reviewer vigilance.

### 7. Correctness Outranks Performance

A fast wrong answer has negative value here. Optimisation of runtime is permitted only where it does not obscure the simulation logic, and never at the cost of an invariant. Where a simple implementation and a clever one differ, the simple one is correct by default and the clever one carries the burden of proof.

### 8. Limitations Are Declared, Not Buried

Every simplifying assumption that could bias results optimistically must be stated in the tool's own output, not merely in documentation the user will not read. A user who understands that fills ignore queue position is protected; a user who does not is misled by the same number. Silence about a known bias is a defect.

### 9. Reproducibility Is a Property of the System, Not a Habit

The same command against the same stored data must produce the same result, always. Ingested market data is immutable, simulation is deterministic, and any source of nondeterminism (sampling, parallel scheduling, iteration order) must be seeded or ordered explicitly. Where the search uses randomised sampling, its seed is recorded with the result and re-running with that seed reproduces the search exactly.

### 10. The Simulator Serves the Domain, Not the Framework

Grid bot mechanics — order placement, fills, inventory, fees, funding, liquidation — are the substance of this project. They belong in a pure, dependency-free core that can be reasoned about and tested in isolation. Infrastructure exists to serve that core, never to shape it.

---

## Technology Constraints

### Required

| Concern | Choice | Rationale |
|---|---|---|
| Language | **Python 3.12+** | Chosen by the user; modern typing syntax (`X \| None`, generics) available without `__future__` imports. |
| Packaging & environments | **uv** | Single tool for dependency resolution, locking and virtualenv management. A committed lockfile is mandatory. |
| Lint & format | **ruff** (both linter and formatter) | One tool, one config, no black/flake8 split. |
| Testing | **pytest** | Sole test runner. |
| Analytical storage & compute | **DuckDB** | System of record for market data and results, and the primary compute engine for aggregation (see Performance Targets). |
| Domain models | **Typed immutable models** (`@dataclass(frozen=True)` or pydantic) | Configurations and results are values, not mutable bags. |
| CLI | A declarative argument-parsing library (e.g. Typer/argparse/click) | Flag-driven interface per the interview; the specific library is a solution-design decision. |

### Interaction Model (binding)

The CLI is **non-interactive and flag-driven**. A command runs to completion without prompting. Every input the tool needs must arrive as a flag or carry a documented default; anything the tool would otherwise have to ask for must instead be derived and disclosed (as with the invalidation condition in Principle 5). This constraint exists so that runs are scriptable, reproducible and diffable, and it may not be relaxed by introducing prompts in individual commands.

### Disallowed

- **Any authenticated Binance SDK or API client.** The project consumes the public data archive only. A dependency capable of placing an order must not enter the dependency tree — this is a structural guarantee, not a policy.
- **Any credential storage, keyring, or secrets mechanism.** There are no secrets, and the absence must remain true.
- **Interactive prompts, wizards or confirmation dialogues** in the primary command paths, per the interaction model above.
- **Telemetry, analytics or crash-reporting services.** The tool reports to its user and to nobody else.
- **Hidden network fetches at import time or during simulation.** Network access is confined to the ingest layer.
- **Unpinned or floating dependency versions in the committed lock.** Reproducibility (Principle 9) begins with the environment.

### Constrained

- **DataFrame libraries** (pandas / polars) are permitted as an interchange or convenience layer, but must not become a second system of record. Where an operation can be expressed in DuckDB SQL over stored data, that is the preferred form.
- **Statistical/numerical libraries** (numpy, scipy) are permitted for metric computation over settled results, subject to the numeric-precision boundary defined in Coding Standards. Any library that fits models to data — and thereby introduces a further overfitting surface — requires explicit justification.
- **Charting/reporting dependencies** are out of MVP scope; output is a text table and JSON to stdout.

---

## Architecture Constraints

### Layered Monolith With One-Way Dependencies

A single installable package, internally divided into layers with strictly one-directional dependencies:

```
cli  →  optimisation  →  simulation (pure core)
 ↓            ↓                ↑
ingest  →  storage  ─────── (reads only)
```

Binding rules:

- **The simulation core is pure.** No network access, no filesystem access, no database handles, no logging side effects, no clock reads, no randomness. It accepts market data and a configuration as inputs and returns a result value. This purity is what makes Principle 6 (no lookahead) structurally enforceable and Principle 9 (reproducibility) automatic — a function that cannot reach outside itself cannot accidentally consume the future.
- **Market data enters the simulator as an explicitly bounded, ordered series.** The core must not be handed a handle capable of querying arbitrary time ranges, because such a handle makes lookahead possible by accident. Bounding the input is the mechanism that makes the invariant hard to violate rather than merely forbidden.
- **No layer may import from a layer above it.** The simulator must not know that a CLI exists. The storage layer must not know what a grid is.
- **Network I/O exists only in the ingest layer.** No other layer may perform outbound requests.

### Bounded Search

The optimiser's search strategy is a **first-class, explicit design decision**, never an emergent property of nested loops. The parameter space spans price range bounds, grid count, spacing mode and capital allocation, plus leverage and direction for futures; multiplied by walk-forward folds, a naïve exhaustive enumeration is computationally intractable and statistically uninterpretable at the same time.

Binding rules:

- Every optimisation run **declares and enforces a maximum trial budget before it begins**. The budget is recorded with the result.
- The search method must be a named, deliberate strategy — coarse-to-fine refinement, randomised or Latin-hypercube sampling, or staged elimination. Exhaustive enumeration is permitted **only** when the full space provably fits within the declared budget.
- The optimiser must **estimate and report the size and expected cost of the search before running it**, and must not begin a run whose cost is grossly disproportionate to the user's request without saying so.
- Rejection accounting is part of the search's output: the counts required by Principle 4's empty-result rule are produced by the search, not reconstructed afterwards.
- A bounded search makes the trial count knowable in advance, which is exactly what Principle 3's deflation arithmetic requires. The budget and the statistical haircut are the same fact viewed from two directions.

### Separation of Search From Evaluation

The optimisation layer decides *which* configurations to evaluate and *how many* were evaluated; the simulation layer decides *how well* one performs. The trial counter lives with the search, and no evaluation may occur that the counter does not observe (Principle 3).

### Strategy Registry

Bot types (**Spot Grid**, **Futures Grid**, and future additions) are implementations behind a common interface, resolved through a registry. Adding a bot type must not require modifying the optimisation loop, the CLI, or the storage schema. The interface must express, at minimum: configuration schema, parameter search space, order generation, and fill/accounting semantics.

### Typed Immutable Domain Models

Configurations, market-data slices, fills, and results are typed and immutable end to end. Configurations must be validated at construction — an invalid configuration must be impossible to hold, not merely detected later. All models must be serialisable, since results are persisted to DuckDB and emitted as JSON.

### Immutable Market Data

Ingested candles are **append-only and never rewritten in place**. Re-ingesting a period already stored must be a verified no-op or an explicit, logged replacement — never a silent overwrite. This is the point-in-time integrity guarantee: a re-run months later reads byte-identical inputs.

---

## Testing Approaches

Testing depth is **deliberately asymmetric**: rigorous where a defect corrupts the numbers, light where a defect is obvious on first use.

### Deep — Simulation Core (mandatory)

The simulator carries the project's entire credibility and is tested to a materially higher standard than any other component.

- **Golden-file tests.** Hand-verified scenarios with independently derived, known-correct outcomes: fills, fees, inventory, realised and unrealised PnL. These are computed by hand or by an independent method and committed as fixtures. A change to a golden file is a change to the meaning of the product and requires explicit justification in review.
- **Synthetic price-series tests.** Analytically constructed inputs with deducible outcomes, including at minimum:
  - A flat series → zero grid cycles, zero fees, zero PnL.
  - A monotonic ramp through the grid → an exactly predictable fill count and direction.
  - A clean oscillation across *n* levels → an exactly predictable number of matched buy/sell pairs and grid profit.
  - Price exiting the configured range → no fills beyond the boundary.
  - For futures: a series driving the position to its liquidation price → liquidation occurs at the correct level.
- **No-lookahead tests (mandatory, Principle 6).** Truncation-equivalence is the required technique: simulating over a series truncated at bar *T* must produce results for bars `0..T` identical to simulating over the full series. Any dependence on future bars breaks this equality and fails the test. This test must exist for every registered strategy type.
- **Determinism tests.** Repeated runs over identical inputs produce byte-identical results, including runs whose search uses a recorded random seed.
- **Numeric-boundary tests.** The conversion from exact arithmetic to floating point (see Coding Standards) is exercised directly, confirming that no rounding occurs before settlement.

### Deep — Guardrail Behaviour (mandatory)

The rules that make this tool honest are tested as rigorously as the arithmetic, because a guardrail that silently fails is worse than no guardrail:

- Fold viability: a run whose folds fall below the cycle threshold reports insufficient evidence and emits **no** headline figure.
- Window restriction: an optimisation request over a 7-day window is refused rather than degraded.
- Drawdown gate: a configuration breaching the drawdown limit is absent from the ranking entirely, not merely ranked lower.
- Empty-result reporting: a run in which every candidate is rejected emits per-gate rejection counts, the best drawdown achieved, a concrete next action, and a success exit status distinguishable from an operational error.
- Invalidation condition: every successful result carries derived drawdown and cycle-rate thresholds in both output formats, and no code path prompts the user for them.
- **Threshold precedence:** the derived drawdown kill threshold never exceeds the applied maximum-drawdown limit, including the case where the worst fold drawdown plus margin would exceed it; the cap is reported when it binds; and a user-supplied override looser than the applied limit is rejected rather than honoured.
- Trial budget: no evaluation escapes the trial counter, and the enforced budget is never exceeded.

### Light — All Other Layers

Ingest, storage, optimisation orchestration and CLI receive conventional happy-path unit tests plus coverage of known failure modes (missing archive file, corrupt ZIP, failed checksum, unknown symbol, empty date range). Depth here is a matter of judgement, not mandate.

### Standing Rules

- A bug found in the simulator is reproduced as a failing test before it is fixed.
- No coverage-percentage target is imposed; coverage is not evidence of correctness, and the golden-file, invariant and guardrail suites are the actual gate.

---

## Coding Standards

- **ruff** governs lint and format. Its configuration is committed; findings are errors, not warnings. Formatting is never debated in review.
- **Full type annotations** on all public functions, methods and domain models.
- **No magic numbers in simulation logic.** Fee rates, funding intervals, tick sizes, grid boundaries, the fold-viability cycle threshold, the default trial budget and the drawdown kill-threshold margin are named, sourced and defined in one place. A literal `0.001` embedded in a fill calculation is a defect regardless of correctness.

### Numeric Precision Boundary

Precision requirements are scoped to the layer they actually govern, and the boundary between them is explicit:

- **Inside the simulator's fill and accounting path**, arithmetic is exact — `Decimal` or scaled integers. Binary floats must not be used to accumulate currency, quantities or fees. This is where a rounding error becomes a wrong PnL.
- **Downstream statistical metrics computed over already-settled results** — Sharpe-style ratios, drawdown series, deflation arithmetic, aggregations in DuckDB or numpy — may and generally will use `float64`. Requiring exact arithmetic here would be pointless: these are statistical estimates, not ledger entries.
- **The conversion between the two happens in exactly one place**, in a single named, tested function invoked at the settlement boundary. Scattered casts are forbidden, and no conversion may occur before a fill is settled.
- Where DuckDB stores monetary values, the column type and its correspondence to the in-memory representation are declared in the schema, so that a round-trip through storage is lossless.

### Other Standards

- **Assumptions are documented where they are made.** Every place the simulator resolves an ambiguity in favour of pessimism carries a comment stating the ambiguity and the choice, so a future reader cannot mistake the conservative branch for a bug.
- **Errors are explicit and actionable.** A failure tells the user what failed, for which symbol and date, and what to do next. Silent fallbacks — substituting a default, skipping a missing day, interpolating a gap — are forbidden in the data path: missing data is reported, never invented.
- **A finding is not an error.** "No viable configuration" and "insufficient out-of-sample evidence" are legitimate results and must be represented as such in exit status and JSON output, distinct from operational failures.
- **Functions in the simulation core are small and single-purpose.** Order generation, fill detection, fee application and accounting are separable and separately testable.

---

## Security Constraints

The threat surface here is unusually small by design, and keeping it small is itself the primary control.

- **No credentials, ever.** The tool must never require, accept, prompt for, or store a Binance API key or secret. It reads only the public data archive. Because it holds no credentials and links no order-capable client, a compromise of the tool cannot reach the user's account.
- **Read-only, public-data-only operation.** No authenticated endpoint is contacted under any circumstance.
- **Downloaded archives are integrity-verified before ingest.** Each daily klines archive is checked against its published checksum; a mismatch aborts ingest for that file with a clear error and no partial write.
- **Archive extraction is defensive.** ZIP contents are validated before extraction — path traversal entries, unexpected members and implausible sizes are rejected rather than written.
- **All state lives in a single, user-specified local data directory.** The DuckDB database, downloaded archives and cached artefacts reside under one root the user chooses. Nothing is written outside it.
- **No outbound transmission of user data.** The tool has nothing to report and no one to report it to.

---

## Performance Targets

Per Principle 7, correctness outranks speed, and **no hard wall-clock budget is imposed on a single backtest**. Optimisation runs are different: their cost is combinatorial, and cost control there is a correctness concern as much as a performance one, because an unbounded search is also an uninterpretable one.

- **The trial budget is the primary performance control.** Runtime is governed by bounding the number of configurations evaluated (see Architecture Constraints — Bounded Search), not by micro-optimising the simulation of each one. This is the only lever with the right order of magnitude.
- **Cost is estimated and shown before a long run starts.** The user learns the planned trial count and an estimated duration up front, and is never surprised by an open-ended run.
- **Push heavy work into DuckDB.** Data loading, filtering, aggregation and metric computation over large candle sets are expressed as SQL over stored data wherever practical, rather than as Python loops. Python orchestrates; DuckDB computes.
- **Vectorise where it does not obscure meaning.** Bar-by-bar Python iteration over the simulation path is acceptable where it is the clearest expression of the mechanics; it must not be replaced by an opaque vectorised form unless the golden-file suite proves equivalence.
- **Single-process by default.** Parallelism across CPU cores is explicitly *not* a day-one requirement. It may be introduced later only if it preserves determinism (Principle 9) and reproducible result ordering. The trial budget, not parallelism, is the first answer to a slow run.
- **Ingest is incremental.** Already-stored days are never re-downloaded. Network volume dominates the first run of the one-command journey, and repeat runs must feel materially cheaper.
- **The user is told what is happening.** Long operations report progress. An unexplained multi-minute silence during a 30-day ingest or a large sweep is a usability defect even when performance is acceptable.

If a wall-clock budget is later warranted, it will be set from measured behaviour rather than guessed in advance.

---

## Integration Points

### Binance Public Market Data (`data.binance.vision`, `fapi.binance.com`)

The only external data dependencies and the only network destinations. Both are public, anonymous and read-only; every other host is refused by an explicit allowlist.

**`data.binance.vision`** — bulk historical archives:

- **Spot klines**, 1-minute interval, daily archives — e.g. `data/spot/daily/klines/BTCUSDT/1m/BTCUSDT-1m-2026-08-20.zip`.
- **USD-M futures klines**, 1-minute interval, daily archives, required for Futures Grid simulation.
- **Published checksums** for every downloaded archive.

**`fapi.binance.com`** — `GET /fapi/v1/fundingRate`:

- **Funding rate history** for USD-M futures, required because funding materially alters futures grid PnL over a 30-day window. Taken from the REST endpoint rather than the archive because the archive publishes funding only in whole-month files released after the month ends, which capped every futures window at the previous month's last day and discarded history the venue was already serving.

Integration rules: both are treated as untrusted external input; requests are rate-limited and retried with backoff on transient failure; a missing or not-yet-published day is reported explicitly, never silently skipped or interpolated. Archive responses are additionally checksummed against their published sidecar and defensively extracted. The REST endpoint publishes no checksum, so its responses are guarded by HTTPS and field-by-field validation alone — a weaker guarantee, stated plainly here because Principle 8 forbids leaving a known weakness implicit.

The allowlist is matched by equality, never by host suffix: Binance's authenticated trading routes share the `binance.com` suffix, and a suffix match would turn a two-endpoint restriction into a whole-API one.

### DuckDB (local, embedded)

System of record for ingested candles, funding rates, run metadata, trial counts, rejection counts, search seeds, applied constraints, derived invalidation thresholds and results. Embedded and file-based — no server, no network exposure. Schema migrations must be explicit and must preserve previously ingested data.

### Binance Fee and Contract Schedules (reference data)

Maker/taker fee rates, funding intervals, and contract specifications are required for fee-accurate simulation. These are versioned reference data within the project, not fetched at runtime, and the applicable values are recorded with each result so a historical run remains interpretable after the schedules change.

### Binance Grid Bot Parameter Semantics (conceptual)

Configuration semantics must correspond to the real product: price range bounds, grid count, arithmetic versus geometric spacing, capital allocation, and — for futures — leverage and direction (long/short/neutral). A configuration the user cannot actually enter into Binance's interface is not a useful output, regardless of its simulated performance.

---

## Declared Simulation Assumptions

Per Principle 8, the following simplifications bias results and are stated here and in the tool's output.

| Assumption | Direction of bias | Status |
|---|---|---|
| Limit orders fill at exactly the grid price, with no slippage. | Slightly optimistic | Accepted for MVP |
| **Queue position is not modelled** — a resting order is assumed filled when price reaches its level. Real orders may not fill on a touch. | **Optimistic, potentially materially so** | Accepted for MVP; must be surfaced in output, and is the primary reason Principle 5 requires a minimum-size live trial |
| Intrabar path is unknown from 1-minute OHLC; where fill ordering within a bar is ambiguous, the least favourable ordering is assumed. | Pessimistic (deliberate) | Mandatory |
| Fees are charged on every simulated fill at the applicable Binance rate. | Neutral / realistic | Mandatory |
| Futures simulation applies funding and models liquidation. | Neutral / realistic | Mandatory |
| Market impact and capacity limits are not modelled. Retail grid sizes are assumed small relative to available liquidity. | Optimistic at large size | Accepted; invalid for large capital |
| A 30-day window covers few market regimes and is a weak statistical sample; a 7-day window is weaker still and is therefore backtest-only. | Overstates confidence | Must trigger an explicit warning in output |

The queue-position and short-window items are the two most likely reasons a live grid bot will underperform this tool's output. They are not footnotes; they are the tool's honest disclosure of its own limits, and together they are why Principle 5 treats every result as evidence for a staged trial rather than a decision to deploy at size.
