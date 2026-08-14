# Filament 5 - Relationships

> **Namespace**: `Filament\Resources\RelationManagers`

---


## Choosing the Right Tool

| Tool | Relationship Type | Use Case |
|------|-------------------|----------|
| RelationManagers | HasMany, BelongsToMany, MorphMany | Full CRUD table below form |
| Select/CheckboxList | BelongsTo, MorphTo, BelongsToMany | Choose existing records |
| Repeater | HasMany, MorphMany | Inline CRUD (few fields) |
| Layout components | BelongsTo, HasOne, MorphOne | Single related record fields |

## Relation Manager

```bash
php artisan make:filament-relation-manager CategoryResource posts title
```

```php
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

Register in resource: `public static function getRelations(): array { return [RelationManagers\PostsRelationManager::class]; }`

## Select with Relationship

```php
Select::make('category_id')
    ->relationship('category', 'name')
    ->searchable()
    ->preload()
    ->createOptionForm([...])   // Inline create new option
    ->editOptionForm([...]);     // Inline edit existing option
```

## Repeater for Related Records

```php
Repeater::make('items')
    ->relationship('items')
    ->schema([
        TextInput::make('name')->required(),
        TextInput::make('quantity')->numeric()->required(),
    ])
    ->collapsible()
    ->itemLabel(fn (array $state): ?string => $state['name'] ?? null)
    ->minItems(1)
    ->maxItems(10);
```

