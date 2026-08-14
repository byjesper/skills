# Filament 5 - Infolists

> **Namespace**: `Filament\Infolists\Components`

---


Read-only display of record data using Entry components:

```php
use Filament\Infolists\Components\TextEntry;
use Filament\Infolists\Components\IconEntry;
use Filament\Infolists\Components\ImageEntry;
use Filament\Infolists\Infolist;

public function infolist(Infolist $infolist): Infolist
{
    return $infolist->schema([
        TextEntry::make('name'),
        TextEntry::make('email')->icon('heroicon-m-envelope'),
        IconEntry::make('is_active')->boolean(),
        ImageEntry::make('avatar')->circular()->size(80),
        TextEntry::make('bio')->markdown()->columnSpanFull(),
    ]);
}
```

Entry types: `TextEntry`, `IconEntry`, `ImageEntry`, `ColorEntry`, `KeyValueEntry`, `RepeatableEntry`, `CodeEntry`.

