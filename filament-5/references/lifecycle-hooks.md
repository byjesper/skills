# Filament 5 - Lifecycle Hooks

> **Namespace**: `Filament\Resources\Pages`

---


## Create Page

```php
use Filament\Support\Enums\Operation;

// Before/after fill
public function beforeFill(): void { }
public function afterFill(): void { }

// Before/after validation
public function beforeValidate(): void { }
public function afterValidate(): void { }

// Before/after create
public function beforeCreate(): void { }
public function afterCreate(): void { }

// Mutate form data
protected function mutateFormDataBeforeCreate(array $data): array { return $data; }

// Custom creation logic
protected function handleRecordCreation(array $data): Model { return static::getModel()::create($data); }

// Redirect after create
protected function getRedirectUrl(): string { return $this->getResource()::getUrl('index'); }
```

## Edit Page

```php
protected function mutateFormDataBeforeFill(array $data): array { return $data; }
protected function mutateFormDataBeforeSave(array $data): array { return $data; }
protected function handleRecordUpdate(Model $record, array $data): Model { $record->update($data); return $record; }
protected function getRedirectUrl(): string { return $this->getResource()::getUrl('index'); }
```

## Table Actions

```php
protected function before(Action $action): void { }
protected function after(Action $action): void { }
```

