# Filament 5 - Panels, Navigation, Theming, Relationships & Advanced Patterns

> **Quick Reference** | Practical, code-heavy guide for building production-grade Filament 5 panels.

---

## 1. Panel Configuration

### PanelProvider Setup

Every panel is a Laravel service provider. Default panel: `app/Providers/Filament/AdminPanelProvider.php` -> `/admin`.

```bash
# Create a new panel
php artisan make:filament-panel app
# Creates AppPanelProvider.php - register in bootstrap/providers.php
```

```php
use Filament\Panel;
use Filament\Support\Enums\Width;
use Filament\Pages\Enums\SubNavigationPosition;

public function panel(Panel $panel): Panel
{
    return $panel
        ->id('admin')                          // Unique panel ID
        ->path('admin')                        // URL path (empty = root)
        ->domain('admin.example.com')          // Scope to domain
        ->maxContentWidth(Width::Full)         // Default: SevenExtraLarge
        ->simplePageMaxContentWidth(Width::Large)  // Login/register pages
        ->subNavigationPosition(SubNavigationPosition::End)  // Start, End, Top
        ->spa()                                // Enable SPA navigation
        ->unsavedChangesAlerts()               // Warn on unsaved changes
        ->databaseTransactions()               // Global DB transactions
        ->strictAuthorization()                // Throw if policy missing
        ->broadcasting(false)                  // Disable Echo auto-connect
        ->bootUsing(function (Panel $panel) {
            // Runs on every request in this panel
        });
}
```

### SPA Mode

```php
$panel->spa()                                     // Enable SPA navigation
$panel->spa(hasPrefetching: true)                 // With hover prefetching
$panel->spaUrlExceptions(['/external'])            // Exclude URLs from SPA
```

### Render Hooks

```php
use Filament\View\PanelsRenderHook;
use Illuminate\Support\Facades\Blade;

$panel->renderHook(
    PanelsRenderHook::BODY_START,
    fn (): string => Blade::render('@livewire(\'livewire-ui-modal\')'),
);
```

| Hook Constant | Location |
|---|---|
| `BODY_START` / `BODY_END` | Inside `<body>` tag |
| `HEAD_START` / `HEAD_END` | Inside `<head>` tag |
| `SCRIPTS_BEFORE` / `SCRIPTS_AFTER` | Livewire scripts |
| `STYLES_BEFORE` / `STYLES_AFTER` | Filament styles |
| `SIDEBAR_NAV_START` / `SIDEBAR_NAV_END` | Sidebar navigation |
| `TOPBAR_START` / `TOPBAR_END` | Top bar |
| `FOOTER` | Page footer |
| `RESOURCE_PAGES_FORM_BEFORE` / `_AFTER` | Resource form area |

### Middleware

```php
// Apply to all routes
$panel->middleware([\App\Http\Middleware\CustomMiddleware::class])

// Persistent middleware (survives Livewire AJAX requests)
$panel->middleware([\App\Http\Middleware\PersistentMiddleware::class], isPersistent: true)

// Auth-only routes
$panel->authMiddleware([\App\Http\Middleware\AdminMiddleware::class])
$panel->authMiddleware([\App\Http\Middleware\PersistentAuth::class], isPersistent: true)
```

### Asset Registration

```php
use Filament\Support\Assets\Css;
use Filament\Support\Assets\Js;

$panel->assets([
    Css::make('custom-stylesheet', resource_path('css/custom.css')),
    Js::make('custom-script', resource_path('js/custom.js')),
]);
// Then run: php artisan filament:assets
```

### Database Transactions

```php
// Global enable
$panel->databaseTransactions()

// Per-action opt-in
CreateAction::make()->databaseTransaction()

// Per-page opt-in
protected ?bool $hasDatabaseTransactions = true;

// Opt-out when globally enabled
CreateAction::make()->databaseTransaction(false);
protected ?bool $hasDatabaseTransactions = false;
```

### Error Notifications

```php
$panel
    ->registerErrorNotification(
        title: 'An error occurred',
        body: 'Please try again later.',
    )
    ->hiddenErrorNotification(403)      // Hide for specific status
    ->disabledErrorNotification(503);   // Fall back to Livewire

// Per-page
protected ?bool $hasErrorNotifications = true;
public function hasErrorNotifications(): bool { return true; }
```

### Strict Authorization & Broadcasting

```php
$panel->strictAuthorization()     // Throws exception if resource lacks policy
$panel->broadcasting(false)       // Disable Echo auto-connect if unused
```

### Multi-Panel Setup

```php
// Panel A: Admin
$panel
    ->id('admin')
    ->path('admin')
    ->resources([AdminPostResource::class])
    ->login();

// Panel B: App
$panel
    ->id('app')
    ->path('')
    ->resources([AppPostResource::class])
    ->registration();

// Cross-panel URLs
CustomerResource::getUrl(panel: 'marketing');
```

---

## 2. Navigation

### Resource Navigation

```php
// Static properties (on Resource or Page)
protected static ?string $navigationLabel = 'Custom Label';
protected static string|BackedEnum|null $navigationIcon = Heroicon::OutlinedDocumentText;
protected static ?string $navigationGroup = 'Settings';
protected static ?int $navigationSort = 2;
protected static ?string $navigationBadge = 'new';
protected static ?string $navigationBadgeColor = 'primary';
protected static ?string $navigationDescription = 'Manage settings';
protected static string|BackedEnum|null $activeNavigationIcon = Heroicon::SolidDocumentText;

// Parent item (nested under another)
protected static ?string $navigationParentItem = 'Products';
// Must also define $navigationGroup if parent has one
```

### Dynamic Navigation

```php
public static function getNavigationLabel(): string
{
    return __('custom.navigation.label');
}

public static function getNavigationBadge(): ?string
{
    return static::getModel()::count();
}

public static function getNavigationBadgeColor(): ?string
{
    return static::getModel()::count() > 10 ? 'warning' : 'primary';
}
```

### Custom Navigation Items

```php
use Filament\Navigation\NavigationItem;
use Filament\Support\Icons\Heroicon;

$panel->navigationItems([
    NavigationItem::make('Analytics')
        ->url('https://example.com', shouldOpenInNewTab: true)
        ->icon(Heroicon::OutlinedPresentationChartLine)
        ->group('Reports')
        ->sort(3),

    NavigationItem::make('dashboard')
        ->label(fn (): string => __('filament-panels::pages/dashboard.title'))
        ->url(fn (): string => Dashboard::getUrl())
        ->isActiveWhen(fn () => request()->routeIs('filament.admin.pages.dashboard')),
]);
```

### Navigation Groups

```php
use Filament\Navigation\NavigationGroup;

$panel->navigationGroups([
    NavigationGroup::make()
        ->label('Shop')
        ->icon(Heroicon::OutlinedShoppingCart),
    NavigationGroup::make()
        ->label(fn (): string => __('navigation.settings'))
        ->icon(Heroicon::OutlinedCog6Tooth)
        ->collapsed(),
]);

// Reorder groups
$panel->navigationGroups(['Shop', 'Blog', 'Settings']);

// Non-collapsible
NavigationGroup::make()->label('Settings')->collapsible(false);
// Or globally:
$panel->collapsibleNavigationGroups(false);
```

### Conditional Visibility

```php
// Hide from nav (does NOT control access)
protected static bool $shouldRegisterNavigation = false;
public static function shouldRegisterNavigation(): bool { return false; }

// Custom nav items
NavigationItem::make('Analytics')
    ->visible(fn (): bool => auth()->user()->can('view-analytics'))
    ->hidden(fn (): bool => !auth()->user()->can('view-analytics'));
```

### Top Navigation Mode

```php
$panel->topNavigation();
```

### Sidebar Customization

```php
$panel->sidebarCollapsibleOnDesktop();       // Collapsible on desktop
$panel->sidebarFullyCollapsibleOnDesktop();  // Fully collapsed on desktop
$panel->sidebarWidth('20rem');
$panel->collapsedSidebarWidth('4rem');
$panel->sidebar(false);                       // Disable sidebar
$panel->topbar(false);                        // Disable topbar
$panel->breadcrumbs(false);                   // Disable breadcrumbs
$panel->navigation(false);                    // Disable nav entirely
```

### Breadcrumbs & Refresh

```php
$panel->breadcrumbs(false);

// Refresh sidebar/topbar dynamically
$this->dispatch('refresh-sidebar');
$this->dispatch('refresh-topbar');
```

---

## 3. Theming

### Color Palettes (OKLCH & Presets)

```php
use Filament\Support\Colors\Color;

$panel->colors([
    'danger'  => Color::Rose,
    'gray'    => Color::Gray,
    'info'    => Color::Blue,
    'primary' => Color::Indigo,
    'success' => Color::Emerald,
    'warning' => Color::Orange,
]);

// Custom OKLCH from hex
'primary' => Color::hex('#1da1f2'),

// Available presets: Amber, Blue, Cyan, Emerald, Fuchsia, Gray, Green,
// Indigo, Lime, Orange, Pink, Purple, Red, Rose, Sky, Slate, Stone,
// Teal, Violet, Yellow, Zinc
```

### Custom Themes

```bash
php artisan make:filament-theme
# Creates resources/css/filament/admin/theme.css
```

```css
@import "tailwindcss";
@source "../../../../app/Filament/**/*";
@source "../../../../resources/views/filament/**/*";
@plugin "../path/to/plugin.js";
```

```php
// Register theme
$panel->viteTheme('resources/css/filament/admin/theme.css');
// Compile: npm run build
```

> **Critical:** Tailwind classes in Blade views won't work without a custom theme.

### Fonts (GoogleFontProvider)

```php
use Filament\Font\Providers\GoogleFontProvider;

$panel->font('Inter', provider: GoogleFontProvider::class);
$panel->font('DM Sans', provider: LocalFontProvider::class);
```

### Logos & Favicons

```php
$panel
    ->brandLogo(fn () => view('filament.admin.logo'))
    ->brandLogoHeight('2rem')
    ->darkModeBrandLogo(fn () => view('filament.admin.dark-logo'))
    ->favicon(asset('images/favicon.png'));
```

### Dark Mode

```php
use Filament\Enums\ThemeMode;

$panel->darkMode(false);                         // Disable toggle
$panel->defaultThemeMode(ThemeMode::System);     // light | dark | system
```

### Tailwind CSS Integration

- Filament uses Tailwind CSS v4 with `@import "tailwindcss"` syntax
- Custom themes extend Filament's Tailwind config
- All Filament components use Tailwind utility classes
- Use `@source` directives to scan custom directories
- Run `npm run build` after any theme changes
- Use `php artisan filament:assets` for asset publication

---

## 4. Multi-Tenancy

### Contracts

```php
use Filament\Models\Contracts\FilamentUser;
use Filament\Models\Contracts\HasDefaultTenant;
use Filament\Models\Contracts\HasTenants;

class User extends Model implements FilamentUser, HasDefaultTenant, HasTenants
{
    public function getTenants(Panel $panel): array|Collection
    {
        return $this->teams;
    }

    public function canAccessTenant(Model $tenant): bool
    {
        return $this->teams->contains($tenant);
    }

    public function getDefaultTenant(Panel $panel): ?Model
    {
        return $this->latestTeam;
    }
}
```

```php
use Filament\Models\Contracts\HasCurrentTenantLabel;
use Filament\Models\Contracts\HasName;

class Team extends Model implements HasCurrentTenantLabel, HasName
{
    public function members(): BelongsToMany
    {
        return $this->belongsToMany(User::class);
    }

    public function getFilamentName(): string
    {
        return "{$this->name} {$this->subscription_plan}";
    }

    public function getCurrentTenantLabel(): string
    {
        return 'Active team';
    }
}
```

### Panel Configuration

```php
$panel
    ->tenant(Team::class)
    ->tenantRegistration(\App\Filament\Pages\Tenancy\RegisterTeam::class)
    ->tenantProfile(\App\Filament\Pages\Tenancy\EditTeamProfile::class)
    ->tenantBillingProvider(\App\Billing\ExampleBillingProvider::class)
    ->requiresSubscription()
    ->tenantMenuItems([
        \Filament\Actions\Action::make('settings')
            ->url(fn (): string => Settings::getUrl())
            ->icon('heroicon-m-cog-8-tooth'),
    ])
    ->searchableTenantMenu()
    ->tenantBillingRouteSlug('billing')
    ->tenantRoutePrefix('t')              // /admin/t/{tenant}/...
    ->domain('{tenant:slug}.example.com'); // Domain-based
```

### Tenant Registration Page

```php
use Filament\Pages\Tenancy\RegisterTenant;

class RegisterTeam extends RegisterTenant
{
    public static function getLabel(): string { return 'Register team'; }

    public function form(Schema $schema): Schema
    {
        return $schema->components([
            TextInput::make('name')->required(),
        ]);
    }

    public function handleRegistration(array $data): Model
    {
        $team = Team::create($data);
        $team->members()->attach(auth()->user());
        return $team;
    }
}
```

### Tenant Profile Page

```php
use Filament\Pages\Tenancy\EditTenantProfile;

class EditTeamProfile extends EditTenantProfile
{
    public static function getLabel(): string { return 'Team profile'; }

    public function form(Schema $schema): Schema
    {
        return $schema->components([
            TextInput::make('name'),
        ]);
    }
}
```

### Billing Providers & Subscriptions

```php
use Filament\Billing\Providers\Contracts\BillingProvider;

class ExampleBillingProvider implements BillingProvider
{
    public function getRouteAction(): string
    {
        return fn (): RedirectResponse => redirect('https://billing.example.com');
    }

    public function getSubscribedMiddleware(): string
    {
        return RedirectIfUserNotSubscribed::class;
    }
}
```

```php
// Require subscription globally
$panel->requiresSubscription();

// For specific resources/pages
$panel->requiresSubscription([PostResource::class, AnalyticsPage::class]);
```

### Accessing Current Tenant

```php
use Filament\Facades\Filament;

$tenant = Filament::getTenant();
$tenantId = Filament::getTenant()->getKey();
```

### Tenant Menu

```php
$panel->tenantMenuItems([
    \Filament\Actions\Action::make('settings')
        ->url(fn (): string => Settings::getUrl())
        ->icon('heroicon-m-cog-8-tooth')
        ->hidden(fn () => !auth()->user()->can('view_settings'))
        ->action(fn () => $this->dispatch('some-event')),
]);
```

### Tenant-Aware Middleware

```php
class ApplyTenantScopes
{
    public function handle(Request $request, Closure $next)
    {
        Author::addGlobalScope('tenant', fn (Builder $query) =>
            $query->whereBelongsTo(Filament::getTenant()),
        );
        return $next($request);
    }
}

// Register as persistent
$panel->tenantMiddleware([ApplyTenantScopes::class], isPersistent: true);
```

### Disabling Tenancy for Resources

```php
class PostResource extends Resource
{
    protected static bool $isScopedToTenant = false;
}
```

### Simple One-to-Many Tenancy (Don't Use Filament's System)

```php
class Post extends Model
{
    protected static function booted(): void
    {
        static::addGlobalScope('team', function (Builder $query) {
            if (auth()->hasUser()) {
                $query->whereBelongsTo(auth()->user()->team);
            }
        });
    }
}

class PostObserver
{
    public function creating(Post $post): void
    {
        $post->team_id = auth()->user()->team_id;
    }
}
```

---

## 5. Relationship Management

### Choosing the Right Tool

| Tool | Compatible Relationships | Use Case |
|------|-------------------------|----------|
| **Relation Managers** | HasMany, HasManyThrough, BelongsToMany, MorphMany, MorphToMany | Interactive tables for full CRUD |
| **Select / Checkbox List** | BelongsTo, MorphTo, BelongsToMany | Choose existing or create inline |
| **Repeaters** | HasMany, MorphMany | Inline CRUD in owner's form |
| **Layout Components** | BelongsTo, HasOne, MorphOne | Save fields to single relationship |

### Relation Managers (Full CRUD)

```bash
php artisan make:filament-relation-manager CategoryResource posts title
# Creates CategoryResource/RelationManagers/PostsRelationManager.php
```

```php
// In resource
public static function getRelations(): array
{
    return [
        RelationManagers\PostsRelationManager::class,
        // With custom URL key:
        'posts' => RelationManagers\PostsRelationManager::class, // ?relation=posts
    ];
}

// Relation Manager class
public function form(Schema $schema): Schema
{
    return $schema->components([
        Forms\Components\TextInput::make('title')->required(),
    ]);
}

public function table(Table $table): Table
{
    return $table->columns([
        Tables\Columns\TextColumn::make('title'),
    ]);
}
```

#### Read-Only Mode

```php
public function isReadOnly(): bool
{
    return false; // Allow editing even on View page
}
```

#### Soft Deletes

```bash
php artisan make:filament-relation-manager CategoryResource posts title --soft-deletes
```

### Pivot Attributes (BelongsToMany / MorphToMany)

```php
// List pivot attributes
public function table(Table $table): Table
{
    return $table->columns([
        Tables\Columns\TextColumn::make('name'),
        Tables\Columns\TextColumn::make('role'), // pivot
    ]);
}

// Edit pivot attributes
public function form(Schema $schema): Schema
{
    return $schema->components([
        Forms\Components\TextInput::make('name')->required(),
        Forms\Components\TextInput::make('role')->required(), // pivot
    ]);
}
```

> **Must list in `withPivot()` on BOTH the relationship AND inverse relationship.**

### Attaching & Detaching

```php
use Filament\Actions\AttachAction;

// Basic attach with pivot
AttachAction::make()
    ->form(fn (AttachAction $action): array => [
        $action->getRecordSelect(),
        Forms\Components\TextInput::make('role')->required(),
    ]);

// Scope options
AttachAction::make()
    ->recordSelectOptionsQuery(fn (Builder $query) => $query->where('is_active', true));

// Multi-column search
AttachAction::make()
    ->recordSelectSearchColumns(['title', 'description']);

// Attach multiple
AttachAction::make()->multiple();

// Modal table selection
AttachAction::make()->tableSelect();

// Allow duplicates (requires `id` on pivot in `withPivot()`)
public function table(Table $table): Table
{
    return $table->allowDuplicates();
}
```

### Associating & Dissociating (HasMany / MorphMany)

```php
use Filament\Actions\AssociateAction;
use Filament\Actions\DissociateBulkAction;

AssociateAction::make()->multiple();
AssociateAction::make()->preloadRecordSelect();
AssociateAction::make()
    ->recordSelectOptionsQuery(fn (Builder $q) => $q->where('is_active', true));

// Bulk dissociate optimizations
DissociateBulkAction::make()->chunkSelectedRecords(250);
DissociateBulkAction::make()->fetchSelectedRecords(false); // single query
```

### Select/CheckboxList for BelongsTo/MorphTo

```php
use Filament\Forms\Components\Select;
use Filament\Forms\Components\CheckboxList;

Select::make('category_id')
    ->relationship('category', 'name')
    ->createOptionForm([/* ... */])
    ->editOptionForm([/* ... */]);

CheckboxList::make('tags')
    ->relationship('tags', 'name');
```

### Repeater for HasMany Inline

```php
use Filament\Forms\Components\Repeater;

Repeater::make('comments')
    ->relationship('comments')
    ->schema([
        Forms\Components\TextInput::make('body')->required(),
    ])
    ->addable()
    ->deletable()
    ->reorderable()
    ->collapsible();
```

### Layout Components for BelongsTo/HasOne

```php
use Filament\Schemas\Components\Group;
use Filament\Schemas\Components\Section;

Group::make()
    ->relationship('profile')
    ->schema([
        Forms\Components\TextInput::make('bio'),
        Forms\Components\DatePicker::make('birth_date'),
    ]);

Section::make('Address')
    ->relationship('address')
    ->schema([/* address fields */]);
```

### Relation Groups (Tabs)

```php
use Filament\Resources\RelationManagers\RelationGroup;

public static function getRelations(): array
{
    return [
        RelationGroup::make('Contacts', [
            RelationManagers\EmailsRelationManager::class,
            RelationManagers\PhonesRelationManager::class,
        ]),
    ];
}

// With badges and icons
RelationGroup::make('Contacts', [/* ... */])
    ->tab(fn (Model $owner): Tab => Tab::make('Contacts')
        ->badge($owner->contacts()->count())
        ->badgeColor('info')
        ->icon('heroicon-m-document-text'));
```

### Deferred Badges

```php
protected static bool $isBadgeDeferred = true;

// Or dynamic
public static function isBadgeDeferred(Model $ownerRecord, string $pageClass): bool
{
    return FeatureFlag::active();
}
```

### Sharing Resource Form/Table

```php
public function form(Schema $schema): Schema
{
    return PostResource::form($schema);
}

public function table(Table $table): Table
{
    return PostResource::table($table);
}
```

### Passing Properties

```php
// In resource
public static function getRelations(): array
{
    return [CommentsRelationManager::make(['status' => 'approved'])];
}

// In relation manager
class CommentsRelationManager extends RelationManager
{
    public string $status;
    // Access via $this->status
}
```

### Disabling Lazy Loading

```php
protected static bool $isLazy = false; // Loads all managers upfront
```

### Nested Resources (`--nested` flag)

```bash
php artisan make:filament-resource Lesson --nested
php artisan make:filament-relation-manager CourseResource lessons title
```

```php
// Link nested resource to parent
public static function getParentResourceRegistration(): ?ParentResourceRegistration
{
    return CourseResource::asParent()
        ->relationship('lessons')
        ->inverseRelationship('course');
}

// Register with key for correct URLs
public static function getRelations(): array
{
    return ['lessons' => LessonsRelationManager::class];
}
```

### Singular Resources

For single-record entities (e.g., "Homepage" settings):

```php
class ManageHomepage extends Page
{
    protected static string $view = 'filament.pages.manage-homepage';
    public ?array $data = [];

    public function mount(): void
    {
        $record = $this->getRecord();
        if ($record) $this->form->fill($record->toArray());
    }

    public function form(Form $form): Form
    {
        return $form
            ->schema([TextInput::make('title')->required()])
            ->statePath('data');
    }

    public function save(): void
    {
        $data = $this->form->getState();
        $record = $this->getRecord() ?? new WebsitePage(['is_homepage' => true]);
        $record->fill($data)->save();

        if ($record->wasRecentlyCreated) {
            $this->form->record($record)->saveRelationships();
        }

        Notification::make()->success()->title('Saved')->send();
    }

    public function getRecord(): ?WebsitePage
    {
        return WebsitePage::query()->where('is_homepage', true)->first();
    }
}
```

```blade
<x-filament::page>
    {{ $this->form }}
</x-filament::page>
```

---

## 6. Plugin Development

### Plugin Interface

```php
use Filament\Contracts\Plugin;
use Filament\Panel;

class BlogPlugin implements Plugin
{
    public function getId(): string
    {
        return 'blog';  // Must be unique
    }

    public function register(Panel $panel): void
    {
        $panel
            ->resources([PostResource::class, CategoryResource::class])
            ->pages([Settings::class]);
    }

    public function boot(Panel $panel): void
    {
        // Runs only when panel is in-use (via middleware)
    }

    public static function make(): static
    {
        return app(static::class);
    }
}
```

### Fluent Configuration

```php
class BlogPlugin implements Plugin
{
    protected bool $hasAuthorResource = false;

    public function authorResource(bool $condition = true): static
    {
        $this->hasAuthorResource = $condition;
        return $this;
    }

    public function hasAuthorResource(): bool
    {
        return $this->hasAuthorResource;
    }

    public function register(Panel $panel): void
    {
        if ($this->hasAuthorResource()) {
            $panel->resources([AuthorResource::class]);
        }
    }
}

// Usage:
BlogPlugin::make()->authorResource()
```

### Using a Plugin

```php
use DanHarrin\FilamentBlog\BlogPlugin;

public function panel(Panel $panel): Panel
{
    return $panel->plugin(BlogPlugin::make());
}
```

### Accessing Configuration

```php
// By plugin ID
filament('blog')->hasAuthorResource();

// Type-safe
BlogPlugin::get()->hasAuthorResource();
```

### Service Providers

```php
use Spatie\LaravelPackageTools\Package;
use Spatie\LaravelPackageTools\PackageServiceProvider;

class MyPluginServiceProvider extends PackageServiceProvider
{
    public static string $name = 'my-plugin';

    public function configurePackage(Package $package): void
    {
        $package->name(static::$name);
    }

    public function packageBooted(): void
    {
        Livewire::component('clock-widget', ClockWidget::class);

        FilamentAsset::register(
            assets: [AlpineComponent::make('clock-widget', __DIR__ . '/../resources/dist/clock-widget.js')],
            package: 'awcodes/clock-widget'
        );
    }
}
```

### Asset Management

> **Don't bundle CSS in plugins.** Use custom Filament themes for styling.

```php
use Filament\Support\Facades\FilamentAsset;

FilamentAsset::register(
    assets: [AlpineComponent::make('my-component', __DIR__ . '/../resources/dist/my-component.js')],
    package: 'vendor/package-name'
);
```

```blade
<div x-load
     x-load-src="{{ \Filament\Support\Facades\FilamentAsset::getAlpineComponentSrc('clock-widget', 'awcodes/clock-widget') }}"
     x-data="clockWidget()">
    <p x-text="time"></p>
</div>
```

---

## 7. Global Search

### Basic Setup

```php
// Required: set title attribute
protected static ?string $recordTitleAttribute = 'title';

// Resource must have Edit or View page for links to work
```

### Custom Titles & Details

```php
use Illuminate\Contracts\Support\Htmlable;

public static function getGlobalSearchResultTitle(Model $record): string|Htmlable
{
    return $record->name;
}

public static function getGlobalSearchResultDetails(Model $record): array
{
    return [
        'Category' => $record->category->name,
        'Author' => $record->author->name,
    ];
}
```

### Custom URLs

```php
public static function getGlobalSearchResultUrl(Model $record): ?string
{
    return static::getUrl('view', ['record' => $record]);
}
```

### Custom Actions

```php
use Filament\Actions\Action;

public static function getGlobalSearchResultActions(Model $record): array
{
    return [
        Action::make('edit')
            ->url(static::getUrl('edit', ['record' => $record])),
        Action::make('view')
            ->url(static::getUrl('view', ['record' => $record]), shouldOpenInNewTab: true),
        Action::make('quickView')
            ->dispatch('quickView', [$record->id]),
    ];
}
```

### Result Limiting & Sorting

```php
protected static int $globalSearchResultsLimit = 20;     // Default: 50
protected static ?string $globalSearchResultsSortColumn = 'created_at';
protected static ?string $globalSearchResultsSortDirection = 'desc';
```

### Position & Key Bindings

```php
use Filament\Enums\GlobalSearchPosition;

$panel
    ->globalSearch(position: GlobalSearchPosition::Sidebar)  // Sidebar | TopBar
    ->globalSearchKeyBindings(['command+k', 'ctrl+k'])
    ->globalSearchDebounce(500);  // milliseconds
```

### Disabling Search

```php
// Per resource
protected static bool $isGloballySearchable = false;

// Panel-level opt-in
$panel->globalSearch(false);
```

### Search Term Splitting

```php
protected static ?bool $shouldSplitGlobalSearchTerms = false; // Improves performance on large datasets
```

---

## 8. Code Quality Tips

### Fluent Interface Patterns

```php
// Everything chains
TextInput::make('name')
    ->required()
    ->maxLength(255)
    ->hint('Your full name')
    ->columnSpan(2);

// Panel configuration is one fluent chain
$panel
    ->id('admin')
    ->path('admin')
    ->spa()
    ->databaseTransactions()
    ->middleware([Custom::class]);
```

### DDD Modular Architecture

```
app/Filament/
  Blog/
    Resources/PostResource.php
    Resources/CategoryResource.php
    Pages/
    Widgets/
  Shop/
    Resources/ProductResource.php
    Resources/OrderResource.php
  Admin/
    Resources/UserResource.php
```

```php
// Configure discovery
$panel
    ->discoverResources(in: app_path('Filament/Blog'), for: 'App\\Filament\\Blog')
    ->discoverResources(in: app_path('Filament/Shop'), for: 'App\\Filament\\Shop')
    ->discoverResources(in: app_path('Filament/Admin'), for: 'App\\Filament\\Admin');
```

### Resource Organization

```
app/Filament/Resources/
  PostResource.php
  PostResource/
    Pages/
      ListPosts.php
      CreatePost.php
      EditPost.php
      ViewPost.php
    RelationManagers/
      CommentsRelationManager.php
      TagsRelationManager.php
    Widgets/
      PostStatsWidget.php
```

### Enum Usage

```php
use Filament\Support\Enums\Width;
use Filament\Support\Enums\Alignment;
use Filament\Pages\Enums\SubNavigationPosition;
use Filament\Enums\ThemeMode;
use Filament\Enums\GlobalSearchPosition;

$panel->maxContentWidth(Width::Full);
$panel->subNavigationPosition(SubNavigationPosition::End);
$panel->defaultThemeMode(ThemeMode::System);
```

### Contracts Pattern

```php
class User extends Model implements 
    FilamentUser,              // Can access panels
    HasDefaultTenant,          // Default tenant
    HasTenants,                // Multiple tenants
    HasName,                   // Custom display name
    HasAvatar,                 // Custom avatar
    HasCurrentTenantLabel      // Custom tenant label
{
}
```

### Common Gotchas

| # | Gotcha | Solution |
|---|--------|----------|
| 1 | `shouldRegisterNavigation` only hides nav | Use **Laravel Policies** for access control |
| 2 | Wildcard routes match before hard-coded | Define hard-coded routes **before** wildcards |
| 3 | Pivot attributes not saving | Must be in `withPivot()` on **both** sides |
| 4 | Duplicate attachments fail | Require `id` column on pivot + `withPivot('id')` |
| 5 | Tailwind classes not working | Must create **custom theme** via `make:filament-theme` |
| 6 | SPA breaks external links | Use `spaUrlExceptions(['/external'])` |
| 7 | Tenant data leaking | Global scopes via **persistent middleware**, not just resources |
| 8 | Plugin CSS not applying | Don't bundle CSS; recommend custom themes |
| 9 | Echo auto-connecting | Disable with `$panel->broadcasting(false)` if unused |
| 10 | All relation managers loading | Default is lazy; only set `$isLazy = false` when needed |

### Best Practices Checklist

- [ ] **Validate** all input with form rules
- [ ] **Authorize** with Laravel Policies per-resource
- [ ] **Lazy load** relation managers (keep default `$isLazy = true`)
- [ ] **Defer badges** (`$isBadgeDeferred`) for expensive count queries
- [ ] **Chunk bulk actions** (`chunkSelectedRecords(250)`) to reduce memory
- [ ] **Scope queries** to tenant/user context
- [ ] **Use contracts** (`HasName`, `HasAvatar`, etc.)
- [ ] **Custom themes** via Vite (not inline styles)
- [ ] **Register assets** per-panel, not globally
- [ ] **Enable transactions** for data integrity; disable when unnecessary
- [ ] **Persistent middleware** for AJAX-surviving operations
- [ ] **Error notifications** configured per-panel and per-page

---

## Cheat Sheet: Common Artisan Commands

```bash
# Panels & Resources
php artisan make:filament-panel app
php artisan make:filament-resource Post
php artisan make:filament-resource Lesson --nested
php artisan make:filament-page SortUsers --resource=UserResource --type=custom
php artisan make:filament-widget CustomerOverview --resource=CustomerResource

# Relationships
php artisan make:filament-relation-manager CategoryResource posts title
php artisan make:filament-relation-manager CategoryResource posts title --soft-deletes
php artisan make:filament-page ManageCourseLessons --resource=CourseResource --type=ManageRelatedRecords

# Theming & Assets
php artisan make:filament-theme
php artisan filament:assets

# Cache / Clear
php artisan filament:optimize
php artisan filament:optimize-clear
```
