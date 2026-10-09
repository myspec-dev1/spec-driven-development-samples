# Requirements — ghtmx UI (`ghtmx-ui`)

## Overview

`ghtmx-ui` is an open source component kit for applications built with the [ghtmx](https://ghtmx.dev) template engine. It gives ghtmx teams accessible, strict-CSP-safe, server-driven UI patterns — data tables, active search, combobox, modal and validated forms, inline edit, tabs, load-more, sign-in — without hand-written JavaScript and without weakening the engine's compile-time checks.

The kit is shaped by one engine rule. Under ghtmx's first carve-out, the five verb attributes (`hx-get`, `hx-post`, `hx-put`, `hx-patch`, `hx-delete`) accept only a handler symbol or a generated route constructor. A component compiled in a third-party module cannot name the consuming application's handlers or import its generated `ghtmxgen` package, so **a conventional importable component library cannot ship htmx behaviour that ghtmx will check**. The kit therefore delivers in two tiers:

- **Primitives** — importable, htmx-free components (`ui` package). They carry no `hx-*` attribute and are valid under every supported htmx pin.
- **Recipes** — parameterised `.ghtmx` source templates that the `ghtmx-ui` CLI instantiates into the application's own source tree, bound to the application's own handlers and rendered for its pinned htmx version. The application's `ghtmx generate` then checks every binding exactly as it checks hand-written templates.

### Scope of this document

This document specifies **what** the kit must do. Delivery structure, module layout, and tooling choices are fixed by the project constitution and elaborated in the solution design. The ghtmx engine is an external dependency; its behaviour is referenced, never re-specified. Engine names are those of the shipped upstream `github.com/go-monolith/ghtmx` API (v0.1.23 or later). Where they differ from the `go-htmx-template-engine` design sample, the constitution's Integration Points section maps one to the other.

### In scope for the MVP

Primitives module; Go helpers (`uikit`); the kit event contract (`uigen`); stylesheet and behaviour module with an asset handler; ten recipes, each in an `htmx2` and an `htmx4` variant; the `ghtmx-ui` CLI (`init`, `add`, `list`, `diff`, `update`, `remove`, `doctor`); the verification harness; the gallery site.

### Out of scope for the MVP

Changes to the ghtmx engine; client-side rendering; charts, rich-text, custom date pickers, drag-and-drop, and upload progress; a visual theme builder; handler stubs for routers other than `net/http` and chi; migration from other component libraries.

### Definition of success

From an empty ghtmx project, `ghtmx-ui init` and `add` produce a users admin screen — sortable, filterable, paginated table; create and edit in a modal with server-side validation; inline rename; delete with confirmation; toast feedback — that passes the accessibility, keyboard, strict-CSP, and no-JavaScript suites under the `2.0.10` and `4.0.0` pins with zero hand-written JavaScript and zero hand-written `hx-*` verb attributes.

## Glossary

| Term | Meaning |
| --- | --- |
| **Primitive** | An importable component in the `ui` package with no `hx-*` attribute. |
| **Recipe** | A versioned, parameterised source template for an interactive pattern, embedded in the CLI. |
| **Instance** | The files produced by one `ghtmx-ui add` of a recipe, identified by its instance name (e.g. `Users`). |
| **Pin family** | `htmx2` (pins `2.0.0`–`2.0.10`, using only constructs available since `2.0.0`) or `htmx4` (pin `4.0.0`). |
| **Base copy** | The pristine rendering of an instance file at instantiation, kept under `.ghtmx-ui/base/` for three-way updates. |
| **Lockfile** | `ghtmx-ui.lock.json`, recording every instance's recipe version, parameters, pin family, and file hashes. |
| **Behaviour module** | `kit.js`, the kit's single ES module, activated by `data-kit-*` attributes. |
| **Navigation-shaped recipe** | A recipe whose interactions only read state: data table (browsing), active search, tabs, load-more. |

## User Roles

`ghtmx-ui` is a developer tool and a library. It has **no authentication or authorization model of its own**; the login recipe wires the engine's `auth` package, and access control remains the application's responsibility.

| Role | Description | Primary interactions |
| --- | --- | --- |
| **Application Developer** | Builds a ghtmx application; instantiates recipes, writes handler bodies and data access. The primary MVP audience. | CLI, primitives API, `uikit` helpers, generated instance files |
| **Theme Author** | Adapts the kit's look to a brand. Often the same person as the developer. | Design tokens, `@layer kit` overrides |
| **End User** | A person using the finished application, including keyboard-only and screen-reader users. | Rendered markup, keyboard interaction, announcements |
| **Build / CI System** | Non-human actor running the toolchain in automation. | `doctor -json`, `diff -check`, exit codes, deterministic output |
| **Recipe Contributor** | Develops primitives, recipes, or the CLI. | Recipe manifests, fixture matrix, browser suites |

## Functional Requirements

### Primitives

#### FR-001 — Primitive catalogue

The `ui` package MUST provide the following primitives: `Button`, `LinkButton`, `Alert`, `Badge`, `Card`, `TextField`, `TextArea`, `Select`, `Checkbox`, `RadioGroup`, `ErrorSummary`, `Indicator`, `VisuallyHidden`, `SkipLink`, `EmptyState`, `ToastRegion`, `Announcer`, `Stylesheet`, and `BehaviourScript`.

Acceptance criteria:

- Each primitive is a `ghtmx.Component`-returning template callable from any application template as `@ui.Name(...)`.
- Each primitive has a gallery page and a golden render test.

#### FR-002 — Typed options

Primitive options with a closed set of values (button variant and size, alert and badge tone, field input type) MUST be typed Go constants.

Acceptance criteria:

- `@ui.Button(ui.ButtonProps{Variant: ui.ButtonDanger})` compiles; passing an untyped string where a variant is expected is a Go compile error.
- The zero value of every props struct renders a valid, accessible default.

#### FR-003 — htmx-free primitives

Primitive templates MUST NOT contain any `hx-*` attribute.

Acceptance criteria:

- A kit build gate scans every primitive template and fails on any attribute whose name begins with `hx-`.
- Every primitive renders identical markup regardless of the consuming application's htmx pin.

#### FR-004 — Attribute pass-through guard

Primitives MUST accept extra attributes (`class`, `id`, `data-*`, `aria-*`) through a typed pass-through field, and MUST refuse `hx-*` keys passed that way.

Acceptance criteria:

- Extra attributes render on the primitive's root interactive element.
- An `hx-*` key in the pass-through set makes the render return a `ghtmx.Error` naming the attribute and pointing the developer to recipes; nothing is written to the response.
- Pass-through attributes cannot replace the attributes that carry the primitive's accessibility contract (`role`, generated `id`s, `aria-describedby`); conflicting keys are a render error.

#### FR-005 — Form field association

Field primitives MUST associate label, hint, and error text with the control.

Acceptance criteria:

- Every control has a programmatic label; a field rendered without a label is a render error.
- Hint and error elements have deterministic ids derived from the field name and are referenced by `aria-describedby` in the order hint, error.
- A field with errors renders `aria-invalid="true"` and its error text; a field without errors renders neither.

#### FR-006 — Error summary

`ErrorSummary` MUST list form errors with links to the offending fields.

Acceptance criteria:

- The summary renders only when errors exist, has a heading, and lists errors in field order.
- Each entry is a link whose fragment targets the field's control id.
- The summary carries `data-kit-focus`, so the behaviour module focuses it after the form is swapped in (FR-052).

#### FR-007 — Live regions

`ToastRegion` and `Announcer` MUST provide the page's notification and announcement regions.

Acceptance criteria:

- `ToastRegion` renders an empty, polite live-region container that the behaviour module fills from kit toast events (FR-036).
- `Announcer` renders a visually hidden polite status region used by the behaviour module and by recipe responses to announce state changes (FR-054).
- Both are present exactly once per page when the shell recipe is used (FR-020).

#### FR-008 — Asset tags

`Stylesheet` and `BehaviourScript` MUST render the tags that load `kit.css` and `kit.js` from the application's asset handler (FR-044).

Acceptance criteria:

- Both tags carry the asset's content-hashed URL and an `integrity` attribute with its SHA-384 digest.
- `BehaviourScript` renders `type="module"` and the request's CSP nonce, read from the context set by `ghtmx.WithNonce`, when one is present.

### Recipes — Common Behaviour

#### FR-010 — Recipe catalogue

The CLI MUST ship the following recipes: `shell`, `data-table`, `active-search`, `combobox`, `modal-form`, `validated-form`, `inline-edit`, `tabs`, `load-more`, and `login-form`.

Acceptance criteria:

- `ghtmx-ui list` shows every recipe with its version, a one-line description, and its supported pin families.
- Every recipe provides an `htmx2` and an `htmx4` variant.

#### FR-011 — Recipe manifest and typed parameters

Each recipe MUST declare its parameters in a manifest, and the CLI MUST validate supplied parameters before writing anything.

Acceptance criteria:

- Parameter kinds include Go identifier, Go type reference, handler reference, field list, enum, and boolean.
- An invalid parameter (non-identifier name, unknown enum value, duplicate column key) is reported with the parameter name and the expected form; no file is written.
- Every parameter can be supplied as a flag or through a JSON parameter file (`--params file.json`), and the two forms are equivalent.

#### FR-012 — Binding form inference

The CLI MUST choose the binding form for each handler parameter from the engine's route table.

Acceptance criteria:

- The CLI reads `ghtmx routes -json` and binds every route through the application's central generated package, named as configured in `ghtmx.json`: the `<Route>Path` constant for a route without path parameters (`hx-get={ ghtmxgen.ListUsersPath }`) and the typed constructor for a parameterised route (`hx-put={ ghtmxgen.RenameUser(u.ID) }`).
- Recipes never emit bare handler-symbol bindings. The engine pins a symbol-bound handler in generated code, which would make the instance import the handler's package while that handler renders the instance; central-package bindings carry no such import edge, so an instance may live in any package.
- The verb attribute rendered matches the route's verb; a recipe slot that requires an unsafe method rejects a handler registered for GET, and the reverse.
- The CLI never renders a string URL into a verb attribute.

#### FR-013 — Pin-family rendering

The CLI MUST render the recipe variant that matches the application's pinned htmx version.

Acceptance criteria:

- The pin is read from `ghtmx.json` (`htmxVersion`), defaulting to the engine default `2.0.10` when the file or key is absent.
- Pins `2.0.0`–`2.0.10` render the `htmx2` variant; `4.0.0` renders the `htmx4` variant.
- The rendered pin family is recorded in the lockfile per instance.

#### FR-014 — Instance namespacing

Every symbol, element id, CSS hook, and event an instance generates MUST be derived from its instance name.

Acceptance criteria:

- Two instances of the same recipe with different names coexist in one package without engine errors (`GHTMX-E0301`, `GHTMX-E0305`, `GHTMX-E0307`) or Go redeclaration errors.
- Adding an instance whose name would collide with an existing instance is refused before any file is written.

#### FR-015 — Recipe-declared change events

Recipes that display a collection MUST declare a per-instance change event in the application's event contract and listen for it.

Acceptance criteria:

- A `data-table` instance named `Users` declares `event UsersChanged()` (wire name `users-changed`), and its container refreshes on that event.
- Mutating recipes composed with that table emit the event through the generated `ghtmxgen.EmitUsersChanged` symbol in their handler stubs.

#### FR-016 — Opt-in handler stubs

`add --stub nethttp|chi` MUST write handler stubs for every handler parameter not yet present in the route table.

Acceptance criteria:

- Stubs are written to a new file; existing files are never edited.
- Each stub parses its inputs with the matching `uikit` helper, renders through the engine's `nethttp` adapter (`Render`, `WithPage`, `Status`), and leaves data access as a clearly marked function the developer implements.
- The CLI prints the router registration lines for the developer to place; it never edits router setup.
- With stubs registered, the instance builds and renders without further edits.

#### FR-017 — Recipe composition

Recipes MUST declare dependencies on other recipes and compose through instance names.

Acceptance criteria:

- Every recipe other than `shell` requires the `shell` instance, which provides the dialog host, toast region, announcer, CSRF header wiring, and (under `htmx2`) the `422` swap rule that FR-026 relies on; adding a recipe without it reports the missing dependency and the command to add it.
- `data-table --edit-with <ModalFormInstance>` wires row edit buttons to an existing `modal-form` instance.

#### FR-018 — Engine-clean instances

Every instance MUST compile through the application's `ghtmx generate` with zero engine errors.

Acceptance criteria:

- Engine warnings are limited to those listed in the recipe manifest's expected-warnings field — for example `GHTMX-W0104` on a full-page-only route that a stub registers, which the stub marks with the engine's `nav` annotation where it can; no other warning is produced.
- `ghtmx fmt -fail` reports no changes for instance files.

### Recipe Behaviours

#### FR-020 — Page shell

`init` MUST instantiate a `shell` recipe that provides the document frame every other recipe relies on.

Acceptance criteria:

- The shell renders `lang`, viewport meta, a skip link to `#main`, `@ghtmxgen.HTMXScript()`, `@ui.Stylesheet()`, `@ui.BehaviourScript()`, a `<main id="main">` children slot, `@ui.ToastRegion()`, `@ui.Announcer()`, and a dialog host (`<dialog id="kit-dialog">` containing `#kit-dialog-content`).
- The dialog host markup lives in the shell instance — inside the application's compiled set — so that recipe `hx-target="#kit-dialog-content"` selectors match a static id and raise no `GHTMX-W0201`.
- With `--csrf`, the shell places `ghtmx.CSRFHeader(token)` on `<body>` as `hx-headers` under `htmx2` and `hx-headers:inherited` under `htmx4`, reading the token with `auth.CSRFTokenFrom`.
- The `htmx2` variant configures htmx through its `htmx-config` meta tag so that `422` responses are swapped and htmx injects no inline styles. Setting `responseHandling` replaces htmx 2's default list, so the shell restates the whole list with the `422` rule placed before the generic error rule: `[{"code":"204","swap":false},{"code":"[23]..","swap":true},{"code":"422","swap":true},{"code":"[45]..","swap":false,"error":true}]`.
- The `htmx4` variant needs no configuration for `422`: htmx 4 swaps every status except `204` and `304` by default. Its error handling is per element instead (FR-039).

#### FR-021 — Data table: state and navigation

The `data-table` recipe MUST render a sortable, filterable, paginated table whose state lives in the URL.

Acceptance criteria:

- Sortable column headers expose `aria-sort`; activating one sorts ascending, then descending; changing sort or filter resets to page 1.
- Pagination offers previous, next, and numbered pages, with `aria-current="page"` on the current page, and a page-size selector bounded by the instance's allowed sizes.
- The filter input issues a request 300 ms after the last keystroke and cancels any in-flight table request.
- An explicit Apply control follows the filter field and precedes every other submit button in the form, making it the form's default button. Pressing Enter in the filter field therefore submits through it, resetting to page 1 and keeping the current sort, with or without JavaScript. Without it, implicit submission would go through the first sort header and flip the sort.
- htmx requests swap only the table body fragment and push the canonical URL; a full-page load of that URL renders the identical state.
- With JavaScript disabled, every control works through the same GET form and handler, producing full-page responses (NFR-009).
- Each update announces "Showing X–Y of Z, sorted by C ascending|descending" through the announcer.

#### FR-022 — Data table: row actions

The `data-table` recipe MUST support optional per-row edit and delete actions.

Acceptance criteria:

- Edit opens the composed `modal-form` instance for that row (FR-017).
- Delete first renders a server-driven confirmation into the dialog host naming the row; only the confirmation's button issues the `DELETE`.
- After a successful delete the dialog closes, a success toast is raised, the change event refreshes the table, and focus moves to the table's caption.

#### FR-023 — Active search

The `active-search` recipe MUST render a search form whose results update while typing.

Acceptance criteria:

- The form has `role="search"`, a labelled `type="search"` input, and a results region.
- Requests are debounced 300 ms, cancel any in-flight search, and are also issued on explicit submit.
- The result count is announced politely; an empty result renders `ui.EmptyState`.
- With JavaScript disabled, submitting the form renders the results page through the same handler.

#### FR-024 — Combobox

The `combobox` recipe MUST implement the WAI-ARIA list-autocomplete combobox pattern backed by a server handler.

Acceptance criteria:

- The input has `role="combobox"`, `aria-expanded`, `aria-controls`, and `aria-autocomplete="list"`; options render server-side as `role="option"` inside a `role="listbox"` fragment.
- Down and Up arrows move the active option through `aria-activedescendant`, Alt+Down opens the listbox without moving the active option, Enter selects, and Escape closes the listbox (clearing the input when the listbox is already closed).
- Selecting an option writes its value to a hidden input named by the instance and its label to the visible input.
- Option requests are debounced 250 ms; a "No results" option-less state is announced.

#### FR-025 — Modal form

The `modal-form` recipe MUST render a form in the shell's dialog host, validate it on the server, and close on success.

Acceptance criteria:

- The open control fetches the form fragment into `#kit-dialog-content`; the behaviour module then opens the dialog with `showModal()` and focuses the first field.
- The dialog is labelled by the form's heading; Escape and the Cancel button close it without a request.
- Validation failure responds `422` with the re-rendered form, error summary, and field errors inside the dialog (FR-026 semantics).
- Success emits `KitDialogClose`, a `KitToast`, and the composed collection's change event, then responds `204`; focus returns to the element that opened the dialog, or to the collection's caption when that element no longer exists.
- The recipe generates a typed values struct and a parse-and-validate function (FR-042) covering the declared fields.

#### FR-026 — Validated form and status semantics

The `validated-form` recipe MUST render a standalone form with server-side validation that uses HTTP status codes truthfully.

Acceptance criteria:

- Validation failure responds `422 Unprocessable Content` with the re-rendered form; the error summary receives focus.
- Under `htmx2`, the response is swapped because the shell's `responseHandling` list includes a `422` swap rule (FR-020). Under `htmx4`, the form carries an exact `hx-status:422` swap rule, which htmx 4 matches before the `4xx` no-swap rule required by FR-039.
- Success either re-renders a confirmation fragment or redirects through the engine's typed redirect helpers, as chosen by a recipe parameter.
- Submit controls are disabled while the request is in flight, and a visible `ui.Indicator` shows progress.

#### FR-027 — Inline edit

The `inline-edit` recipe MUST let a user edit one field in place.

Acceptance criteria:

- The view fragment shows the value and an Edit button bound through the generated constructor for the row (`ghtmxgen.EditUserName(u.ID)`).
- The edit fragment shows a labelled input (focused, contents selected), Save, and Cancel; Escape activates Cancel.
- Save responds with the refreshed view fragment, or `422` with the edit fragment and inline error; Cancel restores the view fragment without a write.
- After save or cancel, focus returns to the restored Edit button.

#### FR-028 — Tabs

The `tabs` recipe MUST implement the WAI-ARIA tabs pattern with lazily loaded, addressable panels.

Acceptance criteria:

- Tabs are links with `role="tab"`; each addresses its panel handler, and the active tab carries `aria-selected="true"`.
- Arrow keys, Home, and End move focus with a roving `tabindex`; Enter or Space activates (manual activation, because each activation is a network request).
- Activation swaps the panel fragment into the `role="tabpanel"` region and pushes the tab's URL; a full-page load of that URL renders the same tab selected.
- With JavaScript disabled, each tab is an ordinary link to a full page.

#### FR-029 — Load more

The `load-more` recipe MUST render an incrementally loaded list.

Acceptance criteria:

- A "Load more" button inside a GET form carries the cursor; activating it appends the next page and replaces itself with the next button, or with an end-of-list note.
- With `--auto`, loading also starts when the button scrolls into view; the button remains the keyboard and no-JavaScript path.
- After loading, focus moves to the first new item and the number of items loaded is announced.

#### FR-030 — Login form

The `login-form` recipe MUST render a sign-in form wired to the engine's `auth` package.

Acceptance criteria:

- The form handler obtains the login CSRF value with `auth.SetLoginCSRFCookie` and renders it in the `login_csrf` hidden field; the submit stub rejects requests failing `auth.ValidLoginCSRF` before reading credentials.
- Inputs use `autocomplete="username"` and `autocomplete="current-password"`; credentials are sent only in a POST body.
- Failure responds `422` with one generic message, whatever the cause; success mints a session with `auth.NewSessionToken` and `auth.SetSessionCookie` in the stub, then redirects with `HX-Redirect` (the runtime's `ghtmx.SetRedirect`) for an htmx request or a `303` otherwise, so the destination page is never swapped into the form.
- The form works with JavaScript disabled.

### Server-Driven Feedback

#### FR-035 — Kit event contract

The kit MUST declare its own events in the kit module and expose them only through generated emitters.

Acceptance criteria:

- The kit declares `event KitToast(level string, message string)` and `event KitDialogClose(id string)`, generated into the kit's `uigen` package as `uigen.EmitKitToast` and `uigen.EmitKitDialogClose`.
- `uikit` offers typed wrappers (`uikit.ToastSuccess(w, msg)` and peers) built on those emitters; there is no string-named emission API.

#### FR-036 — Toast semantics

The behaviour module MUST render kit toasts accessibly.

Acceptance criteria:

- Toasts render inside `ToastRegion` with text set through `textContent`; `success` and `info` toasts are polite, and `error` toasts use `role="alert"`.
- Non-error toasts dismiss after 6 seconds; the timer pauses while the toast is hovered or contains focus; error toasts persist until dismissed.
- Every toast has a labelled dismiss button; at most three toasts are visible, the oldest dismissed first.

#### FR-037 — Dialog lifecycle and focus return

The behaviour module MUST manage the shell dialog host.

Acceptance criteria:

- Content swapped into `#kit-dialog-content` opens the dialog modally; the page behind it is inert.
- `KitDialogClose` with the dialog's id closes it, clears its content, and restores focus as specified in FR-025.
- Closing never discards unsaved input silently: a dialog with modified fields asks for confirmation on Escape or Cancel.

#### FR-038 — Combined trigger headers

Kit and application events emitted on one response MUST arrive as one `HX-Trigger` header.

Acceptance criteria:

- Calling `ghtmxgen.EmitUsersChanged`, `uigen.EmitKitToast`, and `uigen.EmitKitDialogClose` on one response produces a single merged header containing all three events in emission order.
- The behaviour is covered by an integration test against the engine release range the kit supports.

#### FR-039 — Request-failure feedback

Recipes MUST handle error responses and network failures without corrupting the page.

Acceptance criteria:

- A `5xx` response, a `4xx` response other than a recipe's deliberate `422` (for example the `auth` CSRF layer's `403`, or a `404` for a deleted row), or a network failure never swaps an error body into recipe content. This applies whatever element issued the request: forms, filter and search inputs, combobox inputs, tab links, inline-edit buttons, and load-more buttons.
- Under `htmx2`, the shell's `responseHandling` list (FR-020) leaves these statuses unswapped and flags them as errors.
- Under `htmx4`, which swaps every status except `204` and `304`, **every** element in a recipe that carries a verb attribute also carries `hx-status:4xx="swap:none"` and `hx-status:5xx="swap:none"`. An element that expects a validation re-render adds an exact `hx-status:422` rule, which htmx 4 matches before the `42x` and `4xx` wildcards.
- The behaviour module raises a generic error toast and re-enables the controls that were disabled for the request.
- The fixture matrix returns `403` and `500` from every recipe endpoint under each pin and asserts the content is unchanged.

### Go Helpers (`uikit`)

#### FR-040 — Table query parsing

`uikit.ParseTableQuery` MUST convert request values into a validated `TableQuery`.

Acceptance criteria:

- Sort keys outside the instance's declared allow-list are ignored in favour of the default sort.
- Page size is clamped to the declared allowed sizes; page numbers below 1 become 1; filter text is trimmed and truncated to 200 characters.
- Parsing never returns an error for user-supplied values — invalid input falls back to defaults — so a crafted URL cannot produce a 500.

#### FR-041 — Canonical query URLs

`TableQuery` MUST encode to one canonical query string.

Acceptance criteria:

- Default values are omitted and keys are emitted in a fixed order, so equal states produce byte-identical URLs.
- A full-page request with a non-canonical query string can be answered with a `303` redirect to the canonical URL through `uikit.CanonicalRedirect`; htmx requests push the canonical URL instead.

#### FR-042 — Form values and errors

`uikit` MUST provide a form error model that recipes and generated parse functions share.

Acceptance criteria:

- `FormErrors` maps field names to ordered messages, preserves field declaration order, and reports `Empty()`.
- Generated parse functions enforce the declared field constraints (required, max length, email syntax, enumerated options) and return the typed values struct together with `FormErrors`.

#### FR-043 — Pagination math and summaries

`uikit.PageInfo` MUST compute pagination state from a total count and a `TableQuery`.

Acceptance criteria:

- It yields first and last item numbers, total pages, a clamped current page, and a windowed page list (current ± 2, plus first and last, with gaps marked).
- It produces the announcement text used by FR-021.

#### FR-044 — Asset handler

The `assets` package MUST serve `kit.css` and `kit.js`.

Acceptance criteria:

- Assets are embedded with `embed.FS` and served under content-hashed file names with `Cache-Control: public, max-age=31536000, immutable`.
- Unknown or stale hashes return `404`; the handler sets `X-Content-Type-Options: nosniff` and correct content types.
- `assets.Integrity(name)` returns the SHA-384 SRI value used by FR-008.

### Behaviour Module (`kit.js`)

#### FR-050 — Declarative activation

The behaviour module MUST activate exclusively through `data-kit-*` attributes using delegated listeners.

Acceptance criteria:

- Markup swapped in by htmx is activated without any per-swap initialisation call from application code.
- The module defines no globals other than one frozen `window.ghtmxKit` object exposing its version.

#### FR-051 — Version-aware htmx lifecycle

The behaviour module MUST work with both htmx pin families from a single file.

Acceptance criteria:

- At load, the module reads `htmx.version` and subscribes to that family's lifecycle event names.
- When htmx is absent, the module still provides dialog, keyboard, and toast behaviour for server-rendered markup.

#### FR-052 — Focus management after swaps

The behaviour module MUST place focus deliberately after every swap that replaces the focused element.

Acceptance criteria:

- An element marked `data-kit-focus` in swapped content receives focus.
- Otherwise, if the previously focused element was replaced, focus moves to the element with the same id in the new content, or else to the swap target's nearest labelled container.
- Focus never falls back to `<body>` as a result of a kit swap.

#### FR-053 — Keyboard patterns

The behaviour module MUST implement the keyboard interaction of the dialog, combobox, tabs, and inline-edit patterns as specified in FR-024, FR-025, FR-027, and FR-028.

Acceptance criteria:

- Every key binding listed in those requirements is covered by a browser keyboard test under both pin families.

#### FR-054 — Announcements

Recipe responses MUST be able to announce state changes.

Acceptance criteria:

- An element marked `data-kit-announce` in swapped content has its text copied into the `Announcer` region.
- Repeated identical announcements are still spoken, by clearing and re-setting the region.

### Theming

#### FR-060 — Design tokens

The stylesheet MUST expose its design through CSS custom properties inside `@layer kit`.

Acceptance criteria:

- Colour, typography, spacing, radius, shadow, and focus-ring tokens are defined as `--kit-*` properties.
- Application CSS outside the layer overrides kit rules without `!important` or specificity tricks.

#### FR-061 — Colour schemes and forced colours

The stylesheet MUST support light and dark schemes and Windows forced-colours mode.

Acceptance criteria:

- The scheme follows `prefers-color-scheme` and can be pinned with `data-theme="light|dark"` on `<html>`.
- Under `forced-colors: active`, focus rings, selected states, and invalid states remain visible using system colours.

#### FR-062 — Motion

All kit transitions and the htmx swap and settle classes the kit styles MUST be disabled under `prefers-reduced-motion: reduce`.

#### FR-063 — Right-to-left

The stylesheet MUST lay out correctly under `dir="rtl"` using logical properties only.

### CLI

#### FR-070 — `init`

`ghtmx-ui init` MUST prepare an application for the kit.

Acceptance criteria:

- It writes `ghtmx-ui.json` (instance directory, Go package name, default stub router), the empty lockfile, and the `shell` instance.
- It verifies the engine binary is on `PATH` and its version is within the supported range.
- Running `init` twice is refused with a pointer to `doctor`.

#### FR-071 — `add`

`ghtmx-ui add <recipe> --name <Instance> [params]` MUST instantiate a recipe.

Acceptance criteria:

- It validates parameters (FR-011), infers bindings (FR-012), renders the pin-family variant (FR-013), writes new files only, stores base copies, and records the lockfile entry.
- Files are written to the instance directory configured in `ghtmx-ui.json`, or to `--dir <package dir>` for feature-per-package layouts.
- It prints the files written, any registration lines, and the next command to run (`ghtmx generate`).
- `--dry-run` prints the files that would be written without touching disk.

#### FR-072 — `list`

`ghtmx-ui list` MUST show available recipes and, with `--instances`, every instance in the project with its recipe version, pin family, and modification state.

#### FR-073 — `diff`

`ghtmx-ui diff [<Instance>]` MUST show local modifications against base copies, and with `--upstream`, the changes a newer recipe version would bring.

Acceptance criteria:

- `diff -check` exits non-zero if any instance file is missing or its base copy is unreadable, for use in CI.

#### FR-074 — `update`

`ghtmx-ui update [<Instance>…]` MUST move instances to the current recipe version through a three-way merge.

Acceptance criteria:

- The merge uses the base copy as ancestor, the working file as local, and a fresh rendering with the recorded parameters as remote.
- Clean merges are written and the base copy and lockfile updated; conflicting files are left untouched, conflicts are written to a `.ghtmx-ui/conflicts/` file alongside the original, and the command exits non-zero naming each conflict.
- `--repin` re-renders instances for the application's current pin family (FR-086).

#### FR-075 — `remove`

`ghtmx-ui remove <Instance>` MUST delete an instance's files only if they are unmodified, unless `--force` is given, and MUST refuse when another instance depends on it.

#### FR-076 — `doctor`

`ghtmx-ui doctor` MUST diagnose the project's kit health.

Acceptance criteria:

- Checks include: engine version within range; kit module version matches the CLI; every instance's pin family matches `ghtmx.json`; every handler an instance binds is registered with the expected verb in `ghtmx routes -json`; every lockfile entry has its files and base copies; the shell instance exists when a dependent recipe does.
- Each finding has a stable ID (`KIT-E0xx` error, `KIT-W0xx` warning), a message, and a remedy.
- Errors produce a non-zero exit code; warnings do not, unless `-strict` is given.

#### FR-077 — Automation surface

Every CLI command MUST be usable unattended.

Acceptance criteria:

- No command prompts; missing input is an error naming the flag.
- `-json` produces stable machine-readable output for `list`, `diff`, `update`, and `doctor`.
- Exit codes are documented: `0` success, `1` findings or conflicts, `2` usage error, `3` environment error (engine missing or out of range).

### Error and Edge Case Handling

#### FR-080 — Never clobber

The CLI MUST NOT overwrite or delete any file it did not create.

Acceptance criteria:

- If a target path exists and is not recorded in the lockfile, `add` aborts before writing any file and names the conflicting path.
- All multi-file writes are staged and committed together; an interrupted `add` or `update` leaves either all files or none.

#### FR-081 — Unsupported or missing pin

The CLI MUST fail clearly when the pinned htmx version is outside the kit's supported set.

Acceptance criteria:

- An unsupported pin is reported as `KIT-E010` naming the pin and the supported set; no file is written.

#### FR-082 — Handler not yet registered

`add` MUST support handlers the developer has not written yet.

Acceptance criteria:

- When a handler parameter is absent from the route table, `add` requires either `--stub` or an `--assume "<VERB> <path>"` declaration for it; the assumption is recorded in the lockfile.
- `doctor` reports `KIT-W020` while an assumed route is still unregistered, and `KIT-E021` once a registered route contradicts the assumption.

#### FR-083 — Lockfile and base-copy integrity

The CLI MUST detect corrupted or inconsistent kit state.

Acceptance criteria:

- A lockfile that fails schema validation, or whose recorded hashes do not match its base copies, is reported with the offending entry; `update` refuses to run until it is resolved.
- `doctor --rebuild-base <Instance>` re-renders the base copy from recorded parameters when the instance is unmodified.

#### FR-084 — Out-of-order responses

Recipes MUST never display stale results.

Acceptance criteria:

- Table, search, and combobox requests from the same control cancel or supersede earlier in-flight requests, so a slow earlier response never overwrites a newer one.

#### FR-085 — Session expiry mid-interaction

Recipes MUST preserve the engine `auth` middleware's htmx-aware login redirect.

Acceptance criteria:

- When a session expires and an htmx request from any recipe is redirected to login, the browser navigates to the login page rather than swapping it into a fragment target.
- No recipe or `kit.js` behaviour suppresses or rewrites the middleware's redirect.

#### FR-086 — Pin change after instantiation

The kit MUST handle an application changing its htmx pin family.

Acceptance criteria:

- `doctor` reports `KIT-E030` for every instance rendered for a different pin family than `ghtmx.json` now declares.
- `update --repin` re-renders those instances through the three-way merge, preserving local edits that apply cleanly.

## Non-Functional Requirements

### NFR-001 — Accessibility conformance

All primitive and recipe output MUST conform to **WCAG 2.2 Level AA**.

- Zero `serious` or `critical` `axe-core` violations on every gallery page and fixture screen, under both pin families, in light and dark schemes.
- Manual screen-reader verification (NVDA with Firefox, VoiceOver with Safari) of every recipe before each minor release, recorded in the release checklist.

### NFR-002 — Keyboard operability

Every interaction MUST be operable by keyboard alone, with a visible focus indicator that is never obscured by kit-owned overlays.

- Verified by scripted keyboard tests per recipe and pin family.

### NFR-003 — Strict CSP compatibility

The kit MUST produce zero CSP violations under: `default-src 'self'; script-src 'self' 'nonce-…'; style-src 'self'; object-src 'none'; base-uri 'none'; frame-ancestors 'none'` (with the htmx origin added when the application loads htmx from a CDN).

- Verified by a browser suite that fails on any `securitypolicyviolation` event.
- Under `htmx2`, kit behaviour MUST also hold with `htmx.config.allowEval` set to `false`. htmx 4.0.0 has no `allowEval` option, so under `htmx4` the guarantee is the policy itself: no `'unsafe-eval'`, enforced by the same suite for both families.

### NFR-004 — Asset size budgets

`kit.js` MUST be ≤ **6 KB** and `kit.css` ≤ **12 KB**, gzip-compressed exactly as served. The kit ships unminified source with no build step, so the budget applies to the bytes users download.

- Enforced in CI; a breach fails the build.

### NFR-005 — Engine-clean output

Every recipe instance in the fixture matrix MUST pass `ghtmx generate` with zero errors, `go build`, `go vet`, `ghtmx generate -check`, and `ghtmx fmt -fail`; engine warnings MUST be limited to the recipe's documented allow-list.

### NFR-006 — Dual-pin compatibility

Every recipe MUST pass NFR-001 through NFR-005 under pins `2.0.0`, `2.0.10`, and `4.0.0`.

### NFR-007 — Deterministic instantiation

Identical recipe version, parameters, and pin MUST produce byte-identical files on Linux, macOS, and Windows.

- Verified by cross-platform hash comparison in CI; line endings are always `\n`.

### NFR-008 — Render performance

- Data-table body fragment (50 rows × 6 columns): ≤ **1 ms** p95 server render on the CI reference runner.
- Primitive render allocation counts MUST NOT increase against the recorded baseline; baseline revisions require a reviewed commit with justification.

### NFR-009 — Progressive enhancement

Navigation-shaped recipes MUST pass their functional browser tests with JavaScript disabled.

### NFR-010 — Browser support

The current and previous major versions of Chrome, Edge, Firefox, and Safari MUST be supported; browser tests run on Chromium in CI, and the other engines are verified before each minor release.

### NFR-011 — WebAssembly build compatibility

An application importing `ui`, `uikit`, `uigen`, and `assets`, with one instance of every recipe and the `nethttp` or chi adapter, MUST compile for `GOOS=js GOARCH=wasm` and `GOOS=wasip1 GOARCH=wasm`, matching the engine's own guarantee.

### NFR-012 — Dependency posture

- `ui`, `uikit`, `uigen`, and `assets` import only the standard library and ghtmx runtime packages, enforced by an import-isolation gate.
- `govulncheck` is clean on every build, or each finding is accepted with a recorded rationale.

### NFR-013 — CLI portability and latency

- The CLI MUST run on Linux, macOS, and Windows (amd64 and arm64).
- `add` completes in under **1 s** excluding engine invocation; `doctor` completes in under **3 s** on a ~100-template project.

### NFR-014 — Documentation coverage

Every primitive and recipe MUST have a gallery page with a live demo, its parameters, its keyboard map, and its accessibility notes; a CI check fails when one is missing.

## Data Requirements

The kit persists no end-user data and operates no database. "Data" here means the kit's own artifacts and the project-state files it writes into an application.

### DATA-001 — Recipe sources and manifests

Each recipe is a directory containing `recipe.json` (id, version, description, parameter schema, dependencies, expected engine warnings, files to emit) and one source tree per pin family. Recipe sources are embedded in the CLI binary with a content hash verified before rendering.

### DATA-002 — Project configuration (`ghtmx-ui.json`)

Instance directory, Go package name for instances, default stub router, and the kit version that initialised the project. Read alongside the engine's `ghtmx.json`, which remains the single source of the htmx pin and generated-package name.

### DATA-003 — Lockfile (`ghtmx-ui.lock.json`)

Schema-versioned. Per instance: name, recipe id and version, parameters, assumed routes, pin family, and for each file its path and the SHA-256 of its base copy. Keys are sorted so that the file diffs cleanly under version control.

### DATA-004 — Base copies

Pristine renderings of each instance file under `.ghtmx-ui/base/`, mirroring instance paths, committed to version control so that updates work on any clone.

### DATA-005 — Design tokens

The `--kit-*` custom property set for light and dark schemes, with a machine-readable token manifest used by the contrast test.

### DATA-006 — Kit event contract

The `KitToast` and `KitDialogClose` declarations and their generated `uigen` package, committed with the module.

### DATA-007 — Verification fixtures and baselines

Fixture applications per recipe and pin, golden HTML snapshots, render allocation baselines, and asset size records.

### DATA-008 — Data classification

The kit handles no personal data itself. The `login-form` recipe renders credential fields whose values pass straight to application handlers; the kit never logs, stores, or echoes them, and a re-rendered login form never repopulates the password field.

## Integration Requirements

### INT-001 — ghtmx engine toolchain

- The CLI invokes `ghtmx routes -json` for binding inference and verification, and relies on `ghtmx generate`, `generate -check`, and `fmt -fail` for verdicts (FR-012, FR-076, NFR-005).
- The supported engine release range — upstream `github.com/go-monolith/ghtmx` v0.1.23 or later — is declared once and enforced by `init` and `doctor`.
- The kit depends on the following engine diagnostics, and only as documented in the engine's diagnostic catalogue. This table is the single list; other sections cite codes from it.

| Code | Why the kit depends on it |
| --- | --- |
| `GHTMX-E0101` | A binding names a route that does not exist; how an unwritten handler or a renamed one surfaces (FR-082, D12) |
| `GHTMX-E0301`, `GHTMX-E0305`, `GHTMX-E0307` | Fragment, event, and template name collisions that instance namespacing must avoid (FR-014) |
| `GHTMX-E0402` | An unresolvable route registration, surfaced verbatim by the engine bridge |
| `GHTMX-E0601`, `GHTMX-E0602`, `GHTMX-E0603` | The carve-out rules that make importable htmx components uncheckable (D1) |
| `GHTMX-W0102` | A declared event with no template listener; suppressed only in the kit module's own build (M3) |
| `GHTMX-W0104` | A route never bound from a template; the only warning a recipe manifest may list as expected (FR-018) |
| `GHTMX-W0201` | A constant `hx-target` matching no static id; why the dialog host lives in the shell instance (FR-020, D6) |
| `GHTMX-W0202` | An htmx 4 inheritable attribute without `:inherited`; why the shell's CSRF header uses `hx-headers:inherited` (FR-020) |

### INT-002 — ghtmx runtime and adapters

- Primitives return `ghtmx.Component`; asset tags read the nonce set by `ghtmx.WithNonce`.
- Handler stubs render through the `nethttp` adapter's `Render`, `WithPage`, `Status`, `Retarget`, and `Reswap`; chi stubs use the same calls, since chi routes are `net/http` handlers.
- Cross-package event emission relies on the runtime merging every emission into one `HX-Trigger` header (FR-038).

### INT-003 — ghtmx `auth`

- The shell wires `ghtmx.CSRFHeader` with the token from `auth.CSRFTokenFrom` (FR-020).
- The login recipe uses `auth.SetLoginCSRFCookie`, `auth.ValidLoginCSRF`, `auth.NewSessionToken`, and `auth.SetSessionCookie` (FR-030).
- Recipes preserve the middleware's htmx-aware login redirect (FR-085).

### INT-004 — htmx

- The kit supports every pin the engine supports within its declared range and renders by pin family (FR-013).
- htmx itself is always loaded through `ghtmxgen.HTMXScript()`; the kit never ships or loads htmx.

### INT-005 — Routers

- Recipe templates are router-agnostic. Handler stubs and registration lines are provided for `net/http` (Go 1.22+ `ServeMux` patterns) and chi.

### INT-006 — Browsers

- Native `<dialog>`, `inert`, and CSS cascade layers are required platform features; no polyfills ship.

### INT-007 — Verification tooling

- Browser suites use `chromedp` and a pinned, checksummed `axe-core` script, both confined to the test module.

### INT-008 — Gallery deployment

- The gallery is a ghtmx application that runs natively and, compiled to `js/wasm`, on Cloudflare Workers — the same deployment model as the engine's documentation site.
