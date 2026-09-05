# spec-driven-development-samples

A collection of sample Spec-Driven Development (SDD) specifications generated with [MySpec.dev](https://myspec.dev).

Each sample under `specs/` is a complete specification set for a software project, written before any code exists. Every sample contains the same four documents:

| File | Purpose |
| --- | --- |
| `constitution.md` | Project vision, non-negotiable principles, and technology constraints |
| `requirements.md` | What the system must do: functional and non-functional requirements |
| `solution.md` | How it will be built: architecture, design decisions, and module layout |
| `tasks.md` | An ordered, milestone-grouped implementation task list with acceptance criteria |

## Samples

- [**binance-trading-bot-optimiser**](specs/binance-trading-bot-optimiser/) — A local, non-interactive Python CLI that ingests Binance public historical market data into DuckDB and backtests or optimises Spot Grid and Futures Grid bot configurations using walk-forward validation. The design centres on discounting its own optimism: it reports fee-accurate results, applies deflation for search size, and never executes live orders or touches an account.

- [**go-htmx-template-engine**](specs/go-htmx-template-engine/) — `.ghtmx`, an open source Go template engine hard-forked from `templ` that makes htmx a first-class, compile-checked language concept. Bindings such as `hx-get` reference Go handler symbols resolved against a statically derived route table, page fragments get dual render entry points at build time, and `HX-Trigger` events are declared with typed payloads. Ships a CLI, runtime, LSP, dev server, and adapters for common Go routers.
