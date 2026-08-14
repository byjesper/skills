# Filament 5: Actions, Modals, Notifications & Widgets — Quick Reference

> Code-heavy reference for the four UI systems. Copy-paste ready. Assumes `use Filament\Actions\Action;` and `use Filament\Notifications\Notification;` unless otherwise stated.

---

## Table of Contents

1. [Action Types & Configuration](#1-action-types--configuration)
2. [Modal System](#2-modal-system)
3. [Action Grouping](#3-action-grouping)
4. [Notifications](#4-notifications)
5. [Widgets](#5-widgets)

---

## 1. Action Types & Configuration

### 1.1 Trigger Styles

```php
// Default (link style)
Action::make('edit')

// Button style
Action::make('edit')->button()

// Icon button (compact, icon-only)
Action::make('edit')->iconButton()

// Icon button on mobile only
Action::make('edit')->iconButton(fn() => Agent::isMobile())

// Grouped button design
Action::make('edit')->buttonGroup()

// Link style (explicit)
Action::make('edit')->link()

// Outlined button
Action::make('cancel')->button()->outlined()

// Hide label (icon only on any style)
Action::make('edit')->icon('heroicon-m-pencil-square')->hiddenLabel()
```

### 1.2 Built-in Actions

| Action | Class | Auto-Policy | Typical Use |
|--------|-------|-------------|-------------|
| Create | `CreateAction` | — | Header action on List page |
| Edit | `EditAction` | `update` | Row/table action |
| View | `ViewAction` | `view` | Row/table action |
| Delete | `DeleteAction` | `delete` | Row/table action |
| Replicate | `ReplicateAction` | — | Duplicate with data mapping |
| Force Delete | `ForceDeleteAction` | `forceDelete` | Permanently delete soft-deleted |
| Restore | `RestoreAction` | `restore` | Restore soft-deleted records |
| Import | `ImportAction` | — | Header action for CSV/Excel import |
| Export | `ExportAction` | — | Header action for CSV/Excel export |

```php
use Filament\Tables\Actions\EditAction;
use Filament\Tables\Actions\DeleteAction;
use Filament\Tables\Actions\ReplicateAction;
use Filament\Tables\Actions\RestoreAction;
use Filament\Tables\Actions\ForceDeleteAction;

// In table actions() or headerActions():
EditAction::make(),           // auto-checks 'update' policy
DeleteAction::make(),         // auto-checks 'delete' policy
ReplicateAction::make()
    ->excludeAttributes(['slug'])
    ->form([/* extra fields */]),
RestoreAction::make(),        // auto-checks 'restore' policy
ForceDeleteAction::make(),    // auto-checks 'forceDelete' policy
```

### 1.3 Colors, Sizes, Icons

```php
use Filament\Support\Enums\Size;
use Filament\Support\Enums\IconPosition;

// Colors: primary, danger, gray, info, success, warning
Action::make('save')->color('success')
Action::make('delete')->color('danger')
Action::make('cancel')->color('gray')

// Sizes
Action::make('create')->size(Size::Large)
Action::make('edit')->size(Size::Small)
// default = Size::Medium

// Icons
Action::make('edit')
    ->icon('heroicon-m-pencil-square')         // Heroicons: heroicon-o- (outline), heroicon-m- (mini), heroicon-s- (solid)
    ->iconPosition(IconPosition::After)        // Before (default) or After

// Tooltip
Action::make('delete')
    ->icon('heroicon-m-trash')
    ->tooltip('Delete this record')
```

### 1.4 Badges

```php
Action::make('filter')
    ->iconButton()
    ->icon('heroicon-m-funnel')
    ->badge(5)
    ->badgeColor('success')
```

### 1.5 Keybindings

```php
Action::make('save')
    ->action(fn() => $this->save())
    ->keyBindings(['command+s', 'ctrl+s'])
```

### 1.6 Extra Attributes

```php
Action::make('edit')
    ->extraAttributes(['class' => 'my-custom-class', 'data-foo' => 'bar'])
```

### 1.7 Disabling

```php
Action::make('edit')
    ->disabled(fn($record) => $record->isLocked())
```

### 1.8 Authorization

```php
// Closure-based
Action::make('delete')
    ->authorize(fn($record) => auth()->user()->can('delete', $record))

// Automatic policy checks (built-in actions on tables)
// EditAction → checks 'update' policy
// DeleteAction → checks 'delete' policy
// RestoreAction → checks 'restore' policy
// ForceDeleteAction → checks 'forceDelete' policy
```

### 1.9 URL Redirects (No Modal)

```php
Action::make('edit')
    ->url(fn(): string => route('posts.edit', ['post' => $this->post]))

// Open in new tab
Action::make('view')
    ->url(route('posts.show', $post), shouldOpenInNewTab: true)
```

### 1.10 Rate Limiting

```php
use Illuminate\Support\Facades\RateLimiter;

Action::make('delete')
    ->mountUsing(function () {
        $key = 'delete:' . auth()->id();
        if (RateLimiter::tooManyAttempts($key, maxAttempts: 5)) {
            Notification::make()
                ->title('Too many attempts')
                ->body('Try again in ' . RateLimiter::availableIn($key) . 's')
                ->danger()
                ->send();
            return;
        }
        RateLimiter::hit($key);
    })
```

### 1.11 Running JavaScript

```php
Action::make('print')
    ->action('window.print()')
```

### 1.12 Conditional Visibility

```php
Action::make('delete')
    ->hidden(fn($record) => $record->trashed())
    ->visible(fn() => auth()->user()->can('delete'))
```

---

## 2. Modal System

### 2.1 Confirmation Modals

```php
Action::make('delete')
    ->action(fn(Post $record) => $record->delete())
    ->requiresConfirmation()
    ->modalHeading('Delete post')
    ->modalDescription('Are you sure? This cannot be undone.')
    ->modalSubmitActionLabel('Yes, delete it')
    ->modalIcon('heroicon-o-trash')
    ->modalIconColor('warning')
```

> Confirmation modals require `action()` — they do NOT work with `url()`. Redirect inside the `action()` closure instead.

### 2.2 Form Modals (schema)

```php
use Filament\Forms\Components\Select;
use Filament\Forms\Components\TextInput;

Action::make('updateAuthor')
    ->schema([
        Select::make('authorId')
            ->label('Author')
            ->options(User::query()->pluck('name', 'id'))
            ->required(),
    ])
    ->action(function (array $data, Post $record): void {
        $record->author()->associate($data['authorId']);
        $record->save();
    })
```

**Fill form with existing data:**
```php
Action::make('updateAuthor')
    ->fillForm(fn(Post $record): array => [
        'authorId' => $record->author->id,
    ])
    ->schema([/* ... */])
    ->action(function (array $data, Post $record): void { /* ... */ })
```

**Disable all form fields (read-only modal):**
```php
Action::make('preview')
    ->schema([/* ... */])
    ->disabledForm()
```

### 2.3 Wizard Modals (steps)

```php
use Filament\Forms\Components\MarkdownEditor;
use Filament\Forms\Components\TextInput;
use Filament\Forms\Components\Toggle;
use Filament\Schemas\Components\Wizard\Step;

Action::make('create')
    ->steps([
        Step::make('Name')
            ->description('Give a unique name')
            ->schema([
                TextInput::make('name')
                    ->required()
                    ->live()
                    ->afterStateUpdated(fn($state, callable $set) => $set('slug', Str::slug($state))),
                TextInput::make('slug')
                    ->disabled()
                    ->required()
                    ->unique(Category::class, 'slug'),
            ])
            ->columns(2),
        Step::make('Description')
            ->description('Add details')
            ->schema([MarkdownEditor::make('description')]),
        Step::make('Visibility')
            ->description('Control access')
            ->schema([
                Toggle::make('is_visible')->default(true),
            ]),
    ])
    ->action(function (array $data) {
        Category::create($data);
    })
```

### 2.4 Slide-Overs

```php
Action::make('updateAuthor')
    ->schema([/* ... */])
    ->slideOver()                              // slides from right (end)
    ->action(function (array $data): void { /* ... */ })
```

```php
use Filament\Support\Enums\SlideOverPosition;

// Slide from left
Action::make('updateAuthor')
    ->slideOver()
    ->slideOverPosition(SlideOverPosition::Start)
```

### 2.5 Modal Widths

| Width | Description |
|-------|-------------|
| `xs` | Extra small, centered |
| `sm` | Small, centered |
| `md` | Medium (default) |
| `lg` | Large |
| `xl` | Extra large |
| `2xl` through `7xl` | Progressively wider |
| `screen` | Full screen |

```php
Action::make('edit')
    ->schema([/* ... */])
    ->modalWidth('lg')
```

### 2.6 Sticky Headers & Footers

```php
Action::make('edit')
    ->stickyModalHeader()   // header stays fixed while scrolling
    ->stickyModalFooter()   // footer stays fixed while scrolling
```

### 2.7 Custom Content via Blade Views

```php
use Illuminate\Contracts\View\View;

// Simple view modal
Action::make('help')
    ->modalContent(view('actions.help'))
    ->modalSubmitAction(false)
    ->modalCancelAction(false)

// With data
Action::make('advance')
    ->modalContent(fn(Action $action): View => view(
        'filament.pages.actions.advance',
        ['action' => $action],
    ))
```

**Adding actions to custom modal content:**
```php
Action::make('advance')
    ->registerModalActions([
        Action::make('report')
            ->requiresConfirmation()
            ->action(fn(Post $record) => $record->report()),
    ])
    ->modalContent(fn(Action $action): View => view(...))
```

```blade
{{-- In the Blade view --}}
<div>
    {{ $action->getModalAction('report') }}
</div>
```

**Content below the form:**
```php
Action::make('updateAuthor')
    ->schema([/* ... */])
    ->modalContentFooter(view('filament.modal-footer'))
```

### 2.8 Nested Modals

**Overlay child on parent (keep both open):**
```php
Action::make('editItems')
    ->slideOver()
    ->schema([
        Repeater::make('items')
            ->schema([/* ... */])
            ->deleteAction(
                fn(Action $action) => $action
                    ->requiresConfirmation()
                    ->overlayParentActions(),
            ),
    ])
```

**Cancel parent from child (close parent when child runs):**
```php
Action::make('edit')
    ->extraModalFooterActions([
        Action::make('delete')
            ->requiresConfirmation()
            ->cancelParentActions()   // closes the edit modal
            ->action(function() { /* delete logic */ }),
    ])
```

**Access parent action data from child:**
```php
Action::make('edit')
    ->extraModalFooterActions([
        Action::make('next')
            ->action(function(array $mountedActions) {
                $parentData = $mountedActions[0]->getRawData();
                $parentArgs = $mountedActions[0]->getArguments();
                // ...
            }),
    ])
```

### 2.9 Footer Customization

```php
// Modify submit button label
Action::make('create')
    ->modalSubmitAction(fn(StaticAction $action) => $action->label('Create & continue'))

// Remove footer buttons
Action::make('help')
    ->modalSubmitAction(false)
    ->modalCancelAction(false)

// Add extra footer actions
Action::make('create')
    ->schema([/* ... */])
    ->extraModalFooterActions(fn(Action $action): array => [
        $action->makeModalSubmitAction('createAnother', arguments: ['another' => true]),
    ])
    ->action(function (array $data, array $arguments): void {
        // Create logic
        if ($arguments['another'] ?? false) {
            // Reset form, don't close modal
            $this->resetForm();
            return;
        }
    })
```

### 2.10 Closing Control

```php
// Clicking away
Action::make('updateAuthor')
    ->closeModalByClickingAway(false)

// Pressing Escape
Action::make('updateAuthor')
    ->closeModalByEscaping(false)

// Hide close button
Action::make('edit')
    ->hiddenModalClose()

// Disable autofocus
Action::make('updateAuthor')
    ->modalAutofocus(false)

// Close programmatically
Action::make('save')
    ->action(function() {
        // ... save logic
        $this->closeModal();
    })
```

**Global defaults (service provider):**
```php
use Filament\Support\View\Components\ModalComponent;

ModalComponent::closedByClickingAway(false);
ModalComponent::closedByEscaping(false);
ModalComponent::autofocus(false);
```

### 2.11 Modal Lifecycle

```php
Action::make('create')
    ->mountUsing(function (array $data) {
        // Runs when modal OPENS, before form renders
        // Set defaults, perform checks, rate limiting
    })
    ->fillForm(fn($record) => ['status' => 'draft'])
    ->schema([/* ... */])
    ->action(function (array $data) {
        // Runs on SUBMIT after validation
    })
```

| Phase | Method | When |
|-------|--------|------|
| Mount | `mountUsing()` | Modal opens, before form renders |
| Fill | `fillForm()` | Pre-populates form data |
| Validate | Form rules | On submit, before `action()` |
| Execute | `action()` | On submit, after validation |

### 2.12 Modal Alignment & Other Options

```php
Action::make('delete')
    ->modalAlignment('center')  // 'start' (default for md+) or 'center' (default for xs/sm)

// Optimize config (skip redundant checks)
Action::make('updateAuthor')
    ->modal()

// Conditionally hide
Action::make('create')
    ->modalHidden($this->role !== 'admin')

// Extra HTML attributes
Action::make('updateAuthor')
    ->extraModalWindowAttributes(['class' => 'update-author-modal'])
    ->extraModalOverlayAttributes(['class' => 'update-author-overlay'])
```

---

## 3. Action Grouping

### 3.1 Basic Action Group

```php
use Filament\Actions\ActionGroup;

ActionGroup::make([
    Action::make('view')->icon('heroicon-m-eye'),
    Action::make('edit')->icon('heroicon-m-pencil-square'),
    Action::make('delete')->icon('heroicon-m-trash')->color('danger'),
])
```

### 3.2 Customizing the Trigger

```php
use Filament\Support\Enums\Size;

ActionGroup::make([/* ... */])
    ->label('More actions')
    ->icon('heroicon-m-ellipsis-vertical')
    ->size(Size::Small)
    ->color('primary')
    ->button()
    ->tooltip('Additional options')
```

### 3.3 Button Group (Inline, No Dropdown)

```php
use Filament\Support\Icons\Heroicon;

ActionGroup::make([
    Action::make('edit')
        ->color('gray')
        ->icon(Heroicon::PencilSquare)
        ->hiddenLabel(),
    Action::make('delete')
        ->color('gray')
        ->icon(Heroicon::Trash)
        ->hiddenLabel(),
])
->buttonGroup()
```

### 3.4 Dropdown Customization

```php
ActionGroup::make([/* ... */])
    ->dropdownPlacement('top-start')   // top-start, top-end, bottom-start, bottom-end, etc.
    ->dropdownAutoPlacement()          // auto-position based on available space
    ->dropdownWidth('xs')              // xs, sm, md, lg, xl, 2xl, 3xl, 4xl, 5xl
    ->dropdownOffset(10)               // offset in pixels
    ->dropdownMaxHeight('300px')       // max dropdown height
```

### 3.5 Dividers Between Actions

```php
ActionGroup::make([
    ActionGroup::make([
        Action::make('view'),
        Action::make('edit'),
    ]),
    // Divider rendered here
    ActionGroup::make([
        Action::make('delete'),
        Action::make('force-delete'),
    ])->color('danger'),
])
```

---

## 4. Notifications

### 4.1 Flash Notifications (Fluent API)

```php
use Filament\Notifications\Notification;

// Basic
Notification::make()
    ->title('Saved successfully')
    ->success()
    ->send();

// With body and icon
Notification::make()
    ->title('Saved successfully')
    ->body('Changes to the post have been saved.')
    ->icon('heroicon-o-check-circle')
    ->iconColor('success')
    ->success()
    ->send();

// Types
Notification::make()->success()     // green
Notification::make()->danger()      // red
Notification::make()->warning()     // yellow/orange
Notification::make()->info()        // blue
Notification::make()->secondary()   // gray (default)

// Duration
Notification::make()
    ->title('Saved')
    ->seconds(5)        // visible for 5 seconds
    ->success()
    ->send();

// Persistent (manual close required)
Notification::make()
    ->title('Important')
    ->persistent()
    ->danger()
    ->send();

// Markdown in title/body (safe HTML via Str::markdown())
Notification::make()
    ->title(Str::markdown('Saved **successfully**'))
    ->body(Str::markdown('Changes to the **post** have been saved.'))
    ->send();
```

### 4.2 Notification Actions

```php
use Filament\Actions\Action;

Notification::make()
    ->title('Saved successfully')
    ->success()
    ->body('Changes have been saved.')
    ->actions([
        Action::make('view')
            ->button()
            ->url(route('posts.show', $post), shouldOpenInNewTab: true),
        Action::make('undo')
            ->color('gray')
            ->dispatch('undoEditingPost', [$post->id])
            ->close(),  // closes the notification
    ])
    ->send();

// Dispatch targets
Action::make('undo')->dispatch('undoEditingPost')               // to parent
Action::make('undo')->dispatchSelf('undoEditingPost')            // to self
Action::make('undo')->dispatchTo('component', 'undoEditingPost') // to specific component
```

### 4.3 JavaScript API

```javascript
// Basic
new FilamentNotification()
    .title('Saved successfully')
    .success()
    .send()

// With body and actions
new FilamentNotification()
    .title('Saved successfully')
    .success()
    .body('Changes have been saved.')
    .actions([
        new FilamentNotificationAction('view')
            .button()
            .url('/view')
            .openUrlInNewTab(),
        new FilamentNotificationAction('undo')
            .color('gray')
            .dispatch('undoEditingPost')
            .close(),
    ])
    .send()

// Persistent
new FilamentNotification()
    .title('Important')
    .persistent()
    .danger()
    .send()
```

**Bundled JS import:**
```javascript
import { Notification, NotificationAction }
    from '../../vendor/filament/notifications/dist/index.js'
```

### 4.4 Closing Notifications

```php
// From action
Action::make('view')->close()   // close notification after action runs

// With custom ID (close via JS/DOM)
Notification::make('greeting')->title('Hello')->persistent()->send();
```
```html
<button x-on:click="$dispatch('close-notification', { id: 'greeting' })">
    Close
</button>
```

### 4.5 Positioning

```php
use Filament\Notifications\Livewire\Notifications;
use Filament\Support\Enums\Alignment;
use Filament\Support\Enums\VerticalAlignment;

// In service provider or middleware
Notifications::alignment(Alignment::Start);           // Start, Center, End
Notifications::verticalAlignment(VerticalAlignment::End);  // Start, Center, End
```

### 4.6 Database Notifications

**Setup:**
```bash
php artisan make:notifications-table
```

> PostgreSQL: use `json()` for the `data` column. UUID: use `uuidMorphs('notifiable')`.

**Enable in panel:**
```php
use Filament\Panel;

public function panel(Panel $panel): Panel
{
    return $panel
        // ...
        ->databaseNotifications()
        ->databaseNotificationsTrigger('sidebar');  // move trigger to sidebar
}
```

**Send database notifications:**
```php
// Via fluent API
Notification::make()
    ->title('Saved successfully')
    ->sendToDatabase($recipient);

// Via notify() method
$recipient->notify(
    Notification::make()
        ->title('Saved successfully')
        ->toDatabase()
);
```

Database notifications auto-poll. Set up Laravel Echo for real-time delivery.

### 4.7 Broadcast Notifications (Websockets)

**Enable in panel:**
```php
public function panel(Panel $panel): Panel
{
    return $panel
        // ...
        ->broadcastNotifications();
}
```

**Send broadcast notifications:**
```php
// Via fluent API
Notification::make()
    ->title('Saved successfully')
    ->broadcast($recipient);

// Via notify() method
$recipient->notify(
    Notification::make()
        ->title('Saved successfully')
        ->toBroadcast()
);

// In a Laravel notification class
use Illuminate\Notifications\Messages\BroadcastMessage;

public function toBroadcast(User $notifiable): BroadcastMessage
{
    return Notification::make()
        ->title('Saved successfully')
        ->getBroadcastMessage();
}
```

---

## 5. Widgets

### 5.1 Common Widget Configuration

```php
// Sorting
protected static ?int $sort = 2;

// Grid span (1-6 columns or 'full')
protected int $columnSpan = 2;
protected int|string|array $columnSpan = [
    'sm' => 1, 'md' => 2, 'lg' => 3, 'xl' => 4,
];

// Lazy loading (default: true)
protected static bool $isLazy = true;   // load only when visible
protected static bool $isLazy = false;  // load immediately

// Polling
protected ?string $pollingInterval = '5s';  // default for stats
protected ?string $pollingInterval = null;  // disable

// Heading & description
protected ?string $heading = 'Analytics';
protected ?string $description = 'An overview of some analytics.';

// Visibility
public static function canView(): bool
{
    return auth()->user()->isAdmin();
}
```

**Widget Registration in Panel:**
```php
use Filament\Panel;

public function panel(Panel $panel): Panel
{
    return $panel
        // ...
        ->widgets([
            \App\Filament\Widgets\StatsOverview::class,
            \App\Filament\Widgets\BlogPostsChart::class,
            \App\Filament\Widgets\LatestOrders::class,
        ]);
}
```

### 5.2 Stats Overview Widget

```bash
php artisan make:filament-widget StatsOverview --stats-overview
```

```php
use Filament\Widgets\StatsOverviewWidget as BaseWidget;
use Filament\Widgets\StatsOverviewWidget\Stat;

class StatsOverview extends BaseWidget
{
    protected function getStats(): array
    {
        return [
            // Basic stat
            Stat::make('Unique views', '192.1k'),

            // With description and icon
            Stat::make('Bounce rate', '21%')
                ->description('32k increase')
                ->descriptionIcon('heroicon-m-arrow-trending-up'),

            // With color and sparkline
            Stat::make('Avg. time', '3:12')
                ->description('12% increase')
                ->descriptionIcon('heroicon-m-arrow-trending-up')
                ->chart([7, 2, 10, 3, 15, 4, 17])
                ->color('success'),

            // With interaction
            Stat::make('Processed', Order::processed()->count())
                ->extraAttributes([
                    'class' => 'cursor-pointer',
                    'wire:click' => "\$dispatch('setStatusFilter', { filter: 'processed' })",
                ]),
        ];
    }
}
```

**Stat colors:** `success`, `danger`, `warning`, `info`, `primary`, `gray`

### 5.3 Chart Widgets

```bash
php artisan make:filament-widget BlogPostsChart --chart
```

```php
use Filament\Widgets\ChartWidget;

class BlogPostsChart extends ChartWidget
{
    protected ?string $heading = 'Blog Posts';
    protected ?string $maxHeight = '300px';
    protected ?string $pollingInterval = '10s';
    protected bool $isCollapsible = true;

    protected function getType(): string
    {
        return 'line';  // See chart types below
    }

    protected function getData(): array
    {
        return [
            'datasets' => [
                [
                    'label' => 'Blog posts created',
                    'data' => [0, 10, 5, 2, 21, 32, 45, 74, 65, 45, 77, 89],
                    'backgroundColor' => '#36A2EB',
                    'borderColor' => '#9BD0F5',
                ],
            ],
            'labels' => ['Jan', 'Feb', 'Mar', 'Apr', 'May', 'Jun',
                         'Jul', 'Aug', 'Sep', 'Oct', 'Nov', 'Dec'],
        ];
    }
}
```

**Chart Types:**

| Type | Value |
|------|-------|
| Line | `'line'` |
| Bar | `'bar'` |
| Scatter | `'scatter'` |
| Bubble | `'bubble'` |
| Pie | `'pie'` |
| Doughnut | `'doughnut'` |
| Polar Area | `'polarArea'` |
| Radar | `'radar'` |

**Chart.js Options:**
```php
use Filament\Support\RawJs;

protected function getOptions(): array
{
    return [
        'plugins' => [
            'legend' => ['display' => false],
        ],
    ];
}

// Raw JavaScript for callbacks
protected function getOptions(): RawJs
{
    return RawJs::make(<<<JS
    {
        scales: {
            y: {
                ticks: {
                    callback: (value) => '\u20AC' + value,
                },
            },
        },
    }
    JS);
}
```

**Advanced Chart.js configuration with custom tooltips and styling:**
```php
protected function getOptions(): array
{
    return [
        'plugins' => [
            'legend' => ['display' => true, 'position' => 'bottom'],
            'tooltip' => [
                'mode' => 'index',
                'intersect' => false,
            ],
        ],
        'scales' => [
            'y' => [
                'beginAtZero' => true,
                'grid' => [
                    'color' => 'rgba(0, 0, 0, 0.05)',
                    'borderDash' => [5, 5],
                ],
            ],
            'x' => [
                'grid' => ['display' => false],
            ],
        ],
        'elements' => [
            'line' => ['tension' => 0.3],
            'point' => ['radius' => 3, 'hitRadius' => 10],
        ],
        'interaction' => [
            'mode' => 'nearest',
            'axis' => 'x',
            'intersect' => false,
        ],
    ];
}
```

**Chart Filters (Basic Select):**
```php
class BlogPostsChart extends ChartWidget
{
    public ?string $filter = 'today';

    protected function getFilters(): ?array
    {
        return [
            'today' => 'Today',
            'week' => 'Last week',
            'month' => 'Last month',
            'year' => 'This year',
        ];
    }

    protected function getData(): array
    {
        $activeFilter = $this->filter;
        // ... adjust query based on $activeFilter
    }
}
```

**Chart Filters (Schema/Form-based):**
```php
use Filament\Widgets\ChartWidget\Concerns\HasFiltersSchema;

class BlogPostsChart extends ChartWidget
{
    use HasFiltersSchema;

    public function filtersSchema(Schema $schema): Schema
    {
        return $schema->components([
            DatePicker::make('startDate')->default(now()->subDays(30)),
            DatePicker::make('endDate')->default(now()),
        ]);
    }

    protected function getData(): array
    {
        $startDate = $this->filters['startDate'] ?? null;
        $endDate = $this->filters['endDate'] ?? null;
        // ... use in queries
    }
}
```

**Generating data from Eloquent (with laravel-trend):**
```php
use Flowframe\Trend\Trend;
use Flowframe\Trend\TrendValue;

protected function getData(): array
{
    $data = Trend::model(BlogPost::class)
        ->between(start: now()->startOfYear(), end: now()->endOfYear())
        ->perMonth()
        ->count();

    return [
        'datasets' => [
            [
                'label' => 'Blog posts',
                'data' => $data->map(fn(TrendValue $v) => $v->aggregate),
            ],
        ],
        'labels' => $data->map(fn(TrendValue $v) => $v->date),
    ];
}
```

**Custom Chart.js Plugins:**
```javascript
// resources/js/filament-chart-js-plugins.js
import ChartDataLabels from 'chartjs-plugin-datalabels'

window.filamentChartJsPlugins ??= []
window.filamentChartJsPlugins.push(ChartDataLabels)

// For global plugins:
window.filamentChartJsGlobalPlugins ??= []
window.filamentChartJsGlobalPlugins.push(ChartDataLabels)
```

Add to `vite.config.js` input array, run `npm run build`, then register:
```php
use Filament\Support\Assets\Js;
use Filament\Support\Facades\FilamentAsset;

FilamentAsset::register([
    Js::make('chart-js-plugins', Vite::asset('resources/js/filament-chart-js-plugins.js'))
        ->module(),
]);
```

### 5.4 Table Widgets

```bash
php artisan make:filament-widget LatestOrders --table
```

Table widgets extend the table builder — all table API works (columns, filters, actions, bulk actions, pagination, sorting, searching):

```php
use Filament\Tables;
use Filament\Tables\Table;
use Filament\Widgets\TableWidget as BaseWidget;

class LatestOrders extends BaseWidget
{
    protected int|string|array $columnSpan = 'full';
    protected ?string $heading = 'Latest Orders';

    public function table(Table $table): Table
    {
        return $table
            // Always eager load relationships to prevent N+1
            ->query(
                Order::query()
                    ->with(['customer', 'items'])
                    ->latest()
                    ->limit(10)
            )
            ->columns([
                Tables\Columns\TextColumn::make('id')
                    ->copyable()
                    ->sortable(),
                Tables\Columns\TextColumn::make('customer.name')
                    ->searchable()
                    ->sortable(),
                Tables\Columns\TextColumn::make('total')
                    ->money('USD')
                    ->sortable()
                    ->alignment('end'),
                Tables\Columns\TextColumn::make('status')
                    ->badge()
                    ->color(fn (string $state): string => match ($state) {
                        'pending' => 'warning',
                        'processing' => 'info',
                        'shipped' => 'primary',
                        'delivered' => 'success',
                        'cancelled' => 'danger',
                        default => 'gray',
                    }),
                Tables\Columns\TextColumn::make('created_at')
                    ->since()
                    ->dateTooltip()
                    ->sortable(),
            ])
            ->actions([
                Tables\Actions\ViewAction::make(),
            ])
            ->striped()
            ->paginated(false);
    }
}
```

### 5.5 Custom Widgets (Livewire + Blade)

```bash
php artisan make:filament-widget BlogPostsOverview
```

```php
// App/Filament/Widgets/BlogPostsOverview.php
namespace App\Filament\Widgets;

use Filament\Widgets\Widget;

class BlogPostsOverview extends Widget
{
    protected static string $view = 'filament.widgets.blog-posts-overview';
    protected int|string|array $columnSpan = 2;

    protected function getViewData(): array
    {
        return [
            'posts' => BlogPost::published()->count(),
            'drafts' => BlogPost::draft()->count(),
        ];
    }
}
```

```blade
{{-- resources/views/filament/widgets/blog-posts-overview.blade.php --}}
<x-filament-widgets::widget>
    <x-filament::section>
        <div class="grid grid-cols-2 gap-4">
            <div>
                <p class="text-sm text-gray-500">Published</p>
                <p class="text-2xl font-bold">{{ $posts }}</p>
            </div>
            <div>
                <p class="text-sm text-gray-500">Drafts</p>
                <p class="text-2xl font-bold">{{ $drafts }}</p>
            </div>
        </div>
    </x-filament::section>
</x-filament-widgets::widget>
```

### 5.6 Dashboard Filtering

**Always-visible filter form:**
```php
use Filament\Pages\Dashboard as BaseDashboard;
use Filament\Pages\Dashboard\Concerns\HasFiltersForm;

class Dashboard extends BaseDashboard
{
    use HasFiltersForm;

    protected function filtersForm(): array
    {
        return [
            DatePicker::make('startDate'),
            DatePicker::make('endDate'),
        ];
    }
}
```

**Access in widgets:**
```php
use Filament\Widgets\Concerns\InteractsWithPageFilters;

class BlogPostsChart extends ChartWidget
{
    use InteractsWithPageFilters;

    protected function getData(): array
    {
        $startDate = $this->filters['startDate'] ?? null;
        $endDate = $this->filters['endDate'] ?? null;
        // ... use in queries
    }
}
```

**Filter action (modal-based):**
```php
use Filament\Pages\Dashboard as BaseDashboard;
use Filament\Pages\Dashboard\Actions\FilterAction;
use Filament\Pages\Dashboard\Concerns\HasFiltersAction;

class Dashboard extends BaseDashboard
{
    use HasFiltersAction;

    protected function getHeaderActions(): array
    {
        return [
            FilterAction::make()
                ->schema([
                    DatePicker::make('startDate'),
                    DatePicker::make('endDate'),
                ]),
        ];
    }
}
```

**Persist filters:**
```php
class Dashboard extends BaseDashboard
{
    use HasFiltersForm;
    protected bool $persistsFiltersInSession = false;
}
```

**Disable default widgets:**
```php
public function panel(Panel $panel): Panel
{
    return $panel->widgets([]);
}
```

---

## 6. Quick-Copy Cheat Sheets

### 6.1 Action One-Liner

```php
Action::make('name')
    ->button() | ->iconButton() | ->link() | ->buttonGroup()
    ->outlined()->hiddenLabel()
    ->label('Label')->color('primary'|'danger'|'gray'|'info'|'success'|'warning')
    ->size(Size::Small|Medium|Large)
    ->icon('heroicon-m-pencil-square')->iconPosition(IconPosition::Before|After)
    ->badge(5)->badgeColor('success')
    ->requiresConfirmation()
    ->modalHeading('Heading')->modalDescription('Desc')->modalSubmitActionLabel('Submit')
    ->modalIcon('heroicon-o-trash')->modalIconColor('warning')->modalWidth('lg')
    ->slideOver() | ->slideOverPosition(SlideOverPosition::Start)
    ->stickyModalHeader()->stickyModalFooter()
    ->closeModalByClickingAway(false)->closeModalByEscaping(false)
    ->hiddenModalClose()->modalAutofocus(false)
    ->schema([/* form */])->steps([/* wizard */])
    ->fillForm(fn($record) => [...])->disabledForm()
    ->action(function(array $data) { /* ... */ })
    ->url(fn() => route('...'))->url(route('...'), shouldOpenInNewTab: true)
    ->mountUsing(function() { /* ... */ })
    ->authorize(fn($record) => auth()->user()->can('...', $record))
    ->hidden(fn($record) => $record->trashed())
    ->disabled(fn($record) => $record->isLocked())
    ->keyBindings(['command+s', 'ctrl+s'])
    ->extraAttributes(['class' => '...'])
```

### 6.2 Notification One-Liner

```php
Notification::make('id')  // optional ID
    ->title('Title')->body('Body')
    ->icon('heroicon-o-check-circle')->iconColor('success')
    ->success()|->danger()|->warning()|->info()|->secondary()
    ->seconds(5)|->persistent()
    ->actions([Action::make('view')->url($url)->close()])
    ->send();

// Database: ->sendToDatabase($user)
// Broadcast: ->broadcast($user)

// JS:
new FilamentNotification().title('...').success().body('...').send();
```

### 6.3 Widget Properties

```php
// Stats
protected ?string $heading = 'Title';
protected ?string $description = 'Description';
protected ?string $pollingInterval = '5s';
protected static bool $isLazy = true;
protected int|string|array $columnSpan = 1;

// Chart
protected ?string $heading = 'Title';
protected ?string $maxHeight = '300px';
protected ?string $pollingInterval = null;
protected bool $isCollapsible = false;
protected static bool $isLazy = true;
public ?string $filter = 'today';

// All widgets
protected static ?int $sort = 2;
public static function canView(): bool { return true; }
```
