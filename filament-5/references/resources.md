# Filament 5 - Resources

> **Namespace**: `Filament\Resources`

---


## Creating a Resource

```bash
php artisan make:filament-resource Customer
php artisan make:filament-resource Customer --view       # Add View page
php artisan make:filament-resource Customer --simple     # Modal-only manage page
php artisan make:filament-resource Customer --soft-deletes
```

## Resource Class

```php
use App\Models\Customer;
use Filament\Resources\Resource;

class CustomerResource extends Resource
{
    protected static ?string $model = Customer::class;
    protected static ?string $recordTitleAttribute = 'name';
    protected static ?string $navigationIcon = 'heroicon-o-user-group';
    protected static ?int $navigationSort = 2;
    protected static string|UnitEnum|null $navigationGroup = 'Shop';
    protected static ?string $navigationParentItem = 'Products';
    protected static ?string $slug = 'customers';

    public static function form(Schema $schema): Schema { ... }
    public static function table(Table $table): Table { ... }
    public static function getPages(): array { ... }
    public static function getEloquentQuery(): Builder { ... }
    public static function getRelations(): array { ... }
}
```

## Eager Loading (Prevent N+1 Queries)

Always eager load relationships in `getEloquentQuery()` to avoid N+1 query problems:

```php
use Illuminate\Database\Eloquent\Builder;

class CustomerResource extends Resource
{
    // ...

    public static function getEloquentQuery(): Builder
    {
        return parent::getEloquentQuery()
            ->with(['category', 'author', 'tags']); // Eager load relationships
    }
}
```

For widgets that display related data, eager load in the query builder:

```php
// Table widget - eager load relationships
public function table(Table $table): Table
{
    return $table
        ->query(Order::query()->with(['customer', 'items'])->latest()->limit(10))
        ->columns([...]);
}

// Stats widget - use aggregate queries, not loops
protected function getStats(): array
{
    return [
        Stat::make('Orders', Order::count()),                           // Single query
        Stat::make('Revenue', Order::sum('total')),                     // Aggregate
        Stat::make('Pending', Order::where('status', 'pending')->count()), // Filtered count
    ];
}
```

## Page Registration

```php
public static function getPages(): array
{
    return [
        'index' => Pages\ListCustomers::route('/'),
        'create' => Pages\CreateCustomer::route('/create'),
        'view' => Pages\ViewCustomer::route('/{record}'),       // Always include View
        'edit' => Pages\EditCustomer::route('/{record}/edit'),
    ];
}
```

## View Page

Always include a View page for resources unless there's a specific reason not to. It provides a read-only detail view and is required for global search result links.

```bash
php artisan make:filament-resource Customer --view
```

```php
// app/Filament/Resources/CustomerResource/Pages/ViewCustomer.php
namespace App\Filament\Resources\CustomerResource\Pages;

use App\Filament\Resources\CustomerResource;
use Filament\Actions\EditAction;
use Filament\Resources\Pages\ViewRecord;

class ViewCustomer extends ViewRecord
{
    protected static string $resource = CustomerResource::class;

    protected function getHeaderActions(): array
    {
        return [
            EditAction::make(),
        ];
    }
}
```

The View page automatically uses the resource's `infolist()` method (see Infolists section below) to display record data in read-only format. If no `infolist()` is defined, it falls back to a basic text display.

## Authorization

Filament uses Laravel model policies automatically. Define `viewAny`, `view`, `create`, `update`, `delete`, `deleteAny`, `forceDelete`, `restore` methods on your policy.

## Complete Resource Example

Here's a complete Product resource demonstrating View page, eager loading, file upload security, policy, and infolist:

```php
// app/Filament/Resources/ProductResource.php
namespace App\Filament\Resources;

use App\Models\Product;
use Filament\Forms\Components\FileUpload;
use Filament\Forms\Components\Select;
use Filament\Forms\Components\TextInput;
use Filament\Forms\Components\Toggle;
use Filament\Infolists\Components\ImageEntry;
use Filament\Infolists\Components\TextEntry;
use Filament\Infolists\Infolist;
use Filament\Resources\Resource;
use Filament\Schemas\Components\Section;
use Filament\Schemas\Schema;
use Filament\Tables\Columns\TextColumn;
use Filament\Tables\Filters\SelectFilter;
use Filament\Tables\Table;
use Illuminate\Database\Eloquent\Builder;

class ProductResource extends Resource
{
    protected static ?string $model = Product::class;
    protected static ?string $recordTitleAttribute = 'name';
    protected static ?string $navigationIcon = 'heroicon-o-shopping-bag';
    protected static ?string $navigationGroup = 'Shop';

    public static function form(Schema $schema): Schema
    {
        return $schema->components([
            Section::make('Details')->schema([
                TextInput::make('name')->required()->maxLength(255),
                TextInput::make('price')->numeric()->prefix('$')->required(),
                Select::make('category_id')
                    ->relationship('category', 'name')
                    ->searchable()
                    ->preload()
                    ->required(),
                FileUpload::make('image')
                    ->image()
                    ->maxSize(2048)
                    ->acceptedFileTypes(['image/jpeg', 'image/png', 'image/webp'])
                    ->preventFilePathTampering()
                    ->directory('products'),
                Toggle::make('is_active')->default(true),
            ]),
        ]);
    }

    public static function table(Table $table): Table
    {
        return $table
            ->columns([
                TextColumn::make('id')->sortable()->copyable(),
                TextColumn::make('name')->searchable()->sortable(),
                TextColumn::make('category.name')->sortable(),
                TextColumn::make('price')->money('USD')->sortable(),
                TextColumn::make('created_at')->since()->dateTooltip(),
            ])
            ->filters([
                SelectFilter::make('category')
                    ->relationship('category', 'name')
                    ->searchable()
                    ->preload(),
            ])
            ->defaultSort('created_at', 'desc');
    }

    public static function infolist(Infolist $infolist): Infolist
    {
        return $infolist->schema([
            TextEntry::make('name'),
            TextEntry::make('price')->money('USD'),
            TextEntry::make('category.name'),
            ImageEntry::make('image')->size(200),
            TextEntry::make('created_at')->dateTime(),
        ]);
    }

    public static function getEloquentQuery(): Builder
    {
        return parent::getEloquentQuery()->with('category');
    }

    public static function getPages(): array
    {
        return [
            'index' => Pages\ListProducts::route('/'),
            'create' => Pages\CreateProduct::route('/create'),
            'view' => Pages\ViewProduct::route('/{record}'),
            'edit' => Pages\EditProduct::route('/{record}/edit'),
        ];
    }
}
```

```php
// app/Policies/ProductPolicy.php
namespace App\Policies;

use App\Models\Product;
use App\Models\User;

class ProductPolicy
{
    public function viewAny(User $user): bool { return true; }
    public function view(User $user, Product $product): bool { return true; }
    public function create(User $user): bool { return $user->isAdmin(); }
    public function update(User $user, Product $product): bool { return $user->isAdmin(); }
    public function delete(User $user, Product $product): bool { return $user->isAdmin(); }
    public function deleteAny(User $user): bool { return $user->isAdmin(); }
    public function forceDelete(User $user, Product $product): bool { return $user->isAdmin(); }
    public function restore(User $user, Product $product): bool { return $user->isAdmin(); }
}
```
