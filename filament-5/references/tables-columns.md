# Filament 5 - Tables & Columns Quick Reference

## Table of Contents

1. [Column Types](#1-column-types)
   - [TextColumn](#textcolumn)
   - [Badge Display (TextColumn + badge())](#badge-display-textcolumn--badge)
   - [IconColumn](#iconcolumn)
   - [ImageColumn](#imagecolumn)
   - [ColorColumn](#colorcolumn)
   - [ToggleColumn](#togglecolumn)
   - [TextInputColumn](#textinputcolumn)
   - [CheckboxColumn](#checkboxcolumn)
   - [SelectColumn](#selectcolumn)
2. [Column Features](#2-column-features)
3. [Filters](#3-filters)
4. [Actions](#4-actions)
5. [Table Configuration](#5-table-configuration)
6. [Layout Options](#6-layout-options)

---

## 1. Column Types

All columns live in `Filament\Tables\Columns` namespace. Created with static `make('name')`.

```php
use Filament\Tables\Table;

public function table(Table $table): Table
{
    return $table->columns([
        // columns here
    ]);
}
```

### TextColumn

**Class:** `Filament\Tables\Columns\TextColumn`

```php
use Filament\Tables\Columns\TextColumn;
use Filament\Support\Enums\FontWeight;
use Filament\Support\Enums\FontFamily;

TextColumn::make('title')
    // -- Formatting --
    ->formatStateUsing(fn (string $state): string => "#{$state}")
    ->prefix('Mr. ')
    ->suffix(' USD')

    // -- Display as badge --
    ->badge()
    ->color(fn (string $state): string => match ($state) {
        'draft' => 'gray',
        'reviewing' => 'warning',
        'published' => 'success',
        'rejected' => 'danger',
        default => 'primary',
    })

    // -- Icons --
    ->icon(Heroicon::Envelope)
    ->iconColor('primary')
    ->iconPosition('before')           // 'before' | 'after'

    // -- Colors (text color when not in badge mode) --
    ->color('primary')                 // primary, secondary, success, danger, warning, gray

    // -- Typography --
    ->size(TextColumnSize::Large)      // ExtraSmall, Small, Medium, Large, ExtraLarge
    ->weight(FontWeight::Bold)         // Thin, ExtraLight, Light, Normal, Medium, SemiBold, Bold, ExtraBold, Black
    ->fontFamily(FontFamily::Mono)     // Sans, Serif, Mono

    // -- Text handling --
    ->limit(50)                        // Character limit
    ->words(10)                        // Word limit
    ->wrap()                           // Allow wrapping
    ->lineClamp(2)                     // Limit to N lines
    ->description(fn ($record) => $record->slug)  // Subtitle below

    // -- Dates --
    ->date()                           // Format as date
    ->date('M d, Y')                   // Custom format
    ->dateTime()                       // Format as datetime
    ->dateTime('M d, Y H:i')           // Custom format
    ->time()                           // Format as time
    ->since()                          // Relative ("2 hours ago")
    ->dateTooltip()                    // Show formatted date on hover
    ->timezone('America/New_York')

    // -- Numbers --
    ->numeric()                        // Format as number
    ->numeric(decimalPlaces: 2)
    ->money('USD')                     // Format as currency
    ->money('EUR', locale: 'de')

    // -- Content rendering --
    ->markdown()                       // Render markdown
    ->html()                           // Render HTML (sanitized)
    ->html(false)                      // Raw HTML (no sanitization)

    // -- Lists (for array values) --
    ->list()
    ->listWithBullets()
    ->limitList(3)
    ->expandableLimitedList()
    ->separator(',')                   // Split string into list

    // -- Copy to clipboard --
    ->copyable()
    ->copyMessage('Copied!')
    ->copyMessageDuration(2000)
    ->copyableState(fn ($state) => strtoupper($state))

    // -- Row index --
    ->rowIndex()                       // Show 1, 2, 3...
    ->rowIndex(
        isFromZero: true,              // 0, 1, 2...
        isFromEnd: true,               // ...3, 2, 1
    )
```

### Badge Display (TextColumn + badge())

**Note:** There is no separate `BadgeColumn` class. Use `TextColumn` with `->badge()`.

```php
use Filament\Tables\Columns\TextColumn;

TextColumn::make('status')
    ->badge()
    ->color(fn (string $state): string => match ($state) {
        'draft' => 'gray',
        'reviewing' => 'warning',
        'published' => 'success',
        'rejected' => 'danger',
    })
    ->icon(fn (string $state) => match ($state) {
        'published' => Heroicon::CheckCircle,
        'rejected' => Heroicon::XCircle,
        default => null,
    })
    ->badge(FeatureFlag::active());    // Conditional badge
```

### IconColumn

**Class:** `Filament\Tables\Columns\IconColumn`

```php
use Filament\Tables\Columns\IconColumn;
use Filament\Support\Icons\Heroicon;

// -- Boolean (check/x) --
IconColumn::make('is_paid')
    ->boolean()                        // Auto check/x icons
    ->trueIcon(Heroicon::Check)
    ->falseIcon(Heroicon::XMark)
    ->trueColor('success')
    ->falseColor('danger');

// -- Custom icon map --
IconColumn::make('status')
    ->icon(fn (string $state): string => match ($state) {
        'draft' => Heroicon::Pencil,
        'reviewing' => Heroicon::Eye,
        'published' => Heroicon::CheckCircle,
        default => Heroicon::QuestionMarkCircle,
    })
    ->color(fn (string $state): string => match ($state) {
        'draft' => 'gray',
        'reviewing' => 'warning',
        'published' => 'success',
        default => 'gray',
    });
```

### ImageColumn

**Class:** `Filament\Tables\Columns\ImageColumn`

```php
use Filament\Tables\Columns\ImageColumn;

ImageColumn::make('avatar')
    ->circular()                       // Circular avatar
    ->square()                         // Square corners
    ->rounded()                        // Rounded corners
    ->ring(2)                          // Ring border width
    ->height(40)
    ->width(40)
    ->size(40)                         // Both height & width
    ->defaultImageUrl(fn ($record) => 'https://placehold.co/40')
    ->checkFileExistence()             // Check file exists before render
    ->visibility('private')            // For S3 private files
    ->extraImgAttributes(['loading' => 'lazy']);
```

### ColorColumn

**Class:** `Filament\Tables\Columns\ColorColumn`

```php
use Filament\Tables\Columns\ColorColumn;

// Displays a color swatch from a hex/rgb value
ColorColumn::make('color');

ColorColumn::make('color')
    ->copyable();                      // Click to copy hex value
```

### ToggleColumn

**Class:** `Filament\Tables\Columns\ToggleColumn`

Editable toggle that updates the database inline.

```php
use Filament\Tables\Columns\ToggleColumn;

ToggleColumn::make('is_featured')
    ->disabled(fn ($record) => $record->user_id !== auth()->id())
    ->beforeStateUpdated(function ($record, $state) {
        // Runs before DB save
    })
    ->afterStateUpdated(function ($record, $state) {
        // Runs after DB save
    });
```

> **Security:** ToggleColumn does NOT auto-check Laravel policies. Use `disabled()` for authorization.

### TextInputColumn

**Class:** `Filament\Tables\Columns\TextInputColumn`

Editable text input that updates inline.

```php
use Filament\Tables\Columns\TextInputColumn;

TextInputColumn::make('title')
    ->rules(['required', 'max:255'])
    ->type('text')                     // HTML input type
    ->type('color')                    // Color picker
    ->inputMode('decimal')             // numeric | decimal | tel | email | url
    ->step(0.01)                       // For numeric inputs
    ->disabled(fn ($record) => $record->isLocked())
    // Affixes
    ->prefix('$')
    ->suffix('USD')
    ->prefixIcon(Heroicon::CurrencyDollar)
    ->suffixIcon(Heroicon::Check)
    ->prefixIconColor('success')
    // Lifecycle
    ->beforeStateUpdated(function ($record, $state) {
        // Before save
    })
    ->afterStateUpdated(function ($record, $state) {
        // After save
    });
```

### CheckboxColumn

**Class:** `Filament\Tables\Columns\CheckboxColumn`

Editable checkbox that updates inline.

```php
use Filament\Tables\Columns\CheckboxColumn;

CheckboxColumn::make('is_featured')
    ->disabled(fn ($record) => !auth()->user()->can('update', $record))
    ->beforeStateUpdated(function ($record, $state) {
        // Before save
    })
    ->afterStateUpdated(function ($record, $state) {
        // After save
    });
```

### SelectColumn

**Class:** `Filament\Tables\Columns\SelectColumn`

Editable select dropdown that updates inline.

```php
use Filament\Tables\Columns\SelectColumn;

SelectColumn::make('status')
    ->options([
        'draft' => 'Draft',
        'reviewing' => 'Reviewing',
        'published' => 'Published',
    ])
    ->native(false)                    // JavaScript select (not native HTML)
    ->searchable()                     // Searchable options
    ->searchableOptions()              // Enable search with custom query
    // Relationship
    ->optionsRelationship(name: 'author', titleAttribute: 'name')
    // Custom search results
    ->getOptionsSearchResultsUsing(fn (string $search): array =>
        User::query()
            ->where('name', 'like', "%{$search}%")
            ->limit(50)
            ->pluck('name', 'id')
            ->all()
    )
    ->getOptionLabelUsing(fn ($value): ?string => User::find($value)?->name)
    // Custom messages
    ->optionsLoadingMessage('Loading...')
    ->noOptionsSearchResultsMessage('No results found.')
    ->searchPrompt('Start typing to search...')
    ->searchingMessage('Searching...')
    ->searchDebounce('500ms')
    // Validation
    ->rules(['required', 'in:draft,published'])
    ->validationAttribute('status')
    // Disabling
    ->disableOptionWhen(fn (string $value): bool => $value === 'published')
    // Lifecycle
    ->beforeStateUpdated(function ($record, $state) {
        // Before save
    })
    ->afterStateUpdated(function ($record, $state) {
        // After save
    });
```

---

## 2. Column Features

### Searching

```php
use Filament\Tables\Columns\TextColumn;

// Column-level
TextColumn::make('name')->searchable();
TextColumn::make('name')->searchable(isIndividual: true);  // Own search field

// Table-level
$table->searchable(['id', 'author.id']);   // Extra searchable columns
$table->searchable();                       // Enable search without column-level

// Table search customization
$table->searchPlaceholder('Search (ID, Name)');
$table->searchDebounce('500ms');
$table->searchOnBlur();                     // Search when focus leaves
$table->persistSearchInSession();
$table->splitSearchTerms(false);            // Don't split by space
```

### Sorting

```php
// Column-level
TextColumn::make('name')->sortable();
TextColumn::make('full_name')
    ->sortable(query: fn ($query, $direction) =>
        $query->orderBy('last_name', $direction)
            ->orderBy('first_name', $direction)
    );

// Table-level
$table->defaultSort('stock', direction: 'desc');
$table->defaultSort(fn (Builder $query) => $query->orderBy('stock'));
$table->defaultSortOptionLabel('Date');
$table->persistSortInSession();
$table->disableDefaultSorting();
```

### Relationships via Dot Notation

```php
use Filament\Tables\Columns\TextColumn;

TextColumn::make('author.name');            // Relationship attribute
TextColumn::make('meta.title');             // JSON/array key
TextColumn::make('users_count')->counts('users');
TextColumn::make('users_exists')->exists('users');
TextColumn::make('users_avg_age')->avg('users', 'age');
TextColumn::make('users_max_age')->max('users', 'age');
TextColumn::make('users_min_age')->min('users', 'age');
TextColumn::make('users_total_age')->sum('users', 'age');

// Scoped aggregates
TextColumn::make('active_users_count')
    ->counts(['users' => fn ($q) => $q->where('is_active', true)]);

// Scoped existence
TextColumn::make('active_users_exists')
    ->exists(['users' => fn (Builder $query) => $query->where('is_active', true)]);
```

### Toggleable Columns

```php
use Filament\Tables\Columns\TextColumn;

TextColumn::make('id')->toggleable();
TextColumn::make('id')->toggleable(isToggledHiddenByDefault: true);

// Table-level column management
$table->reorderableColumns();               // Let users reorder columns
$table->deferColumnManager(false);          // Live changes (no apply button)
$table->columnManagerLayout(ColumnManagerLayout::Modal);
$table->columnManagerColumns(2);
```

### Column Groups

```php
use Filament\Tables\Columns\ColumnGroup;
use Filament\Tables\Columns\TextColumn;
use Filament\Tables\Columns\IconColumn;
use Filament\Support\Enums\Alignment;

$table->columns([
    TextColumn::make('title'),
    ColumnGroup::make('Visibility', [
        TextColumn::make('status'),
        IconColumn::make('is_featured'),
    ])
    ->alignCenter()
    ->wrapHeader(),
]);
```

### Summaries / Aggregations

```php
use Filament\Tables\Columns\Summarizers\Average;
use Filament\Tables\Columns\Summarizers\Count;
use Filament\Tables\Columns\Summarizers\Range;
use Filament\Tables\Columns\Summarizers\Sum;
use Filament\Tables\Columns\Summarizers\Summarizer;
use Filament\Tables\Columns\TextColumn;
use Filament\Tables\Columns\IconColumn;

TextColumn::make('price')
    ->summarize(Sum::make()->label('Total')->money('USD'));

TextColumn::make('rating')
    ->numeric()
    ->summarize([
        Average::make(),
        Range::make(),
    ]);

// Count with icons (for boolean columns)
IconColumn::make('is_published')
    ->boolean()
    ->summarize(Count::make()->icons());

// Custom summarizer
TextColumn::make('name')
    ->summarize(Summarizer::make()
        ->using(fn (Builder $query): string => $query->count('is_featured'))
    );

// Conditional summary
TextColumn::make('price')
    ->summarize(Sum::make()->hidden(fn (): bool => !auth()->user()->isAdmin()));

// Range with date formatting
Range::make()->minimalDateTimeDifference();
Range::make()->excludeNull(false);
```

> **Gotcha:** First column cannot use summarizers (reserved for summary heading).

---

## 3. Filters

### Filter Types

```php
use Filament\Tables\Filters\Filter;
use Filament\Tables\Filters\SelectFilter;
use Filament\Tables\Filters\TernaryFilter;
use Filament\Tables\Filters\TrashedFilter;
use Filament\Tables\Filters\QueryBuilder;
use Filament\Tables\Enums\FiltersLayout;
use Illuminate\Database\Eloquent\Builder;

// -- Basic checkbox filter --
Filter::make('is_featured')
    ->query(fn (Builder $query): Builder => $query->where('is_featured', true))
    ->label('Featured')
    ->toggle()                                     // Use toggle instead of checkbox
    ->default()                                    // Apply by default
    ->indicator('Featured');                       // Active filter indicator

// -- Select filter --
SelectFilter::make('status')
    ->options([
        'draft' => 'Draft',
        'reviewing' => 'Reviewing',
        'published' => 'Published',
    ])
    ->native(false)
    ->searchable()
    ->multiple()
    ->default('draft');

// Select filter by relationship
SelectFilter::make('author')
    ->relationship('author', 'name')
    ->searchable()
    ->preload()
    ->multiple();

// -- Ternary filter (3-state) --
TernaryFilter::make('is_featured')
    ->trueLabel('Featured')
    ->falseLabel('Not featured')
    ->placeholder('All posts')
    ->native(false);

// -- Trashed filter (pre-built) --
TrashedFilter::make();

// -- Query builder (complex filters) --
QueryBuilder::make()
    ->constraints([
        // Define constraints
    ]);

// -- Custom filter schema (any form components) --
Filter::make('published_between')
    ->form([
        Forms\Components\DatePicker::make('published_from'),
        Forms\Components\DatePicker::make('published_until'),
    ])
    ->query(function (Builder $query, array $data): Builder {
        return $query
            ->when($data['published_from'],
                fn ($q, $date) => $q->whereDate('published_at', '>=', $date))
            ->when($data['published_until'],
                fn ($q, $date) => $q->whereDate('published_at', '<=', $date));
    });
```

### Filter Configuration

```php
Filter::make('is_featured')
    ->modifyFormFieldUsing(fn (Checkbox $field) => $field->inline(false))
    ->baseQuery(fn ($query) => $query->withoutGlobalScopes([SoftDeletingScope::class]))
    ->excludeWhenResolvingRecord();

// Table-level filter settings
$table->filters([/* ... */])
    ->persistFiltersInSession()
    ->deferFilters(false)                          // Live filters (no apply button)
    ->deselectAllRecordsWhenFiltered(false)        // Keep selections on filter change
    ->filtersTriggerAction(fn (Action $action) => $action->iconButton())
    ->filtersRemoveAllAction(fn (Action $action) => $action->tooltip('Clear filters'))
    ->filtersApplyAction(fn (Action $action) => $action->label('Apply'));
```

### Filter Layout

```php
use Filament\Tables\Enums\FiltersLayout;

$table->filtersLayout(FiltersLayout::Modal);
$table->filtersLayout(FiltersLayout::Dropdown);
$table->filtersLayout(FiltersLayout::AboveContent);
$table->filtersLayout(FiltersLayout::AboveContentCollapsible);
$table->filtersLayout(FiltersLayout::BelowContent);

// Global default
use Filament\Tables\Table;

Table::configureUsing(function (Table $table): void {
    $table->filtersLayout(FiltersLayout::AboveContentCollapsible);
});
```

---

## 4. Actions

### Record Actions (Row Actions)

```php
use Filament\Tables\Table;
use Filament\Actions\Action;
use Filament\Actions\ActionGroup;
use Filament\Actions\EditAction;
use Filament\Actions\DeleteAction;
use Filament\Actions\ViewAction;
use Filament\Tables\Enums\RecordActionsPosition;

$table->recordActions([
    // Simple action
    Action::make('edit')
        ->url(fn (Post $record): string => route('posts.edit', $record))
        ->openUrlInNewTab(),

    // Action with confirmation modal
    Action::make('delete')
        ->requiresConfirmation()
        ->color('danger')
        ->icon(Heroicon::Trash)
        ->action(fn (Post $record) => $record->delete()),

    // Action that accesses selected records
    Action::make('copyToSelected')
        ->accessSelectedRecords()
        ->action(function (Model $record, Collection $selectedRecords) {
            $selectedRecords->each(
                fn (Model $selected) => $selected->update(['is_active' => $record->is_active])
            );
        }),

    // Grouped actions
    ActionGroup::make([
        ViewAction::make(),
        EditAction::make(),
        DeleteAction::make(),
    ])->label('Actions'),
]);

// Position (default: after columns)
$table->recordActions([/* ... */], position: RecordActionsPosition::BeforeCells);
```

### Bulk Actions

```php
use Filament\Actions\BulkAction;
use Filament\Actions\BulkActionGroup;
use Filament\Actions\DeleteBulkAction;
use Illuminate\Database\Eloquent\Collection;

$table->toolbarActions([
    BulkActionGroup::make([
        DeleteBulkAction::make(),

        BulkAction::make('publish')
            ->requiresConfirmation()
            ->icon('heroicon-m-arrow-up-tray')
            ->action(fn (Collection $records) => $records->each->update(['status' => 'published']))
            ->deselectRecordsAfterCompletion()
            ->successNotificationTitle('Posts published')
            ->failureNotificationTitle(fn (int $successCount, int $totalCount): string =>
                "{$successCount} of {$totalCount} posts published"
            )
            ->authorizeIndividualRecords('update'),   // Check policy per record
    ]),
]);
```

### Header Actions

```php
use Filament\Actions\Action;

$table->headerActions([
    Action::make('create')
        ->label('Create Post')
        ->url(route('posts.create'))
        ->icon('heroicon-m-plus'),
]);
```

### Toolbar Actions

```php
$table->toolbarActions([
    BulkActionGroup::make([/* bulk actions */]),
    Action::make('create')->url(route('posts.create')),
]);
```

### Record Selection

```php
// Check if record is selectable
$table->checkIfRecordIsSelectableUsing(fn (Model $record): bool => $record->status === Status::Enabled);

// Max selectable records
$table->maxSelectableRecords(4);

// Select current page only (prevent "select all pages")
$table->selectCurrentPageOnly();
```

### Global Settings

```php
use Filament\Tables\Table;

Table::configureUsing(function (Table $table): void {
    $table->modifyUngroupedRecordActionsUsing(
        fn (Action $action) => $action->iconButton()
    );
});
```

---

## 5. Table Configuration

### Pagination

```php
use Filament\Tables\Enums\PaginationMode;

// Custom page options
$table->paginated([10, 25, 50, 100, 'all']);
$table->defaultPaginationPageOption(25);
$table->extremePaginationLinks();               // Show first/last page links

// Pagination modes
$table->paginationMode(PaginationMode::Simple);  // Previous/Next only
$table->paginationMode(PaginationMode::Cursor);  // Cursor pagination

// Multiple tables on same page
$table->queryStringIdentifier('users');

// Disable pagination
$table->paginated(false);
```

### Reordering

```php
// Basic
$table->reorderable('sort');

// With spatie/eloquent-sortable
$table->reorderable('order_column');

// Conditional
$table->reorderable('sort', auth()->user()->isAdmin());

// Descending direction
$table->reorderable('sort', direction: 'desc');

// Keep pagination while reordering (disabled by default)
$table->paginatedWhileReordering();

// Custom trigger action
$table->reorderRecordsTriggerAction(
    fn (Action $action, bool $isReordering) =>
        $action->button()
            ->label($isReordering ? 'Done' : 'Reorder'),
);

// Hooks
$table->reorderable('sort')
    ->beforeReordering(function (array $order): void { /* ... */ })
    ->afterReordering(function (array $order): void { /* ... */ });
```

### Record URLs (Clickable Rows)

```php
use Illuminate\Database\Eloquent\Model;

$table->recordUrl(
    fn (Model $record): string => route('posts.edit', ['record' => $record]),
);
$table->openRecordUrlInNewTab();
```

### Polling

```php
$table->poll('10s');                    // Poll every 10 seconds
$table->deferLoading();                 // Async loading (shows spinner)
```

### Empty State

```php
$table
    ->emptyStateHeading('No posts yet')
    ->emptyStateDescription('Once you write your first post, it will appear here.')
    ->emptyStateIcon(Heroicon::DocumentText)
    ->emptyStateActions([
        Action::make('create')
            ->label('Create post')
            ->url(route('posts.create'))
            ->icon('heroicon-m-plus'),
    ])
    ->emptyState(view('tables.empty-state'));  // Custom view
```

### Custom Data Sources

```php
// Array data (non-Eloquent)
$table->records(fn (): array => [
    1 => ['title' => 'First', 'slug' => 'first'],
    2 => ['title' => 'Second', 'slug' => 'second'],
]);

// External API
$table->records(fn (): array => Http::get('https://api.example.com/posts')->json());
```

### Row Styling

```php
// Striped rows
$table->striped();

// Custom row classes
use App\Models\Post;

$table->recordClasses(
    fn (Post $record) => match ($record->status) {
        'draft' => 'draft-post-table-row',
        'reviewing' => 'reviewing-post-table-row',
        'published' => 'published-post-table-row',
        default => null,
    }
);
```

### Table Header

```php
$table->heading('Clients');
$table->description('Manage your clients here.');
$table->header(view('tables.header', ['heading' => 'Clients']));
```

### Grouping

```php
use Filament\Tables\Grouping\Group;
use Illuminate\Database\Eloquent\Builder;

// Simple groups
$table->defaultGroup('status');
$table->groups(['status', 'author.name', 'created_at']);

// Advanced group configuration
$table->groups([
    Group::make('status')
        ->label('Status')
        ->collapsible()
        ->date()                                // Group by date only
        ->titlePrefixedWithLabel(false)
        ->getTitleFromRecordUsing(fn (Post $record): string =>
            ucfirst($record->status->getLabel())
        )
        ->getDescriptionFromRecordUsing(fn (Post $record): string =>
            "By {$record->author->name}"
        )
        ->getKeyFromRecordUsing(fn (Post $record): string => $record->status->value)
        ->orderQueryUsing(fn (Builder $query, string $direction) =>
            $query->orderBy('status', $direction)
        )
        ->scopeQueryByKeyUsing(fn (Builder $query, string $key) =>
            $query->where('status', $key)
        ),
]);

// Table-level group settings
$table
    ->collapsedGroupsByDefault()
    ->defaultGroup('status')
    ->groupsOnly()                             // Show only summaries
    ->groupsInDropdownOnDesktop()
    ->hiddenGroupingSettings()                 // Hide all group settings
    ->hiddenGroupingDirectionSetting();        // Hide direction toggle only
```

### Searching with Laravel Scout

```php
$table->searchUsing(
    fn (Builder $query, string $search) =>
        $query->whereKey(Post::search($search)->keys())
);
$table->searchable();
```

---

## 6. Layout Options

### stackedOnMobile

```php
// All columns stack vertically on mobile
$table->stackedOnMobile();
```

### Split Layout

Horizontal arrangement that stacks below a breakpoint.

```php
use Filament\Tables\Columns\Layout\Split;

Split::make([
    ImageColumn::make('avatar')->circular()->size(40),
    TextColumn::make('name')->weight(FontWeight::Bold)->searchable()->sortable(),
    TextColumn::make('email'),
])
->from('md')                                   // Horizontal from md breakpoint, stack below
```

### Stack Layout

Vertical stacking of columns.

```php
use Filament\Tables\Columns\Layout\Stack;
use Filament\Support\Enums\Alignment;

Stack::make([
    TextColumn::make('phone')->icon('heroicon-m-phone'),
    TextColumn::make('email')->icon('heroicon-m-envelope'),
])
->visibleFrom('md')                            // Hide on mobile
->alignment(Alignment::End)                    // Align content
->space(2);                                    // Spacing between items
```

### Grid Layout

```php
use Filament\Tables\Columns\Layout\Grid;

Grid::make([
    'lg' => 2,                                  // 2 columns at lg
    '2xl' => 4,                                 // 4 columns at 2xl
])
->schema([
    // Columns...
]);

// With column span
Grid::make(['lg' => 2, '2xl' => 5])
    ->schema([
        Stack::make([/* ... */])
            ->columnSpan(['lg' => 'full', '2xl' => 2]),
        TextColumn::make('phone')->columnSpan(['2xl' => 2]),
        TextColumn::make('email'),
    ]);
```

### Panel (Collapsible)

```php
use Filament\Tables\Columns\Layout\Panel;
use Filament\Tables\Columns\Layout\Stack;

Panel::make([
    Stack::make([
        TextColumn::make('phone')->icon('heroicon-m-phone'),
        TextColumn::make('email')->icon('heroicon-m-envelope'),
    ]),
])
->collapsible()
->collapsed(false);                             // Expanded by default
```

### Content Grid

Converts table rows into a grid of cards.

```php
$table->columns([
    Stack::make([
        ImageColumn::make('avatar')->circular()->size(60),
        TextColumn::make('name')->weight(FontWeight::Bold),
        TextColumn::make('email'),
    ]),
])
->contentGrid([
    'md' => 2,                                  // 2 columns at md
    'xl' => 3,                                  // 3 columns at xl
]);
```

### Custom View

```php
use Filament\Tables\Columns\Layout\View;

View::make('tables.components.custom-row'),
```

### Embedding Components

```php
use Filament\Tables\Columns\Layout\Component;

Component::make(MyCustomComponent::make('name')),
```

### Responsive Breakpoints (on columns)

```php
TextColumn::make('slug')->visibleFrom('md');    // Show from md up
TextColumn::make('slug')->hiddenFrom('md');     // Hide from md up
```

---

## Universal Column Methods

These methods work on all column types:

```php
// State
->state('Hello')                              // Static state
->state(fn ($record) => strtoupper($record->name))  // Dynamic
->default('Untitled')                         // Default value
->placeholder('N/A')                          // Placeholder text (light gray)

// Visibility
->hidden()                                    // Hide column
->hidden(fn () => !auth()->user()->isAdmin()) // Conditional
->visible()                                   // Show column
->visibleFrom('md')                           // Responsive
->hiddenFrom('md')

// Column label
->label('Full name')
->label(fn () => __('columns.name'))

// Alignment
->alignStart()                                // Horizontal
->alignCenter()
->alignEnd()
->alignment(Alignment::Center)
->verticallyAlignStart()                      // Vertical
->verticallyAlignCenter()
->verticallyAlignEnd()

// Width
->grow()                                      // Allow column to grow
->width('1%')                                 // Fixed width

// Clickable
->url(fn ($record) => route('edit', $record))
->openUrlInNewTab()
->action(fn ($record) => /* ... */)
->disabledClick()                             // Prevent click

// Tooltips
->tooltip('Tooltip text')
->headerTooltip('Column header tooltip')

// Toggleable
->toggleable()
->toggleable(isToggledHiddenByDefault: true)

// Extra attributes
->extraAttributes(['class' => 'slug-column'], merge: true)
->extraCellAttributes(['class' => 'slug-cell'])
->extraHeaderAttributes(['class' => 'slug-header-cell'])

// Sortable
->sortable()
->sortable(query: fn ($query, $dir) => $query->orderBy('name', $dir))

// Searchable
->searchable()
->searchable(isIndividual: true)
```

---

## Quick Gotchas

| # | Gotcha | Solution |
|---|--------|----------|
| 1 | First column can't use summarizers | Reserve it for a label column |
| 2 | Pagination disabled during reordering | Use `paginatedWhileReordering()` |
| 3 | Filter `query()` runs in scoped `where()` | Use `baseQuery()` for global scopes |
| 4 | Multiple tables on same page conflict | Use `queryStringIdentifier()` |
| 5 | Mass assignment on reorder | Add sort column to `$fillable` |
| 6 | Filters require clicking "Apply" by default | Use `deferFilters(false)` for live |
| 7 | Column manager requires clicking "Apply" | Use `deferColumnManager(false)` |
| 8 | Search terms split by space | Use `splitSearchTerms(false)` |
| 9 | Records deselected on filter change | Use `deselectAllRecordsWhenFiltered(false)` |
| 10 | ToggleColumn doesn't check policies | Use `disabled()` for authorization |
| 11 | No separate `BadgeColumn` | Use `TextColumn` + `->badge()` |
