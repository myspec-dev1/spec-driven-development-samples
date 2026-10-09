# Project Constitution — ghtmx UI (`ghtmx-ui`)

> This document defines the non-negotiable engineering constraints for the `ghtmx-ui` project.
> Changes to this constitution require an explicit, recorded decision; individual features may not silently override it.

## Project Vision

### Problem

The [ghtmx](https://github.com/go-monolith/ghtmx) engine makes the htmx contract compile-checked: `hx-post={ handlers.CreateUser }` resolves against the application's real routes, `fragment` blocks compile to byte-identical inline and standalone entry points, and `event` declarations generate the only symbols that can emit `HX-Trigger`. What the engine deliberately does not provide is the **interaction layer** that sits on top: the accessible data table that sorts and paginates through a fragment, the modal form that re-renders its own validation errors and closes on success, the combobox that fetches options as the user types. Every ghtmx team rebuilds these patterns by hand, and each rebuild gets accessibility, focus management, CSRF wiring, and htmx-version differences slightly wrong.

A conventional component library cannot fill the gap. The engine's first carve-out makes `hx-get`, `hx-post`, `hx-put`, `hx-patch`, and `hx-delete` typed bindings: a string URL is `GHTMX-E0602`, an arbitrary expression is `GHTMX-E0601`. A component compiled inside a third-party module therefore **cannot** carry a binding to the consuming application's handlers — it can neither name them nor import the application's generated `ghtmxgen` package. Passing URLs in as parameters is exactly what the engine forbids, and routing them through attribute spreads would hide them from the checker that makes ghtmx worth using.

### Target Users

Primary audience for the MVP: **Go teams already building with ghtmx** who want production-grade, accessible interactive UI without writing JavaScript or re-deriving htmx patterns. Secondary audience: teams evaluating ghtmx who need to see a credible path from "hello world" to a real admin screen.

### Long-term Outcome

`ghtmx-ui` is an open source component kit for ghtmx applications, delivered in two tiers:

- **Primitives** — an importable Go module of accessible, themeable, htmx-free components (buttons, alerts, form fields, error summaries, live regions, the dialog host). Because they carry no `hx-*` attribute, they are valid under every htmx version the engine supports.
- **Recipes** — parameterised source templates for interactive htmx patterns (data table, active search, combobox, modal form, validated form, inline edit, tabs, load-more, login form, page shell). A CLI instantiates a recipe **into the application's own source tree**, bound to the application's own handler symbols and rendered for its pinned htmx version, so the engine checks every binding exactly as it checks hand-written templates.

Alongside both tiers ship one small behaviour module (`kit.js`) for the interactions HTML cannot express on its own — focus management, combobox and tab keyboard handling, toast rendering — and one stylesheet built on CSS custom properties.

### Product Boundary

`ghtmx-ui` is a **component kit only**. It consumes the ghtmx engine as a fixed external dependency and **does not change, fork, or extend the engine** — its template language, route discovery, diagnostics, code generation, and LSP are out of this project's scope. The kit does not own routing, sessions, persistence, or authorization; it renders HTML, wires htmx through engine bindings, and hands typed values to handlers the application writes.

### Relationship to the Engine

The engine is the **only checker**. The kit adds no parallel resolver, attribute validator, or route discoverer; its own diagnostics consume engine outputs (`ghtmx routes -json`, `ghtmx generate` diagnostics). Engine compiler packages live under `internal/` and are unimportable by design — the kit talks to the engine through its CLI and public runtime packages only.

### MVP Deliverables

- Primitives module (`ui`), typed Go helpers (`uikit`), and the kit's generated event package (`uigen`)
- Stylesheet (`kit.css`) and behaviour module (`kit.js`), embedded and served by an asset handler
- Recipe catalogue (ten recipes, each with an htmx 2 and an htmx 4 variant)
- CLI `ghtmx-ui`: `init`, `add`, `list`, `diff`, `update`, `remove`, `doctor`
- Verification harness: dual-pin fixture matrix, browser accessibility, keyboard, CSP, and no-JavaScript suites
- Gallery site with a live demo of every primitive and recipe, itself a ghtmx application

### Explicitly Out of MVP Scope

- Changes to the ghtmx engine, its diagnostics, or its LSP
- Client-side rendering, a virtual DOM, or any JavaScript framework integration
- Charts, rich-text editors, date pickers beyond the native `<input type="date">`, drag-and-drop, and file upload with progress
- A visual theme builder; theming is CSS custom properties edited by hand
- Recipes for routers beyond `net/http` and chi (the templates are router-agnostic; only handler stubs are router-specific)
- Migration tooling from other component libraries

### MVP Success Criterion

Starting from an empty ghtmx project, `ghtmx-ui init` plus `add` produce a **users admin screen** — sortable, filterable, paginated table; create and edit in a modal with server-side validation; inline rename; delete with a confirmation step; toast feedback — that passes the kit's accessibility, keyboard, strict-CSP, and no-JavaScript suites under both the htmx 2.0.10 and 4.0.0 pins, with **zero hand-written JavaScript and zero hand-written `hx-*` verb attributes**. The application developer writes only handler bodies and data access.

## Core Principles

These principles govern technical trade-offs. When they conflict with convenience, they win.

### P1 — The engine checks everything; the kit never weakens it

Every `hx-*` verb attribute the kit emits is a binding through the application's central generated package — a `<Route>Path` constant or a typed route constructor — which the engine checks for existence, verb agreement, and arity. The kit MUST NOT emit string URLs at binding sites, route verb attributes through attribute spreads, render htmx markup through `ghtmx.Raw`, or require any engine check to be silenced. Kit output that triggers an engine error diagnostic is a kit defect.

### P2 — Accessible by default, not by option

Every primitive and recipe meets **WCAG 2.2 Level AA** and follows the matching **WAI-ARIA Authoring Practices** pattern out of the box. There is no "accessible mode" flag; accessibility that must be switched on is accessibility most users never get.

### P3 — You own the code

Instantiated recipes belong to the application. The kit MUST NOT modify a file it did not create, MUST NOT overwrite local edits to a file it did create without a recoverable base and an explicit command, and MUST NOT require the application to keep vendored files pristine.

### P4 — Works without JavaScript wherever the pattern allows

Navigation-shaped recipes (data table, active search, tabs, load-more) MUST remain fully usable with JavaScript disabled, through real links and GET forms that address the same handlers. htmx is an enhancement over working HTML, not a prerequisite for it.

### P5 — Strict-CSP compatible

Kit markup and the behaviour module MUST run under a Content Security Policy with no `'unsafe-inline'` and no `'unsafe-eval'` for scripts or styles. The kit therefore never uses inline event handlers, `hx-on:*` listeners, `js:`-prefixed `hx-vals`, `hx-trigger` filter expressions, or inline `style` attributes.

### P6 — One pin, one truth

A recipe is rendered for the htmx version the application pins in `ghtmx.json`. The kit MUST NOT emit markup that is valid only under a different pin, and MUST NOT rely on attribute inheritance that the pinned version does not perform.

### P7 — Deterministic, reproducible output

The same recipe version, parameters, and pin MUST produce byte-identical files on every platform. Instantiation is a pure function of its inputs; timestamps, map iteration order, and host paths never reach generated output.

### P8 — Minimal, justified dependencies

The importable module depends on the Go standard library and the ghtmx runtime only. Every other dependency — in the CLI, tests, or gallery — requires a written justification in the dependency register.

## Technology Constraints

### Required

- **Go** — the two most recent Go releases, matching the engine's CI matrix.
- **ghtmx engine** — a declared minimum release that ships the `auth` package and htmx 4 support; the supported range is recorded in `go.mod` and checked by `ghtmx-ui doctor`.
- **htmx** — every pin the engine supports (`2.0.0`–`2.0.10`, `4.0.0`). Recipes target two **pin families**: `htmx2` (constructs available since 2.0.0) and `htmx4`.
- **Templates** — `.ghtmx` source compiled by `ghtmx generate`; generated `_ghtmx.go` files committed, matching engine convention.
- **Browser behaviour** — one hand-written ES module, no build step, no transpilation.
- **Styling** — one stylesheet in a CSS cascade layer (`@layer kit`) themed through custom properties.

### Dependency Policy

- `ui`, `uikit`, `uigen`, `assets`: Go standard library and `github.com/go-monolith/ghtmx` runtime packages only, enforced by an import-isolation gate.
- CLI: standard library plus the engine binary on `PATH`; no git, Node, or network access at runtime.
- Tests: `chromedp` for browser automation and a pinned `axe-core` script asset for accessibility scanning, isolated in a test-only module.
- Gallery: its own Go module, free to depend on chi and the engine's chi adapter.

### Disallowed

- Node, npm, or any JavaScript build tooling in the build, test, or consumption path
- JavaScript frameworks or helpers (Alpine.js, hyperscript, React islands) and CSS frameworks as a requirement
- `eval`, `new Function`, inline event handlers, `innerHTML` assignment of server-supplied text in `kit.js`
- Loading kit assets from a third-party CDN; the application serves them
- `cgo` in any kit package

## Architecture Constraints

### A1 — Two-tier delivery

Components without htmx behaviour are importable **primitives**; components with htmx behaviour are vendored **recipes**. A primitive template containing any `hx-*` attribute fails the kit's build. A recipe never ships as importable compiled code.

### A2 — Instantiate, then own

`ghtmx-ui add` renders a recipe into new files in the application, records a pristine **base copy** and a **lockfile** entry, and stops. From that moment the files are the application's. The only paths by which the kit touches them again are explicit `update` and `remove` commands, both of which preserve local edits or refuse.

### A3 — The engine is consumed, never re-implemented

The kit obtains route information from `ghtmx routes -json` and compile verdicts from `ghtmx generate`. It does not parse Go source for routes, validate `hx-*` attributes, or resolve bindings itself.

### A4 — One behaviour module, declarative wiring

All client behaviour lives in `kit.js`, attached to markup through `data-kit-*` attributes using event delegation. There are no per-component scripts and no script elements inside recipe output.

### A5 — Engine, not framework

The kit does not register routes, edit router setup files, choose a persistence layer, or own sessions. Handler stubs are opt-in, written to new files, and router registration lines are printed for the developer to place.

### A6 — Gallery is a separate module

The gallery site lives in its own Go module so that its router and deployment dependencies never enter the importable module's graph.

## Testing Approaches

The test strategy is weighted towards **what a user of the finished application experiences**: correct markup, keyboard operation, assistive-technology semantics, and behaviour under real browsers and real CSP headers.

### T1 — Golden render tests

Every primitive and every recipe instantiation (fixed parameters, each pin family) has a golden HTML snapshot. Snapshot changes require review.

### T2 — Dual-pin fixture matrix

Every recipe is instantiated into a fixture application for pins `2.0.0`, `2.0.10`, and `4.0.0`, then `ghtmx generate`, `go build`, `go vet`, and `ghtmx generate -check` must all succeed with zero engine errors and only allow-listed warnings.

### T3 — Browser behaviour suites

Headless Chromium driven by `chromedp` runs, per recipe and pin: an `axe-core` scan, the WAI-ARIA keyboard script for the pattern, a strict-CSP run that fails on any `securitypolicyviolation`, and — for navigation-shaped recipes — a JavaScript-disabled run.

### T4 — CLI behaviour tests

Lockfile, base-copy, and three-way merge behaviour is tested against fixture projects, including conflicting local edits, missing base copies, and pin changes.

### T5 — Budgets as gates

Asset size budgets, render allocation baselines, and CLI latency targets are enforced in CI; a breach fails the build.

## Coding Standards

- Go code is `gofmt`-clean, `go vet`-clean, and passes `staticcheck`; `.ghtmx` files are `ghtmx fmt -fail`-clean.
- Exported Go identifiers carry doc comments stating the accessibility contract they uphold, where one applies.
- Variant-style options are typed Go constants (`ui.ButtonDanger`), never free-form strings.
- `kit.js` is written as plain ES2020, under 400 lines, with every exported behaviour documented by the `data-kit-*` attribute that activates it.
- CSS uses logical properties (`margin-inline`, `padding-block`) so right-to-left layouts work without overrides.

## Security Constraints

### S1 — Output encoding

All dynamic text reaches HTML through engine interpolation and its context-aware escaping. `kit.js` writes server-supplied event payloads with `textContent` only.

### S2 — CSRF wiring is correct for the pin

The shell recipe wires `ghtmx/auth` CSRF headers using the form the pinned htmx version requires (`hx-headers` under htmx 2, `hx-headers:inherited` under htmx 4). Mutation recipes use unsafe HTTP methods exclusively; no recipe mutates state on GET.

### S3 — Query-state hardening

Sort keys are validated against a declared allow-list, page sizes are clamped, and filter text is length-bounded before a handler sees it. The kit hands typed values to the application and never assembles queries itself.

### S4 — Sign-in hygiene

The login recipe uses the engine's pre-session login CSRF (`auth.SetLoginCSRFCookie` / `auth.ValidLoginCSRF`), posts credentials in the request body only, sets correct `autocomplete` tokens, and renders one generic failure message so that responses do not reveal which accounts exist.

### S5 — Supply chain

Kit assets are served by the application with Subresource Integrity hashes and the request's CSP nonce. Releases ship checksums; `govulncheck` runs on every build; recipe sources are embedded in the CLI binary and hash-verified before rendering.

## Accessibility Constraints

- **Standard:** WCAG 2.2 Level AA across all output.
- **Patterns:** WAI-ARIA Authoring Practices for Dialog (Modal), Combobox (list autocomplete), Tabs, sortable Table, and Alert/Status live regions.
- **Focus:** after every swap, focus lands on a deliberate element — never on `<body>` because the focused node was replaced.
- **Announcements:** content updated by htmx that changes meaning (result counts, sort state, validation outcome) is announced through a polite live region.
- **Contrast and targets:** colour tokens meet 4.5:1 for text and 3:1 for UI components in both light and dark themes; interactive targets are at least 24×24 CSS pixels.
- **Motion:** all transitions honour `prefers-reduced-motion`; forced-colours mode keeps focus indicators and states visible.

## Performance Targets

| Metric | Target |
| --- | --- |
| `kit.js` transfer size | ≤ 6 KB gzip-compressed, as served (unminified source) |
| `kit.css` transfer size | ≤ 12 KB gzip-compressed, as served (unminified source) |
| Data-table body fragment render (50 rows × 6 columns, server only) | ≤ 1 ms p95 on the CI reference runner |
| Primitive render allocations | No increase against the recorded baseline |
| `ghtmx-ui add` (excluding `ghtmx generate`) | < 1 s |
| `ghtmx-ui doctor` on a ~100-template project | < 3 s |

## Integration Points

### ghtmx engine

Template language, `ghtmx generate`, `ghtmx generate -check`, `ghtmx routes -json`, the stable diagnostic catalogue, the runtime (`ghtmx.Component`, `ghtmx.CSRFHeader`, nonce context), the `nethttp` adapter (`Render`, `WithPage`, `Status`, `Retarget`, `Reswap`), and the `auth` package.

### htmx

Pinned per application by the engine; the kit reads the pin and renders the matching recipe variant. The kit never serves its own copy of htmx — `ghtmxgen.HTMXScript()` remains the single source.

### Browsers

Current and previous major versions of Chrome, Edge, Firefox, and Safari, with native `<dialog>` support.

## Stability and Versioning Contract

- The kit follows semantic versioning and is pre-1.0 for the MVP. Breaking changes to primitive Go APIs, the lockfile schema, or the `data-kit-*` DOM contract carry a changelog entry with a migration note.
- Each recipe carries its own version. A recipe version bump never changes already-instantiated files until the developer runs `update`.
- A published compatibility table maps each kit release to the supported ghtmx engine range and htmx pins; `doctor` enforces it.

## License

MIT, matching the ghtmx engine.
