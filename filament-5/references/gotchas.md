# Filament 5 — Gotchas

Behaviours that are easy to get wrong and hard to diagnose, because the failure is silent:
the code looks right, the tests pass, and the browser disagrees. Each entry gives the
symptom first, because the symptom is what you arrive with.

Measured against **filament/filament 5.7–5.8**, **livewire/livewire 4.4**, Laravel 13,
PHP 8.5, September 2026. Several of these are version-specific and will stop being true;
re-measure before trusting one against a later release.

## The principle behind most of this page

**A test that drives a component through a door the browser does not use proves nothing
about the browser.** Setting a property, calling `mount()` directly, or asserting on a
component you instantiated yourself all bypass the request cycle that the real page runs
through. When what you are testing is Livewire lifecycle or rendered markup, the
assertion has to cross a real request boundary — `call(...)`, `callSchemaComponentMethod`,
or rendering the schema that the page actually renders.

Every silent failure below was first found in a browser, with a green suite.

---

## Tables

### A `->records()` table has no query builder, so summarizers do nothing

`Sum::make()` and `Summarizer::using()` run against a query builder. A table fed by
`->records()` has none. Put the total in the table's `->description()` or heading instead
— native, no CSS, and assertable.

### A hand-built paginator must carry the table's page name, or the pager goes inert

Filament's pager writes `gotoPage(n, '<name>')` into every button from **the paginator's**
`getPageName()`, while the table reads **`getTablePaginationPageName()`**. A
`new LengthAwarePaginator(...)` defaults to Laravel's `page`, so the moment the table
declares its own page name the two diverge: right totals, right links, and clicking one
does nothing at all.

Pass `['pageName' => $this->getTablePaginationPageName()]` in the paginator's options.
Prefer overriding `getTablePaginationPageName()` to `->queryStringIdentifier()`, because
the identifier resolves through `getTable()`, which `mount()` cannot call.

**A table inside a modal needs its own page name regardless** — otherwise it reads and
writes the host page's `page` parameter, so opening a modal from page 2 of the page behind
it shows page 2 of the modal's table, which for a short list is empty.

### Asserting pagination by setting the page property cannot see any of that

Drive it with `call('gotoPage', $n, $nameReadFromTheRenderedButton)` instead.

### A reorder table is verified through its handler, not through a synthetic drag

`->reorderable()` reorders through Alpine's SortableJS, which needs a real pointer
sequence and ignores a two-point synthetic drag. Nothing raises, so a browser drive reads
as a table that refuses to reorder. Call what the row's `x-on:end.stop` calls —
`Alpine.$data(tbody).$wire.reorderTable(ids, movedId)` — and assert on the rows. That
covers the renumbering, the scope of the write, and any deferred unique constraint the
swap passes through; only the drag gesture itself is left to the library.

### A relation manager is lazy, and an unbooted one is indistinguishable from a missing one

`Filament\Support\Concerns\CanBeLazy::$isLazy` is true, so a relation manager renders as
`<div class="fi-section fi-loading-section" x-intersect="$wire.__lazyLoad(…)">` whose only
text is a screen-reader "Loading…", and boots when Alpine's `x-intersect` fires.

If it never fires the panel is simply absent: no heading in the page text, no rows, **no
`livewire/update` request**, no console error, nothing in the log, and no cache clear
changes it. `assertSeeLivewire()` still passes — the component is registered in the tree,
just never booted — and a test that mounts the manager directly passes too.

Declare `protected static bool $isLazy = false;` on any relation manager that *is* the
point of its page, and assert the page renders its heading and a row and carries no
`fi-loading-section`.

---

## Forms and fields

### Making a field creation-only: `visible()` is the guard, `dehydrated()` is not

A field hidden with `visible()` is absent from the payload. A field left visible but
`dehydrated(false)` still renders and still accepts input; it is simply dropped on save,
which looks to the operator like the application ignoring them.

### A dehydrated-when-hidden field is also a validated field

`dehydrated()` and validation travel together: a hidden field that is still dehydrated is
still validated, so a `required()` rule on it fails a form the operator cannot see or fix.

### Narrowing a Select's options is also a guard — but you supply the sentence

Filament resolves a Select's options to `Rule::in([...])`, so an option that is not offered
is refused server-side even if it is posted past the control. That makes option-narrowing a
real guard. It is not a *message*: the default refusal is Laravel's generic one, so supply
the sentence that says why.

A `multiple()` Select carries **no** implied `in` rule. Scoping its options alone does not
refuse a posted value; add an explicit rule.

### Wait for `Upload complete` before submitting a form carrying a `FileUpload`

Attaching a file starts an upload the submit button knows nothing about. Submit before it
finishes and validation refuses the form with "The … field is required" over a file the
modal is visibly showing, filename and byte count and all — a sentence that reads as a bug
in the field you just filled. Poll the rendered text for `Upload complete` before
submitting. This bites hardest when driving a browser.

---

## Wizards, actions and modals

### A Wizard step hook must read `$data`, never `getState()`

Inside a step's lifecycle hook, `getState()` returns state that has not yet been merged.
Read the `$data` argument.

### Every wizard step renders at once

All steps are in the DOM from the first render; the wizard shows one. So an assertion that
some text "is not visible" because its step has not been reached will fail, and a closure
in a later step evaluates earlier than you expect.

### A mounted action's modal is not in a Livewire test's `html()`

Filament renders action modals into a `wire:partial="action-modals"` block, and Livewire 4
does not carry a partial's content in the payload `Testable::html()` reads. After
`mountAction()` the modal's div is present and empty, `assertSee` finds nothing inside it,
and `assertActionMounted()` passes all the same.

Render the mounted schema instead — `getMountedActionSchema()` is protected, so reach it
with `Closure::bind(fn () => $page->getMountedActionSchema(), null, $page::class)` and call
`toHtml()`. Advance a wizard step the way the browser does:
`call('callSchemaComponentMethod', $wizard->getKey(), 'nextStep', ['currentStepIndex' => 0])`.

---

## Schemas and embedded Livewire

### `Livewire::make()` components receive ids, not models

A component embedded in a schema is serialised between requests, so pass identifiers and
re-resolve inside the component. Re-resolving is also where ownership is enforced: do not
trust a record handed down from the parent.

Mark any property the component must not accept from the browser `#[Locked]`. A component
that takes a filesystem path and leaves it unlocked lets a viewer point it at any file the
process can read.

### Layout components and entries live in different namespaces

Layout components are in `Filament\Schemas\Components`; infolist entries are in
`Filament\Infolists\Components`. Importing one from the other's namespace is the most
common Filament 5 upgrade error, and the failure is a class-not-found at render time.
