# Implementation Tasks — ghtmx UI (`ghtmx-ui`)

## Plan Overview

**Scope:** Full MVP — every deliverable through the success criterion: from an empty ghtmx project, `ghtmx-ui init` and `add` produce a users admin screen that passes the accessibility, keyboard, strict-CSP, and no-JavaScript suites under the `2.0.10` and `4.0.0` pins, with zero hand-written JavaScript and zero hand-written `hx-*` verb attributes.

**Sequencing strategy:** Vertical slice first. Milestone 2 builds a walking skeleton — one primitive, a minimal asset handler, a minimal renderer, `init` writing the shell instance, and `add validated-form` — and proves it with one `chromedp` test of a `422` round trip under both pin families. This front-loads the decisions that are expensive to change later: `text/template` rendering of `.ghtmx` source with `[[ ]]` delimiters (D2), one source tree per pin family (D3), the dialog host living in the shell instance (D6), and `422` handling that differs by family (D9). Later milestones widen each tier against a pipeline that already works end to end.

**Executor:** Two-person team — one Go/back-end engineer (lane **G**) and one front-end/accessibility engineer (lane **F**). Dependencies are recorded to make the critical path visible and to show which chains can run concurrently. Lane G owns the engine bridge, recipe engine, project state, lifecycle commands, and platform gates; lane F owns tokens, primitives, `kit.js`, the interactive recipes, the browser suites, and the gallery. The lanes run in parallel from Milestone 2 to Milestone 6 and converge on recipes in Milestone 7; the split is given under Critical Path.

**Complexity calibration.** Sized for one engineer at the requested 2–5 day granularity:

| Label | Effort |
| --- | --- |
| Small | ~1–2 days |
| Medium | ~2–3 days |
| Large | ~4–5 days |

Anything that would exceed 5 days has been split.

**Task annotations.** Every task records its acceptance criteria, the requirements it satisfies (FR, NFR, DATA, and INT IDs), and the solution modules it touches — so no requirement is orphaned and no task is written without a spec anchor. Modules: M1 primitives (`ui`), M2 Go helpers (`uikit`), M3 kit event contract (`uigen`), M4 assets, M5 recipe catalogue, M6 recipe renderer, M7 project state, M8 engine bridge, M9 CLI and doctor, M10 verification harness, M11 gallery site. D1–D12 refer to the solution design's key decisions.

## Milestone 1 — Foundations and Repository Infrastructure

- [ ] 1\. Repository and module skeleton
  - Create the root module `github.com/go-monolith/ghtmx-ui` with `ui/`, `uikit/`, `uigen/`, `assets/` (`assets/src/kit.css`, `assets/src/kit.js`), `recipes/`, `cmd/ghtmx-ui/`, and `internal/{recipe,project,merge,enginebridge,doctor,gates}/`; create `verify/` and `gallery/` as separate Go modules (constitution A6). Declare the minimum engine release in the root `go.mod`; add MIT `LICENSE`, `README`, and `CHANGELOG`.
  - Acceptance Criteria:
    - `go build ./...` succeeds in all three modules with `CGO_ENABLED=0`.
    - Dependencies of `verify/` and `gallery/` never appear in the root module's build list.
  - _Dependencies: none_
  - _Requirements: NFR-012_
  - _Modules: M1, M2, M3, M4, M5, M6, M7, M8, M9, M10, M11_
  - _Complexity: Small_

- [ ] 2\. CI matrix baseline
  - GitHub Actions workflow running build and test for all three modules on Linux, macOS, and Windows across the two most recent Go releases, plus `gofmt -l`, `go vet`, `staticcheck`, `ghtmx fmt -fail` over every `.ghtmx` file, and `ghtmx generate -check` over the kit's committed generated code. Build the CLI for amd64 and arm64 on all three operating systems.
  - Acceptance Criteria:
    - The skeleton is green on every platform and Go version in the matrix.
    - An unformatted Go or `.ghtmx` file, a `go vet` or `staticcheck` finding, or stale generated output fails the build.
  - _Dependencies: 1_
  - _Requirements: NFR-013_
  - _Modules: M10_
  - _Complexity: Small_

- [ ] 3\. Import-isolation, htmx-free, and dependency gates
  - Implement `internal/gates`: an import-isolation check over `go list -deps` for `ui`, `uikit`, `uigen`, and `assets` (standard library and `github.com/go-monolith/ghtmx` runtime packages only); a scanner that fails on any attribute name beginning `hx-` in `ui/*.ghtmx`; and a dependency-register check requiring a written justification for every `require` in the three `go.mod` files. Run `govulncheck` on every build against an in-repo accepted-findings file.
  - Acceptance Criteria:
    - A forbidden import added to `uikit` fails CI naming the package and its import chain.
    - A test primitive containing `hx-get` fails the gate naming the file and attribute.
    - A `require` without a register entry, or an unaccepted `govulncheck` finding, fails the build.
  - _Dependencies: 2_
  - _Requirements: FR-003, NFR-012_
  - _Modules: M1, M10_
  - _Complexity: Medium_

## Milestone 2 — Walking Skeleton (End-to-End Vertical Slice)

- [ ] 4\. Engine bridge
  - Implement `internal/enginebridge`: run `ghtmx version` and compare it with the supported range declared once in the package; decode `ghtmx routes -json` into a typed route table (verb, pattern, path parameters, handler symbol, import path); read `htmxVersion` and the generated-package name from `ghtmx.json`. Route information comes only from the engine (D7, constitution A3); tests replay recorded output through a fake `ghtmx` binary on `PATH`.
  - Acceptance Criteria:
    - A missing engine binary or an out-of-range version is reported as an environment error.
    - An absent `ghtmx.json` or `htmxVersion` key yields the engine default `2.0.10`.
  - _Dependencies: 1_
  - _Requirements: FR-013, INT-001, INT-004_
  - _Modules: M8_
  - _Complexity: Medium_

- [ ] 5\. Asset handler, asset tags, and minimal Button
  - Implement `assets`: `embed.FS` of placeholder `kit.css` and `kit.js` served under content-hashed names with `Cache-Control: public, max-age=31536000, immutable`, `X-Content-Type-Options: nosniff`, correct content types, `404` for unknown or stale hashes, and `assets.Integrity(name)` returning the SHA-384 SRI value. Implement `ui.Stylesheet`, `ui.BehaviourScript` (`type="module"`, nonce read from the `ghtmx.WithNonce` context), and a minimal `ui.Button`, with generated code committed.
  - Acceptance Criteria:
    - Tag URLs and `integrity` values match the bytes the handler serves; a stale hash returns `404`.
    - The `nonce` attribute is rendered exactly when the request context carries one.
  - _Dependencies: 1_
  - _Requirements: FR-001, FR-008, FR-044, INT-002_
  - _Modules: M1, M4_
  - _Complexity: Medium_

- [ ] 6\. Minimal recipe renderer
  - Implement `internal/recipe`: load `recipes/<id>/recipe.json` from the embedded catalogue, select the `htmx2/` or `htmx4/` source tree by pin family (D3), and render with `text/template` using `[[ ]]` delimiters (D2) so engine `{ }` syntax passes through untouched; normalise output to `\n` and emit files in sorted path order (P7). Handler parameters bind through `ghtmxgen.<Route>Path` constants only at this stage.
  - Acceptance Criteria:
    - Rendering identical inputs twice yields byte-identical files.
    - `.ghtmx` braces and `@` calls in recipe sources survive rendering unchanged.
  - _Dependencies: 4_
  - _Requirements: FR-013, NFR-007, DATA-001_
  - _Modules: M5, M6_
  - _Complexity: Medium_

- [ ] 7\. Seed recipes and minimal `init` / `add`
  - Author a minimal `shell` (`@ghtmxgen.HTMXScript()`, asset tags, `<main id="main">` slot, the D6 dialog host, and the `htmx2` `responseHandling` configuration that swaps `422`) and a minimal `validated-form` (one required field, `422` on failure, `hx-status:5xx="swap:none"` in the `htmx4` variant), each in both pin families (D9). Wire `cmd/ghtmx-ui`: `init` writes `ghtmx-ui.json`, an empty lockfile, and the shell instance; `add validated-form --name <Instance>` writes instance files directly (staging arrives in task 24).
  - Acceptance Criteria:
    - On an empty ghtmx project, `init`, `add`, `ghtmx generate`, and `go build` succeed under pins `2.0.10` and `4.0.0` with zero engine errors.
    - The rendered `hx-post` binds the route's `ghtmxgen.<Route>Path` constant; no string URL or bare handler symbol appears at any binding site.
    - `ghtmx-ui.json` records instance directory, package name, default stub router, and kit version.
  - _Dependencies: 5, 6_
  - _Requirements: FR-020, FR-026, FR-070, FR-071, DATA-002_
  - _Modules: M5, M7, M9_
  - _Complexity: Large_

- [ ] 8\. Walking-skeleton fixture and first browser test
  - In `verify/`, generate a fixture application from task 7's CLI output under pins `2.0.10` and `4.0.0`, serve it from `httptest`, and drive headless Chromium with `chromedp` (D11): submit an invalid form, assert the `422` response is swapped with its error text, then submit a valid one. Add the CI job that runs the `verify` module.
  - Acceptance Criteria:
    - The test passes under both pins.
    - Removing the `htmx2` shell's `422` configuration makes the `2.0.10` run fail, proving the test detects D9 regressions.
  - _Dependencies: 7_
  - _Requirements: FR-026, NFR-006, INT-007_
  - _Modules: M10_
  - _Complexity: Large_

## Milestone 3 — Design Tokens and Primitives

- [ ] 9\. Design tokens, colour schemes, and motion
  - Write `assets/src/kit.css` inside `@layer kit`: `--kit-*` colour, typography, spacing, radius, shadow, and focus-ring tokens; light and dark schemes from `prefers-color-scheme`, pinnable with `data-theme` on `<html>`; `forced-colors: active` rules for focus, selected, and invalid states; `prefers-reduced-motion: reduce` disabling kit transitions and the htmx swap and settle classes the kit styles. Publish `assets/tokens.json` (DATA-005) with a Go contrast test, and add a stylesheet lint test in `assets` rejecting physical properties and `!important`.
  - Acceptance Criteria:
    - The contrast test fails when a text pair drops below 4.5:1 or a UI-component pair below 3:1, in either scheme.
    - `margin-left`, `padding-right`, or `!important` in `kit.css` fails the lint test.
  - _Dependencies: 5_
  - _Requirements: FR-060, FR-061, FR-062, FR-063, DATA-005_
  - _Modules: M4_
  - _Complexity: Large_

- [ ] 10\. Pass-through attributes, typed options, and golden harness
  - Define `ui.Attrs`, the typed pass-through for `class`, `id`, `data-*`, and `aria-*`, rendered on each primitive's root interactive element; a key beginning `hx-` (pointing the developer to recipes) or one colliding with the accessibility contract (`role`, generated `id`s, `aria-describedby`) makes the render return a `ghtmx.Error` before any byte is written. Variant types use a non-string underlying type (for example `type ButtonVariant uint8`) so an untyped string constant cannot convert. Add the golden HTML snapshot harness (T1) with an `-update` flag.
  - Acceptance Criteria:
    - An `hx-post` pass-through key returns a `ghtmx.Error` and zero bytes reach the response writer.
    - A compile-failure test over a `testdata` package proves `ui.ButtonProps{Variant: "danger"}` does not compile.
  - _Dependencies: 5_
  - _Requirements: FR-002, FR-004, DATA-007_
  - _Modules: M1, M10_
  - _Complexity: Medium_

- [ ] 11\. Action and content primitives
  - Implement `Button` (variants, sizes), `LinkButton`, `Alert`, `Badge`, `Card`, `VisuallyHidden`, `SkipLink`, `EmptyState`, and `Indicator`, with their `kit.css` rules and doc comments stating each accessibility contract.
  - Acceptance Criteria:
    - Every primitive and variant has a golden snapshot, zero-value props render a valid, accessible default, and the htmx-free gate passes.
    - A stylesheet test asserts every interactive primitive class sets a minimum target of 24×24 CSS pixels.
  - _Dependencies: 3, 9, 10_
  - _Requirements: FR-001, FR-002, FR-003_
  - _Modules: M1, M4_
  - _Complexity: Medium_

- [ ] 12\. Form field primitives
  - Implement `TextField`, `TextArea`, `Select`, `Checkbox`, and `RadioGroup` (`fieldset` and `legend`), deriving hint and error ids deterministically from the field name, with their `kit.css` rules.
  - Acceptance Criteria:
    - A field rendered without a label returns a render error.
    - `aria-describedby` references hint then error; `aria-invalid="true"` and error text render only when errors exist.
  - _Dependencies: 9, 10_
  - _Requirements: FR-001, FR-005_
  - _Modules: M1, M4_
  - _Complexity: Large_

- [ ] 13\. Error summary and live-region primitives
  - Implement `ErrorSummary` (heading, entries in field order, links to control ids, `data-kit-focus`), `ToastRegion` (empty polite container), and `Announcer` (visually hidden polite status region); switch `Stylesheet` and `BehaviourScript` from placeholders to the real assets.
  - Acceptance Criteria:
    - `ErrorSummary` renders nothing without errors, and each entry's fragment equals the control id the matching field primitive generates.
    - `ToastRegion` and `Announcer` render the stable ids `kit.js` targets.
  - _Dependencies: 11, 12_
  - _Requirements: FR-001, FR-006, FR-007, FR-008_
  - _Modules: M1, M4_
  - _Complexity: Medium_

## Milestone 4 — Behaviour Module (`kit.js`)

- [ ] 14\. `kit.js` core: activation, lifecycle, focus, announcements
  - Write `assets/src/kit.js` as one plain ES2020 module: delegated listeners for `data-kit-*`; one frozen `window.ghtmxKit` exposing the version; read `htmx.version` at load and map that family's lifecycle event names through a single table; keep dialog, keyboard, and toast behaviour working when htmx is absent. Implement post-swap focus (`data-kit-focus`, then the same-id element, then the swap target's nearest labelled container) and `data-kit-announce`, which clears and re-sets the `Announcer` region so repeats are spoken.
  - Acceptance Criteria:
    - Browser tests under both pin families show swapped markup activating with no application-side initialisation call.
    - After every kit swap in the test fixtures, `document.activeElement` is never `<body>`.
    - Two identical consecutive announcements both reach the region.
  - _Dependencies: 8, 13_
  - _Requirements: FR-050, FR-051, FR-052, FR-054_
  - _Modules: M4, M10_
  - _Complexity: Large_

- [ ] 15\. Dialog lifecycle and toasts
  - Dialog: content swapped into `#kit-dialog-content` opens `#kit-dialog` with `showModal()` and leaves the page behind inert; `KitDialogClose` carrying the dialog's id closes it, clears its content, and returns focus to the opener or the collection caption; modified fields trigger an in-dialog confirmation step on Escape or Cancel. Toasts: render `KitToast` payloads into `ToastRegion` with `textContent`; `success` and `info` are polite, `error` uses `role="alert"` and persists; others dismiss after 6 seconds, paused on hover or focus-within; each has a labelled dismiss button; at most three are visible.
  - Acceptance Criteria:
    - Browser tests cover open, close-by-event, focus return with the opener present and removed, and the unsaved-changes guard.
    - A toast payload containing markup renders as literal text.
    - Toast timing, pause, and the three-toast cap (oldest dismissed first) are asserted under both pin families.
  - _Dependencies: 14_
  - _Requirements: FR-036, FR-037_
  - _Modules: M4_
  - _Complexity: Large_

- [ ] 16\. Busy state and request-failure feedback
  - Set `aria-busy` on the requesting container and disable its submit controls for the request's duration (D10); `ui.Indicator` visibility keys off `[aria-busy]`, never htmx classes. On a `5xx` or network failure, raise a generic error toast and re-enable controls; leave redirects to htmx so the `auth` middleware's login redirect is never rewritten. Extend `internal/gates` to reject `eval`, `new Function`, `innerHTML`, and inline handlers in `kit.js`, and to cap it at 400 lines.
  - Acceptance Criteria:
    - Browser tests over recipe-shaped fixture markup inject `500` responses and dropped connections and assert no error body is swapped, an error toast appears, and controls re-enable.
    - A forbidden construct in `kit.js`, or a 401st line, fails the gate.
  - _Dependencies: 15_
  - _Requirements: FR-039, FR-085, NFR-003_
  - _Modules: M4, M10_
  - _Complexity: Medium_

- [ ] 17\. Keyboard patterns
  - `data-kit-combobox`: Down and Up move `aria-activedescendant`, Alt+Down opens without moving, Enter selects into the hidden and visible inputs, Escape closes or clears. `data-kit-tabs`: roving `tabindex`, arrows, Home, End, manual activation on Enter or Space. `data-kit-inline-edit`: focus and select the input on swap-in; Escape activates Cancel.
  - Acceptance Criteria:
    - Every key binding listed in FR-024, FR-027, and FR-028 has a scripted keyboard test on static fixtures under both pin families.
    - The focused element shows a visible focus indicator at every step of those tests.
  - _Dependencies: 14_
  - _Requirements: FR-024, FR-027, FR-028, FR-053, NFR-002_
  - _Modules: M4, M10_
  - _Complexity: Large_

## Milestone 5 — Go Helpers and Kit Event Contract

- [ ] 18\. Kit event contract (`uigen`) and toast helpers
  - Declare `event KitToast(level string, message string)` and `event KitDialogClose(id string)` in the kit module and generate them into the kit-owned `uigen` package (D5), committed and covered by `ghtmx generate -check`. Add `uikit.ToastSuccess`, `uikit.ToastInfo`, `uikit.ToastError`, and `uikit.CloseDialog` over `uigen.EmitKitToast` and `uigen.EmitKitDialogClose`, plus a `verify/` integration test emitting an application event and both kit events on one response.
  - Acceptance Criteria:
    - The response carries one merged `HX-Trigger` header holding all three events in emission order.
    - The integration test runs against the lowest and highest engine releases in the supported range.
  - _Dependencies: 3, 8_
  - _Requirements: FR-035, FR-038, DATA-006, INT-002_
  - _Modules: M2, M3, M10_
  - _Complexity: Medium_

- [ ] 19\. `uikit` request helpers
  - Implement `TableQuery` and `ParseTableQuery` (allow-listed sort keys, page size clamped to declared sizes, page ≥ 1, filter trimmed and truncated to 200 characters on a rune boundary, never an error); canonical `Encode` (defaults omitted, fixed key order) and `CanonicalRedirect` (`303`); `PageInfo` (item range, total pages, clamped page, current ± 2 window plus first and last with gaps, announcement text); `FormErrors` (ordered, `Empty()`) with the constraint validators generated parse functions call (required, max length, email syntax, enumerated options).
  - Acceptance Criteria:
    - A fuzz test over arbitrary query strings never panics or errors and always yields an allow-listed sort and a declared page size.
    - Table tests cover `PageInfo` windows at first, middle, and last pages and the "Showing X–Y of Z, sorted by C ascending|descending" text.
  - _Dependencies: 1_
  - _Requirements: FR-040, FR-041, FR-042, FR-043_
  - _Modules: M2_
  - _Complexity: Large_

## Milestone 6 — Recipe Engine and Project State

- [ ] 20\. CLI finding model and exit codes
  - In `internal/doctor`, define `Finding{ID, Severity, Subject, Message, Remedy}` and the single registry of `KIT-E0xx` and `KIT-W0xx` IDs; fix exit codes `0` success, `1` findings or conflicts, `2` usage, `3` environment; make every missing input a usage error naming its flag; provide the stable `-json` encoder.
  - Acceptance Criteria:
    - A duplicate or unregistered finding ID fails a registry test.
    - Commands never read stdin; a missing required flag exits `2` naming the flag.
  - _Dependencies: 7_
  - _Requirements: FR-076, FR-077_
  - _Modules: M9_
  - _Complexity: Small_

- [ ] 21\. Manifest schema, typed parameters, and pin-family selection
  - Define the `recipe.json` schema (id, version, description, parameters, dependencies, expected engine warnings, files to emit) and parameter kinds (Go identifier, Go type reference, handler reference, field list, enum, boolean); parse flags and `--params file.json` into one value; record a SHA-256 of each embedded recipe tree at build time and verify it before rendering (S5). Map pins `2.0.0`–`2.0.10` to `htmx2` and `4.0.0` to `htmx4`; anything else is `KIT-E010`.
  - Acceptance Criteria:
    - Invalid parameters (non-identifier name, unknown enum value, duplicate column key) are reported with the parameter name and expected form, and nothing is written.
    - Flag and parameter-file forms produce identical renders.
    - An unsupported pin reports `KIT-E010` naming the pin and the supported set; a tampered recipe tree fails hash verification.
  - _Dependencies: 6, 20_
  - _Requirements: FR-011, FR-013, FR-081, DATA-001_
  - _Modules: M5, M6_
  - _Complexity: Large_

- [ ] 22\. Binding inference from the route table
  - Resolve each handler parameter against the `ghtmx routes -json` table: every route binds through the application's central generated package named in `ghtmx.json` (D12) — routes without path parameters through their `<Route>Path` constant (`hx-get={ ghtmxgen.ListUsersPath }`), parameterised routes through the typed constructor (`hx-put={ ghtmxgen.RenameUser(u.ID) }`); enforce verb agreement in both directions.
  - Acceptance Criteria:
    - An unsafe-method slot given a GET handler, or the reverse, is refused before any write.
    - A renderer assertion rejects any verb attribute whose value is a string literal or a bare handler symbol, so no code path can emit a string URL or create an import edge from an instance to a handler package.
  - _Dependencies: 4, 21_
  - _Requirements: FR-012, INT-001_
  - _Modules: M6, M8_
  - _Complexity: Medium_

- [ ] 23\. Instance namespacing and change events
  - Derive every symbol, element id, CSS hook, and event from the instance name (`Users` yields `event UsersChanged()` with wire name `users-changed`); collection recipes declare the change event in the application's event contract and listen for it without `hx-trigger` filter expressions (P5). Refuse a name that collides with an existing instance before writing.
  - Acceptance Criteria:
    - Two instances of a test collection recipe in one package generate and build with no `GHTMX-E0301`, `GHTMX-E0305`, `GHTMX-E0307`, or Go redeclaration error.
    - A colliding name is refused and no file is written.
  - _Dependencies: 21_
  - _Requirements: FR-014, FR-015_
  - _Modules: M6_
  - _Complexity: Medium_

- [ ] 24\. Project state and staged atomic writes
  - Implement `internal/project`: `ghtmx-ui.json`; the schema-versioned `ghtmx-ui.lock.json` with sorted keys (name, recipe id and version, parameters, assumed routes, pin family, and per file its path and base-copy SHA-256); base copies under `.ghtmx-ui/base/` mirroring instance paths; validation naming the offending entry. Every multi-file write goes through a staging directory committed as a unit, with rollback on failure.
  - Acceptance Criteria:
    - `add` aborts before writing anything when a target path exists and is not in the lockfile, naming the path.
    - Fault injection between staged commits leaves either all files or none, on Linux, macOS, and Windows.
    - A lockfile failing schema validation, or a base copy whose hash differs from its entry, is reported with the offending entry.
  - _Dependencies: 7, 20_
  - _Requirements: FR-080, FR-083, DATA-002, DATA-003, DATA-004_
  - _Modules: M7_
  - _Complexity: Large_

- [ ] 25\. Handler stubs and assumed routes
  - `add --stub nethttp|chi` writes a new stub file covering every handler parameter absent from the route table: input parsing through `uikit`, rendering through `nethttp.Render`, `WithPage`, and `Status`, data access isolated in a marked function the developer implements, and the engine's `nav` annotation on full-page-only routes where it applies. Registration lines for Go 1.22+ `ServeMux` patterns or chi are printed, never written into router setup; `--assume "<VERB> <path>"` is the alternative and is recorded in the lockfile.
  - Acceptance Criteria:
    - With stubs registered, the instance builds and renders with no further edits, and `ghtmx generate` reports only manifest-listed warnings such as `GHTMX-W0104`.
    - A handler absent from the route table with neither `--stub` nor `--assume` is a usage error naming both flags.
    - `--stub` never modifies an existing file.
  - _Dependencies: 19, 22, 24_
  - _Requirements: FR-016, FR-018, FR-082, INT-002, INT-005_
  - _Modules: M6, M7_
  - _Complexity: Large_

- [ ] 26\. Recipe composition and complete `init` / `add`
  - Resolve declared recipe dependencies (every recipe other than `shell` requires the `shell` instance) and composition by instance name (`data-table --edit-with <ModalFormInstance>`); complete `add` over tasks 21–25 with `--dry-run` and a summary of files written, registration lines, and the next command (`ghtmx generate`); complete `init` with the engine range check and a refusal to run twice.
  - Acceptance Criteria:
    - Adding a shell-dependent recipe without a shell instance reports the missing dependency and the command that adds it.
    - A second `init` is refused with a pointer to `doctor`; an out-of-range engine exits `3`.
  - _Dependencies: 23, 25_
  - _Requirements: FR-017, FR-070, FR-071_
  - _Modules: M6, M9_
  - _Complexity: Medium_

## Milestone 7 — Recipes

- [ ] 27\. `shell` recipe
  - Complete the shell: `lang`, viewport meta, skip link to `#main`, asset tags, `<main id="main">` children slot, `@ui.ToastRegion()`, `@ui.Announcer()`, and `<dialog id="kit-dialog">` containing `#kit-dialog-content` (D6). With `--csrf`, place `ghtmx.CSRFHeader` with the token from `auth.CSRFTokenFrom` on `<body>` as `hx-headers` (`htmx2`) or `hx-headers:inherited` (`htmx4`); the `htmx2` configuration also stops htmx injecting inline styles.
  - Acceptance Criteria:
    - A test recipe with `hx-target="#kit-dialog-content"` generates with no `GHTMX-W0201`.
    - Golden snapshots for both families match FR-020 element by element; `ToastRegion` and `Announcer` appear exactly once.
    - With `--csrf`, an unsafe request from a fixture page carries the CSRF header under both pins.
  - _Dependencies: 13, 26_
  - _Requirements: FR-007, FR-020, INT-003, INT-004_
  - _Modules: M5_
  - _Complexity: Medium_

- [ ] 28\. `validated-form` recipe
  - Generate the typed values struct and a `Parse<Instance>` function returning it with `uikit.FormErrors` for the declared fields; render `ErrorSummary` and field errors on `422`; expose the success mode as a parameter (confirmation fragment, or the engine's typed redirect helpers); disable submit controls and show `ui.Indicator` while in flight.
  - Acceptance Criteria:
    - Under both pins, an invalid submission is swapped with status `422` and focus lands on the error summary.
    - A `500` response never replaces the form (`htmx2` by default, `htmx4` through `hx-status:5xx="swap:none"`).
    - Generated parse functions enforce required, max-length, email, and enumerated-option constraints.
  - _Dependencies: 16, 19, 27_
  - _Requirements: FR-026, FR-039, FR-042_
  - _Modules: M5, M6_
  - _Complexity: Medium_

- [ ] 29\. `modal-form` recipe
  - The open control fetches the form fragment into `#kit-dialog-content`; the form heading labels the dialog; Cancel and Escape close it without a request; `422` re-renders inside the dialog with `validated-form` semantics; success stubs emit `KitDialogClose`, a `KitToast`, and the composed collection's change event, then respond `204`.
  - Acceptance Criteria:
    - Browser tests under both pins cover open with first-field focus, `422` re-render, and close on success.
    - After success, focus returns to the opener, or to the collection caption when the opener was removed.
  - _Dependencies: 15, 18, 28_
  - _Requirements: FR-015, FR-025, FR-037_
  - _Modules: M5, M6_
  - _Complexity: Large_

- [ ] 30\. `data-table` recipe: markup, sorting, pagination
  - One GET form carries all table state (D8): sortable headers expose `aria-sort` and cycle ascending then descending; pagination renders previous, next, and the `PageInfo` window with `aria-current="page"`; the page-size selector offers only the allowed sizes; sort and size changes reset to page 1. The body is a `fragment` that refreshes on the instance's change event and carries a `data-kit-announce` summary.
  - Acceptance Criteria:
    - Browser tests assert `aria-sort`, `aria-current`, and the announced "Showing X–Y of Z, sorted by …" text after each update.
    - Emitting the instance's change event from another response refreshes the body.
  - _Dependencies: 14, 19, 27_
  - _Requirements: FR-015, FR-021, FR-043_
  - _Modules: M5, M6_
  - _Complexity: Large_

- [ ] 31\. `data-table` recipe: filter, URL state, and no-JS path
  - The filter input requests 300 ms after the last keystroke and supersedes any in-flight table request using the pin family's request-synchronisation construct; htmx requests swap only the body fragment and push the canonical `TableQuery` URL; stubs answer non-canonical full-page requests with `uikit.CanonicalRedirect`.
  - Acceptance Criteria:
    - A full-page load of any pushed URL renders the identical state.
    - A delayed earlier response never overwrites a later one.
    - With JavaScript disabled, every control works through the same GET form and handler.
  - _Dependencies: 30_
  - _Requirements: FR-021, FR-041, FR-084, NFR-009_
  - _Modules: M5, M6_
  - _Complexity: Medium_

- [ ] 32\. `data-table` recipe: row actions
  - Edit opens the composed `modal-form` instance for the row (`--edit-with`). Delete first renders a server-driven confirmation naming the row into the dialog host; only the confirmation's button issues the `DELETE`, and success closes the dialog, raises a success toast, emits the change event, and focuses the table caption.
  - Acceptance Criteria:
    - No `DELETE` request is issued before the confirmation step.
    - After a delete, the row is gone, a toast is announced, and focus is on the caption, under both pins.
  - _Dependencies: 29, 31_
  - _Requirements: FR-017, FR-022_
  - _Modules: M5, M6_
  - _Complexity: Medium_

- [ ] 33\. `active-search` recipe
  - A `role="search"` form with a labelled `type="search"` input and a results region; requests debounce 300 ms, supersede in-flight searches, and also fire on explicit submit; the result count is announced politely and an empty result renders `ui.EmptyState`.
  - Acceptance Criteria:
    - Browser tests assert debounce, supersession, and the count announcement under both pins.
    - With JavaScript disabled, submitting renders the results page through the same handler.
  - _Dependencies: 14, 27_
  - _Requirements: FR-023, FR-084, NFR-009_
  - _Modules: M5, M6_
  - _Complexity: Medium_

- [ ] 34\. `combobox` recipe
  - An input with `role="combobox"`, `aria-expanded`, `aria-controls`, and `aria-autocomplete="list"`; options render server-side as `role="option"` in a `role="listbox"` fragment, requested 250 ms after typing and superseding earlier requests; selection writes the value to the instance's hidden input and the label to the visible input; the option-less state announces "No results".
  - Acceptance Criteria:
    - The FR-024 keyboard script passes under both pins against server-rendered options.
    - The submitted form carries the hidden-input value, not the visible label.
  - _Dependencies: 17, 27_
  - _Requirements: FR-024, FR-084_
  - _Modules: M5, M6_
  - _Complexity: Large_

- [ ] 35\. `inline-edit` recipe
  - A view fragment showing the value and an Edit button bound through the row's generated constructor (`ghtmxgen.EditUserName(u.ID)`); an edit fragment with a labelled input, Save, and Cancel; Save returns the view fragment or `422` with the edit fragment and inline error; Cancel restores the view without a write.
  - Acceptance Criteria:
    - The edit input is focused with its contents selected, and Escape activates Cancel.
    - After save or cancel, focus is on the restored Edit button under both pins.
    - Cancel issues no unsafe-method request.
  - _Dependencies: 17, 19, 27_
  - _Requirements: FR-027_
  - _Modules: M5, M6_
  - _Complexity: Medium_

- [ ] 36\. `tabs` recipe
  - Tabs are links with `role="tab"`, each addressing its panel handler; activation swaps the panel fragment into the `role="tabpanel"` region and pushes the tab's URL; a full-page load of that URL renders the same tab selected.
  - Acceptance Criteria:
    - Roving `tabindex`, arrow, Home, and End movement and manual activation pass the keyboard script under both pins.
    - The active tab carries `aria-selected="true"` after both htmx and full-page navigation.
    - With JavaScript disabled, each tab is an ordinary link to a full page.
  - _Dependencies: 17, 27_
  - _Requirements: FR-028, NFR-009_
  - _Modules: M5, M6_
  - _Complexity: Medium_

- [ ] 37\. `load-more` recipe
  - A "Load more" button inside a GET form carries the cursor; activation appends the next page and replaces the button with the next one or an end-of-list note; `--auto` adds a viewport-entry trigger while the button remains the keyboard and no-JavaScript path.
  - Acceptance Criteria:
    - After loading, focus is on the first new item and the loaded count is announced.
    - With `--auto`, scrolling the button into view loads the next page under both pins.
    - With JavaScript disabled, the button submits the form and a full page renders the next items.
  - _Dependencies: 14, 27_
  - _Requirements: FR-029, NFR-009_
  - _Modules: M5, M6_
  - _Complexity: Medium_

- [ ] 38\. `login-form` recipe
  - The form stub obtains the login CSRF value with `auth.SetLoginCSRFCookie` and renders it in `login_csrf`; the submit stub rejects requests failing `auth.ValidLoginCSRF` before reading credentials, responds `422` with one generic message on any failure, and on success calls `auth.NewSessionToken` and `auth.SetSessionCookie` and redirects. Inputs use `autocomplete="username"` and `autocomplete="current-password"`; credentials travel only in a POST body.
  - Acceptance Criteria:
    - Unknown-user and wrong-password failures produce byte-identical response bodies.
    - A re-rendered form never repopulates the password field, and stubs never log credential values.
  - _Dependencies: 28_
  - _Requirements: FR-030, FR-085, INT-003, DATA-008_
  - _Modules: M5, M6_
  - _Complexity: Medium_

## Milestone 8 — Lifecycle Commands

- [ ] 39\. `list`, `diff`, and `remove`
  - `list` shows each embedded recipe with version, description, and pin families, and with `--instances` each instance's recipe version, pin family, and modification state. `diff [<Instance>]` compares working files to base copies, `--upstream` previews a newer recipe version, and `diff -check` exits non-zero on a missing instance file or unreadable base copy. `remove` deletes an unmodified instance through staged writes, requires `--force` otherwise, and refuses when another instance depends on it.
  - Acceptance Criteria:
    - `list` output derives solely from embedded manifests; a test recipe added to the catalogue appears without code changes.
    - `diff -check` fails on a deleted instance file or base copy.
    - Removing a modified instance without `--force`, or a shell with dependents, is refused and leaves every file in place.
  - _Dependencies: 24, 26_
  - _Requirements: FR-010, FR-072, FR-073, FR-075_
  - _Modules: M7, M9_
  - _Complexity: Medium_

- [ ] 40\. In-process three-way merge
  - Implement `internal/merge`: a line-based diff3 over base, local, and remote (D4) that yields a clean merge or explicit conflict hunks, with no git or external diff tool.
  - Acceptance Criteria:
    - Table tests cover non-overlapping edits, identical edits on both sides, overlapping conflicts, and edits at file start and end.
    - Merging is deterministic and normalises `\r\n` input to `\n`.
  - _Dependencies: 1_
  - _Requirements: FR-074_
  - _Modules: M7_
  - _Complexity: Medium_

- [ ] 41\. `update` and `--repin`
  - Merge each instance file with the base copy as ancestor, the working file as local, and a fresh render from recorded parameters as remote; write clean merges together with the new base copy and lockfile entry; leave conflicting files untouched, write conflicts under `.ghtmx-ui/conflicts/`, and exit `1` naming each. Refuse to run while lockfile or base-copy integrity fails; `--repin` re-renders for the pin family `ghtmx.json` now declares.
  - Acceptance Criteria:
    - A local edit and a non-overlapping recipe change both survive `update`.
    - A conflicting edit leaves the working file byte-identical, writes a conflict file, and exits `1`.
    - After switching a fixture from `2.0.10` to `4.0.0`, `update --repin` yields an instance that generates cleanly with local edits preserved.
  - _Dependencies: 39, 40_
  - _Requirements: FR-074, FR-083, FR-086_
  - _Modules: M6, M7, M9_
  - _Complexity: Large_

- [ ] 42\. `doctor`
  - Check the engine range; that the kit module version matches the CLI; each instance's pin family against `ghtmx.json` (`KIT-E030`); each bound handler's verb in `ghtmx routes -json`; assumed routes still unregistered (`KIT-W020`) or contradicted (`KIT-E021`); every lockfile entry's files and base copies; and the shell when a dependent recipe exists. Add `--rebuild-base <Instance>` for unmodified instances and `-strict`.
  - Acceptance Criteria:
    - Each check has a fixture project that triggers it and asserts the finding's ID, message, and remedy.
    - Errors exit non-zero; warnings exit `0` unless `-strict` is given.
    - `--rebuild-base` refuses a modified instance and restores a deleted base copy for an unmodified one.
  - _Dependencies: 22, 39_
  - _Requirements: FR-076, FR-082, FR-083, FR-086, INT-001_
  - _Modules: M8, M9_
  - _Complexity: Large_

- [ ] 43\. Automation surface
  - Freeze the `-json` schemas of `list`, `diff`, `update`, and `doctor` with golden outputs; document exit codes `0`–`3` in CLI help; run every command in CI with stdin closed and no TTY.
  - Acceptance Criteria:
    - A change to any `-json` shape fails its golden test.
    - A test asserts the help text's exit-code table matches the `internal/doctor` constants.
  - _Dependencies: 41, 42_
  - _Requirements: FR-077_
  - _Modules: M9_
  - _Complexity: Small_

## Milestone 9 — Verification Gates

- [ ] 44\. Dual-pin fixture matrix
  - Build the `verify/` generator that instantiates every recipe, with stubs, into fixture applications for pins `2.0.0`, `2.0.10`, and `4.0.0`, including two instances of each collection recipe in one package; each fixture runs `ghtmx generate`, `go build`, `go vet`, `ghtmx generate -check`, and `ghtmx fmt -fail`, and reported engine warnings are compared with the manifest's expected-warnings list. Golden snapshots cover every instance file per family (T1, T2).
  - Acceptance Criteria:
    - All ten recipes instantiate in both variants, and every fixture passes all five steps with zero engine errors.
    - A warning outside a recipe's allow-list fails the matrix.
    - Fixtures and snapshots are committed as the shared corpus for the browser suites.
  - _Dependencies: 32, 33, 34, 35, 36, 37, 38_
  - _Requirements: FR-010, FR-014, FR-018, NFR-005, NFR-006, DATA-007_
  - _Modules: M5, M10_
  - _Complexity: Large_

- [ ] 45\. Accessibility and keyboard suites
  - Embed a pinned, checksummed `axe-core` script in `verify/` and inject it with `chromedp` into every fixture screen under both pin families, in light and dark schemes and under `dir="rtl"`, emulating `forced-colors: active` and `prefers-reduced-motion: reduce` in dedicated passes. Consolidate the per-pattern keyboard scripts into one suite run per recipe and pin family on Chromium.
  - Acceptance Criteria:
    - Any `serious` or `critical` violation fails CI with the rule and selector.
    - Every key binding in FR-024, FR-025, FR-027, and FR-028 is exercised per recipe and pin family.
    - Focus and selected states stay visible in forced-colours mode, kit transitions compute to zero duration under reduced motion, and inline-start content mirrors under `dir="rtl"`.
    - Native `<dialog>`, `inert`, and cascade layers work with no polyfill, and an unlayered application rule overrides a kit rule without `!important`.
  - _Dependencies: 44_
  - _Requirements: FR-053, FR-060, FR-061, FR-062, FR-063, NFR-001, NFR-002, NFR-006, NFR-010, INT-006, INT-007_
  - _Modules: M10_
  - _Complexity: Large_

- [ ] 46\. Strict-CSP suite
  - Serve every fixture with `default-src 'self'; script-src 'self' 'nonce-…'; style-src 'self'; object-src 'none'; base-uri 'none'; frame-ancestors 'none'` (plus the htmx origin when loaded from a CDN) and a per-request nonce set through `ghtmx.WithNonce`; record every `securitypolicyviolation` event across each recipe's flows, then repeat with htmx's `allowEval` disabled.
  - Acceptance Criteria:
    - Any violation fails CI with its directive and source.
    - Both runs pass for every recipe under both pin families, including the shell's htmx configuration and toast rendering.
  - _Dependencies: 44_
  - _Requirements: FR-020, NFR-003, NFR-006_
  - _Modules: M10_
  - _Complexity: Medium_

- [ ] 47\. No-JavaScript and resilience suites
  - Run the functional flows of `data-table`, `active-search`, `tabs`, `load-more`, and `login-form` with JavaScript disabled; add resilience scenarios for every recipe: delayed earlier responses, injected `5xx` responses and dropped connections, and an expired session behind the engine's `auth` middleware.
  - Acceptance Criteria:
    - Every navigation-shaped flow completes through full-page responses with JavaScript disabled.
    - No stale response overwrites a newer one; no error body is swapped; an error toast appears and controls re-enable.
    - An htmx request after session expiry navigates the browser to the login page under both pins.
  - _Dependencies: 44_
  - _Requirements: FR-039, FR-084, FR-085, NFR-009_
  - _Modules: M10_
  - _Complexity: Medium_

- [ ] 48\. Platform gates: determinism, WebAssembly, CLI portability and latency
  - Instantiate a fixed parameter set on Linux, macOS, and Windows and compare SHA-256 manifests across runners; compile a fixture importing `ui`, `uikit`, `uigen`, and `assets` with one instance of every recipe, in `nethttp` and chi variants, for `GOOS=js GOARCH=wasm` and `GOOS=wasip1 GOARCH=wasm`; time `add` with engine invocation excluded and `doctor` on a generated ~100-template project.
  - Acceptance Criteria:
    - A byte difference across platforms, or any `\r\n` in output, fails CI.
    - Both WebAssembly targets compile; a failure names the offending package.
    - `add` completes in under 1 s and `doctor` in under 3 s on the reference runner, and the CLI test suite passes on amd64 and arm64 for all three operating systems.
  - _Dependencies: 42, 44_
  - _Requirements: NFR-007, NFR-011, NFR-013_
  - _Modules: M9, M10_
  - _Complexity: Medium_

- [ ] 49\. Asset, render, and allocation budgets
  - Measure `kit.js` and `kit.css` gzip-compressed exactly as served; the kit ships unminified source with no build step, so the budget applies to the bytes users actually download. Benchmark the data-table body fragment (50 rows × 6 columns) and record primitive allocation counts as an in-repo baseline (DATA-007).
  - Acceptance Criteria:
    - `kit.js` above 6 KB or `kit.css` above 12 KB fails CI.
    - A body-fragment p95 above 1 ms on the CI reference runner fails CI.
    - Any primitive allocation increase fails CI; baseline revisions require a reviewed commit with justification.
  - _Dependencies: 44_
  - _Requirements: NFR-004, NFR-008, DATA-007_
  - _Modules: M4, M10_
  - _Complexity: Medium_

## Milestone 10 — Gallery and Documentation

- [ ] 50\. Gallery application, primitive pages, and Workers deployment
  - Create `gallery/` as a ghtmx application on chi whose recipe instances are produced by the `ghtmx-ui` CLI itself, with one page per primitive; build it natively and for `js/wasm` on Cloudflare Workers, the same deployment model as the engine's documentation site.
  - Acceptance Criteria:
    - Every primitive has a page with a live demo.
    - The native and `js/wasm` builds serve the same pages, and CI deploys the Workers build.
  - _Dependencies: 13, 27_
  - _Requirements: FR-001, NFR-011, NFR-014, INT-008_
  - _Modules: M11_
  - _Complexity: Large_

- [ ] 51\. Gallery recipe pages and documentation-coverage check
  - Add a page per recipe with a live demo, its parameter table generated from `recipe.json`, its keyboard map, and its accessibility notes, and give primitive pages the same sections; add a CI check for missing pages or sections and run the `axe-core` suite over every gallery page.
  - Acceptance Criteria:
    - Removing a recipe page or its keyboard-map section fails CI.
    - Every gallery page has zero `serious` or `critical` `axe-core` violations in both schemes.
  - _Dependencies: 45, 50_
  - _Requirements: NFR-001, NFR-014_
  - _Modules: M10, M11_
  - _Complexity: Large_

- [ ] 52\. Reference documentation
  - Publish in the gallery: getting started (empty project to first recipe), the CLI reference, the `KIT-*` finding catalogue derived from the `internal/doctor` registry, the theming guide with the token list derived from `tokens.json`, the `data-kit-*` DOM contract, and the lockfile schema.
  - Acceptance Criteria:
    - A finding ID or token missing from the documentation fails a coverage test.
    - Every `data-kit-*` attribute `kit.js` handles is documented.
  - _Dependencies: 43, 51_
  - _Requirements: FR-060, FR-076, FR-077, NFR-014_
  - _Modules: M9, M11_
  - _Complexity: Medium_

## Milestone 11 — MVP Acceptance and Release

- [ ] 53\. Users admin reference application
  - From an empty ghtmx project, build the success-criterion screen with only `init` and `add`: `data-table` `Users` with filter, sort, pagination, `--edit-with UserForm`, and delete; `modal-form` `UserForm` for create and edit; `inline-edit` for rename; `--stub` handlers; toast feedback. The developer writes only handler bodies and data access.
  - Acceptance Criteria:
    - The application passes the accessibility, keyboard, strict-CSP, and no-JavaScript suites under pins `2.0.10` and `4.0.0`.
    - A CI check finds zero `.js` files and zero `hx-*` verb attributes outside files recorded in the lockfile.
  - _Dependencies: 45, 46, 47_
  - _Requirements: FR-021, FR-022, FR-025, FR-027, FR-036, NFR-001, NFR-002, NFR-003, NFR-006, NFR-009_
  - _Modules: M5, M9, M10_
  - _Complexity: Large_

- [ ] 54\. Screen-reader and cross-browser release pass
  - Verify every recipe with NVDA on Firefox and VoiceOver on Safari, and run the gallery and the reference application on the current and previous Chrome, Edge, Firefox, and Safari; record the results in the release checklist.
  - Acceptance Criteria:
    - Every recipe has a recorded pass for both screen-reader pairings.
    - Each defect found is fixed or recorded as a release blocker.
  - _Dependencies: 53_
  - _Requirements: NFR-001, NFR-010_
  - _Modules: M10_
  - _Complexity: Medium_

- [ ] 55\. Release automation and compatibility table
  - Tag-driven releases of checksummed `ghtmx-ui` binaries for Linux, macOS, and Windows on amd64 and arm64, gated on the full suite (`govulncheck`, matrix, browser suites, budgets, platform gates); a compatibility table mapping each release to its engine range and htmx pins, generated from the declaration `init` and `doctor` enforce; changelog entries with migration notes for breaking changes to primitive APIs, the lockfile schema, or the `data-kit-*` contract.
  - Acceptance Criteria:
    - A tagged release publishes binaries and a checksum file; any failing gate blocks it.
    - The compatibility table and `doctor` read the same range declaration.
    - Release is blocked until task 54's checklist is complete.
  - _Dependencies: 48, 52, 54_
  - _Requirements: NFR-012, NFR-013, INT-001_
  - _Modules: M9, M10_
  - _Complexity: Medium_

## Requirement Coverage Map

This map is **derived from the `_Requirements:_` annotation on each task** and must be regenerated whenever an annotation changes. A CI check regenerates it from the task list and fails on drift, so it cannot silently diverge from the tasks it describes.

### Functional requirements

| Requirement | Covered by tasks |
| --- | --- |
| FR-001 primitive catalogue | 5, 11, 12, 13, 50 |
| FR-002 typed options | 10, 11 |
| FR-003 htmx-free primitives | 3, 11 |
| FR-004 attribute pass-through guard | 10 |
| FR-005 form field association | 12 |
| FR-006 error summary | 13 |
| FR-007 live regions | 13, 27 |
| FR-008 asset tags | 5, 13 |
| FR-010 recipe catalogue | 39, 44 |
| FR-011 recipe manifest and typed parameters | 21 |
| FR-012 binding form inference | 22 |
| FR-013 pin-family rendering | 4, 6, 21 |
| FR-014 instance namespacing | 23, 44 |
| FR-015 recipe-declared change events | 23, 29, 30 |
| FR-016 opt-in handler stubs | 25 |
| FR-017 recipe composition | 26, 32 |
| FR-018 engine-clean instances | 25, 44 |
| FR-020 page shell | 7, 27, 46 |
| FR-021 data table state and navigation | 30, 31, 53 |
| FR-022 data table row actions | 32, 53 |
| FR-023 active search | 33 |
| FR-024 combobox | 17, 34 |
| FR-025 modal form | 29, 53 |
| FR-026 validated form and status semantics | 7, 8, 28 |
| FR-027 inline edit | 17, 35, 53 |
| FR-028 tabs | 17, 36 |
| FR-029 load more | 37 |
| FR-030 login form | 38 |
| FR-035 kit event contract | 18 |
| FR-036 toast semantics | 15, 53 |
| FR-037 dialog lifecycle and focus return | 15, 29 |
| FR-038 combined trigger headers | 18 |
| FR-039 request-failure feedback | 16, 28, 47 |
| FR-040 table query parsing | 19 |
| FR-041 canonical query URLs | 19, 31 |
| FR-042 form values and errors | 19, 28 |
| FR-043 pagination math and summaries | 19, 30 |
| FR-044 asset handler | 5 |
| FR-050 declarative activation | 14 |
| FR-051 version-aware htmx lifecycle | 14 |
| FR-052 focus management after swaps | 14 |
| FR-053 keyboard patterns | 17, 45 |
| FR-054 announcements | 14 |
| FR-060 design tokens | 9, 45, 52 |
| FR-061 colour schemes and forced colours | 9, 45 |
| FR-062 motion | 9, 45 |
| FR-063 right-to-left | 9, 45 |
| FR-070 `init` | 7, 26 |
| FR-071 `add` | 7, 26 |
| FR-072 `list` | 39 |
| FR-073 `diff` | 39 |
| FR-074 `update` | 40, 41 |
| FR-075 `remove` | 39 |
| FR-076 `doctor` | 20, 42, 52 |
| FR-077 automation surface | 20, 43, 52 |
| FR-080 never clobber | 24 |
| FR-081 unsupported or missing pin | 21 |
| FR-082 handler not yet registered | 25, 42 |
| FR-083 lockfile and base-copy integrity | 24, 41, 42 |
| FR-084 out-of-order responses | 31, 33, 34, 47 |
| FR-085 session expiry mid-interaction | 16, 38, 47 |
| FR-086 pin change after instantiation | 41, 42 |

### Non-functional requirements

| Requirement | Covered by tasks |
| --- | --- |
| NFR-001 accessibility conformance | 45, 51, 53, 54 |
| NFR-002 keyboard operability | 17, 45, 53 |
| NFR-003 strict CSP compatibility | 16, 46, 53 |
| NFR-004 asset size budgets | 49 |
| NFR-005 engine-clean output | 44 |
| NFR-006 dual-pin compatibility | 8, 44, 45, 46, 53 |
| NFR-007 deterministic instantiation | 6, 48 |
| NFR-008 render performance | 49 |
| NFR-009 progressive enhancement | 31, 33, 36, 37, 47, 53 |
| NFR-010 browser support | 45, 54 |
| NFR-011 WebAssembly build compatibility | 48, 50 |
| NFR-012 dependency posture | 1, 3, 55 |
| NFR-013 CLI portability and latency | 2, 48, 55 |
| NFR-014 documentation coverage | 50, 51, 52 |

### Data and integration requirements

| Requirement | Covered by tasks |
| --- | --- |
| DATA-001 recipe sources and manifests | 6, 21 |
| DATA-002 project configuration (`ghtmx-ui.json`) | 7, 24 |
| DATA-003 lockfile (`ghtmx-ui.lock.json`) | 24 |
| DATA-004 base copies | 24 |
| DATA-005 design tokens | 9 |
| DATA-006 kit event contract | 18 |
| DATA-007 verification fixtures and baselines | 10, 44, 49 |
| DATA-008 data classification | 38 |
| INT-001 ghtmx engine toolchain | 4, 22, 42, 55 |
| INT-002 ghtmx runtime and adapters | 5, 18, 25 |
| INT-003 ghtmx `auth` | 27, 38 |
| INT-004 htmx | 4, 27 |
| INT-005 routers | 25 |
| INT-006 browsers | 45 |
| INT-007 verification tooling | 8, 45 |
| INT-008 gallery deployment | 50 |

## Critical Path

### Derived critical path

Computed mechanically from the recorded `_Dependencies:_` edges — the longest chain through the graph, eighteen tasks ending at release:

```
1 → 4 → 6 → 7 → 20 → 21 → 22 → 25 → 26 → 27 → 28 → 29 → 32
  → 44 → 45 → 53 → 54 → 55
```

The recipe engine dominates the path: from the minimal renderer (6) to the complete `add` (26), seven lane-G tasks sit in series before the shell recipe can be finished, and every recipe, gate, and acceptance task hangs off them. That chain — not the primitives or `kit.js` — is where schedule risk sits. The path has equal-length variants on the same spine: 27 → 30 → 31 → 32 in place of 27 → 28 → 29 → 32; 46 or 47 in place of 45 before 53; and 45 → 51 → 52 → 55 in place of 45 → 53 → 54 → 55.

Notable secondary chains, none of which lengthen the critical path:

| Chain | Depth | Ends at |
| --- | --- | --- |
| Primitives and behaviour module: 1 → 5 → 10 → 12 → 13 → 14 → 15 → 16 | 8 | busy and failure feedback, needed by 28 at depth 11 |
| Keyboard patterns: … → 14 → 17 | 7 | keyboard handlers, needed by 34–36 at depth 11 |
| Event contract: 1 → 4 → 6 → 7 → 8 → 18 | 6 | `uigen`, needed by 29 at depth 12 |
| Lifecycle commands: … → 26 → 39 → 41 → 43 | 12 | automation surface |
| Gallery: … → 27 → 50 | 11 | primitive pages and Workers deployment |

Four tasks can start as soon as their single prerequisite lands: **19** (`uikit` helpers) and **40** (three-way merge) need only task 1, and **9** and **10** (tokens, pass-through guard) need only task 5.

**Lane split.** Lane G: 1–4, 6, 7, 19–28, 33, 37–44, 48, 49, 55. Lane F: 5, 9–18, 29–32, 34–36, 45–47, 50–54. Task 8 is done jointly. The critical path crosses lanes at 28 → 29, 32 → 44, 44 → 45, and 54 → 55. Lane F's slack through Milestones 3–5 is two dependency levels but only about a week of calendar time, because its primitive and `kit.js` run (11–18) includes four Large tasks; that week goes into drafting recipe markup against the primitives, so its Milestone 7 tasks start from reviewed markup once task 27 lands.

### Suggested execution order

This is a **narrative reading order for the two lanes**, not the derived path above. Every lane sequence respects the cross-lane edges; each phase ends on something demonstrable:

```
phase                       lane G (Go / back-end)            lane F (front-end / accessibility)
foundations                 1 → 2 → 3 → 4 → 6 → 7             5 → 9 → 10
walking skeleton proven     8 (joint)                         8 (joint)
engine | primitives, js     19 → 20 → 21 → 22 → 23 → 24       11 → 12 → 13 → 14 → 15 → 16
                              → 25 → 26                         → 17 → 18, then recipe drafts
recipes                     27 → 28 → 38 → 33 → 37            30 → 31 → 29 → 32 → 34 → 35 → 36
lifecycle commands          40 → 39 → 41 → 42 → 43            (recipe browser tests continue)
verification gates          44 → 48 → 49                      45 → 46 → 47
gallery and documentation   (gate maintenance)                50 → 51 → 52
MVP acceptance, release     55                                53 → 54
```

Tasks 8 and 53 are the two proof points: task 8 establishes that a CLI-rendered recipe round-trips a `422` under both pin families, and task 53 establishes that the MVP success criterion is met. Everything between widens the surface between those two demonstrations.
