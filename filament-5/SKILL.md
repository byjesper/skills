---
name: filament-5
description: >
  Comprehensive skill for Filament 5, a Server-Driven UI (SDUI) framework for Laravel.
  Use this skill when building admin panels, CRUD interfaces, dashboards, form-based apps,
  or any Filament 5 project. Covers resources, tables, forms, schemas, infolists, actions,
  notifications, widgets, relationships, panels, theming, plugins, and multi-tenancy.
  Triggers on: "Filament", "Filament 5", "filament resource", "filament table",
  "filament form", "admin panel Laravel", "filament plugin", "filament widget",
  "filament action", "filament notification", "filament relationship",
  "filament panel", "filament multi-tenancy", or any Laravel admin panel task.
---

# Filament 5

Filament 5 is a Server-Driven UI (SDUI) framework for Laravel. UIs are defined entirely in PHP using configuration objects, rendered server-side into HTML via Livewire, Alpine.js, and Tailwind CSS.

## Read first

Load the reference file for the area you are working in — each one is a complete,
code-heavy guide. Do not guess at an API surface that a reference documents.

| Working on | Load |
|------------|------|
| Resource classes, eager loading, pages, policies | `references/resources.md` |
| Form fields, validation, layout, utility injection | `references/forms-fields.md` |
| Table columns, filters, pagination, table actions | `references/tables-columns.md` |
| Actions, modals, notifications | `references/actions-modals.md` |
| Panel config, navigation, theming, multi-tenancy | `references/panels-theming.md` |
| Read-only record display | `references/infolists.md` |
| Dashboard stats and charts | `references/widgets.md` |
| Relation managers, Select/Repeater relationships | `references/relationships.md` |
| Create/edit hooks, mutating form data | `references/lifecycle-hooks.md` |

## Architecture

```
Panel (e.g., /admin)
  ├── Resources (CRUD for Eloquent models)
  │     ├── Pages: List, Create, Edit, View
  │     ├── Forms (schemas with fields)
  │     ├── Tables (columns, filters, actions)
  │     └── RelationManagers
  ├── Pages (custom non-CRUD pages)
  └── Widgets (dashboard: stats, charts, tables)
```

### Core packages

| Package | Purpose |
|---------|---------|
| `filament/filament` | Panel builder (requires all others) |
| `filament/schemas` | Core UI component system |
| `filament/forms` | Form field components |
| `filament/tables` | Data table builder |
| `filament/infolists` | Read-only description lists |
| `filament/actions` | Buttons + modals + logic |
| `filament/notifications` | Flash / database / broadcast notifications |
| `filament/widgets` | Dashboard widgets |

### Schema component types

All UI is built from `Schema` objects with `components()`. Four component types:

1. **Form Fields** (`Filament\Forms\Components`) — writable, validated inputs
2. **Infolist Entries** (`Filament\Infolists\Components`) — read-only display
3. **Layout Components** (`Filament\Schemas\Components`) — structure (Grid, Section, Tabs, Wizard)
4. **Prime Components** (`Filament\Schemas\Components`) — static content (Text, Icon)

## File structure conventions

```
app/Filament/Resources/
+-- Customers/
|   +-- CustomerResource.php      # Main resource class
|   +-- Pages/
|   |   +-- CreateCustomer.php
|   |   +-- EditCustomer.php
|   |   +-- ListCustomers.php
|   +-- Schemas/
|   |   +-- CustomerForm.php      # Form schema (optional, can be inline)
|   +-- Tables/
|   |   +-- CustomersTable.php    # Table config (optional, can be inline)
```

## Resource workflow

1. Generate with `make:filament-resource`, including `--view` unless there is a
   specific reason not to — the View page is required for global search result links.
2. Define `form()`, `table()`, and `infolist()`.
3. Override `getEloquentQuery()` to eager load every relationship the table or
   infolist touches. Skipping this is the most common source of N+1 queries.
4. Register all pages in `getPages()`, including `view`.
5. Write the model policy — Filament reads it automatically and hides resources
   whose `viewAny()` returns false.

See `references/resources.md` for the class skeleton, a complete worked example,
and the matching policy.

## Quick reference

### Common field types

| Field | Key Methods |
|-------|-------------|
| `TextInput` | `->email()`, `->numeric()`, `->password()->revealable()`, `->url()`, `->tel()`, `->mask()`, `->prefix()`, `->suffix()`, `->copyable()` |
| `Select` | `->options()`, `->searchable()`, `->multiple()`, `->relationship()`, `->preload()`, `->createOptionForm()` |
| `Checkbox` | `->accepted()` |
| `Toggle` | `->onIcon()`, `->offIcon()`, `->onColor()` |
| `DateTimePicker` | `->native(false)`, `->displayFormat()`, `->timezone()`, `->minDate()`, `->maxDate()` |
| `FileUpload` | `->multiple()`, `->image()`, `->imageEditor()`, `->avatar()`, `->maxSize()`, `->acceptedFileTypes()` |
| `RichEditor` | TipTap-based, JSON/HTML storage, toolbar customization |
| `Repeater` | `->relationship()`, `->collapsible()`, `->itemLabel()`, `->minItems()`, `->maxItems()` |
| `Builder` | Block-based content with multiple block types |
| `Textarea` | `->autosize()`, `->rows()` |
| `TagsInput` | `->separator()`, `->suggestions()` |
| `KeyValue` | Dynamic key-value pairs |
| `ColorPicker` | `->hex()`, `->rgb()`, `->hsl()` |
| `Hidden` | Invisible form field |

### Column types

| Column | Key Methods |
|--------|-------------|
| `TextColumn` | `->searchable()`, `->sortable()`, `->date()`, `->dateTime()`, `->money()`, `->numeric()`, `->badge()`, `->color()`, `->icon()`, `->description()`, `->copyable()`, `->limit(50)` |
| `IconColumn` | `->boolean()`, `->trueIcon()`, `->falseIcon()`, `->trueColor()` |
| `ImageColumn` | `->size(40)`, `->circular()`, `->stacked()`, `->overlap(2)` |
| `BadgeColumn` | `->color()`, `->icon()`, `->iconPosition()` |
| `ColorColumn` | `->copyable()` |
| `ToggleColumn` | Inline editable boolean toggle |

Access related data via dot notation: `TextColumn::make('author.name')`.

### Built-in action types

| Action | Purpose |
|--------|---------|
| `CreateAction` | Create new records |
| `EditAction` | Edit existing records |
| `ViewAction` | View record details |
| `DeleteAction` | Delete with confirmation |
| `ReplicateAction` | Duplicate records |
| `ForceDeleteAction` | Permanently delete soft-deleted |
| `RestoreAction` | Restore soft-deleted records |
| `ImportAction` | Bulk import |
| `ExportAction` | Bulk export |

### Component utility injection

Configuration methods accept closures with injected utilities:

| Utility | Description |
|---------|-------------|
| `$get` | Get value of another component (`$get('field_name')`) |
| `$set` | Set value of another component (`$set('field_name', 'value')`) |
| `$record` | Current Eloquent model |
| `$operation` | `'create'`, `'edit'`, or `'view'` |
| `$livewire` | Livewire component instance |
| `$state` / `$rawState` | Current field value (casted / uncasted) |
| `$schema` | Current schema instance |

## Common gotchas

1. **Authorization**: Resource not showing? Check the `viewAny()` policy method returns `true`.
2. **Navigation parents**: Both child AND parent must have matching `$navigationGroup`.
3. **Soft deletes**: Remove `SoftDeletingScope` in `getRecordRouteBindingEloquentQuery()` AND use `getEloquentQuery()`.
4. **Use `Operation` enum**: Use `Operation::Create` / `Operation::Edit` (not strings) for `hiddenOn` / `visibleOn`.
5. **All model attributes exposed to JS by default** - use `$hidden` on models for sensitive fields.
6. **File upload security**: Never use `preserveFilenames()` with local/public disks (RCE vulnerability). Use `storeFileNamesIn()` + random filesystem names + `preventFilePathTampering()`.
7. **Simple resources** cannot have relation managers (no Edit page to attach them to).
8. **Double backslashes** in `--model-namespace` Artisan flag: `--model-namespace=Custom\\Path\\Models`.
9. **Bulk actions** use `*Any()` policy methods (`deleteAny`, `forceDeleteAny`, `restoreAny`).
10. **Global scopes** are respected by default in queries.
11. **Widget data**: always use database queries, not hardcoded arrays. See `references/widgets.md`.

## Code generation commands

```bash
# Resources
php artisan make:filament-resource Customer
php artisan make:filament-resource Customer --view --simple --soft-deletes

# Pages
php artisan make:filament-page Settings
php artisan make:filament-page ViewOrder --type=custom --resource=OrderResource

# Widgets
php artisan make:filament-widget BlogPostsChart --chart
php artisan make:filament-widget LatestOrders --table
php artisan make:filament-widget StatsOverview

# Relation Managers
php artisan make:filament-relation-manager CategoryResource posts title
php artisan make:filament-relation-manager CategoryResource posts title --soft-deletes

# Theme
php artisan make:filament-theme
```
