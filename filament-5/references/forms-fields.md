# Filament 5 - Form Fields & Layout Components Quick Reference

> **Namespace**: `Filament\Forms\Components` (fields) | `Filament\Schemas\Components` (layout)

---

## Table of Contents

- [TextInput](#textinput)
- [Select](#select)
- [Checkbox](#checkbox)
- [Toggle](#toggle)
- [CheckboxList](#checkboxlist)
- [Radio](#radio)
- [DateTimePicker](#datetimepicker)
- [FileUpload](#fileupload)
- [RichEditor](#richeditor)
- [Textarea](#textarea)
- [KeyValue](#keyvalue)
- [ColorPicker](#colorpicker)
- [Hidden](#hidden)
- [Repeater](#repeater)
- [Builder](#builder)
- [TagsInput](#tagsinput)
- [ToggleButtons](#togglebuttons)
- [Slider](#slider)
- [CodeEditor](#codeeditor)
- [Layout Components](#layout-components)
  - [Grid](#grid)
  - [Section](#section)
  - [Tabs](#tabs)
  - [Wizard](#wizard)
  - [Fieldset](#fieldset)
  - [Flex](#flex)
- [Validation](#validation)
- [Utility Injection](#utility-injection)
- [Conditional Visibility](#conditional-visibility)

---

## TextInput

```php
use Filament\Forms\Components\TextInput;
use Filament\Support\Icons\Heroicon;
use Filament\Support\RawJs;
use Filament\Actions\Action;
```

### Input Types

| Method | Description |
|--------|-------------|
| `->email()` | Type="email" + email validation |
| `->numeric()` | Type="number" + numeric validation |
| `->integer()` | Type="number" + integer validation |
| `->password()` | Type="password" |
| `->tel()` | Type="tel" + phone validation |
| `->url()` | Type="url" + URL validation |
| `->type('color')` | Any HTML input type |

```php
TextInput::make('name')
    // Types
    ->email()
    ->numeric()->step(100)
    ->integer()
    ->password()->revealable()
    ->tel()
    ->url()
    ->type('color')

    // Input mode (mobile keyboards)
    ->inputMode('decimal')    // none, text, tel, url, email, numeric, decimal, search

    // Autocomplete
    ->autocomplete('new-password')
    ->autocomplete(false)

    // Datalist (suggestions, not strict)
    ->datalist(['BMW', 'Ford', 'Mercedes-Benz', 'Porsche', 'Toyota'])

    // Autocapitalize
    ->autocapitalize('words')  // none, off, sentences, on, words, characters

    // Affixes
    ->prefix('https://')
    ->suffix('.com')
    ->prefixIcon(Heroicon::Envelope)
    ->suffixIcon(Heroicon::CurrencyDollar)
    ->prefixIconColor('primary')
    ->suffixIconColor('success')
    ->suffixAction(Action::make('copy')->icon(Heroicon::Clipboard))

    // Password reveal toggle
    ->password()->revealable()

    // Copy to clipboard
    ->copyable(copyMessage: 'Copied!', copyMessageDuration: 1500)
    // Note: copyable() only works when SSL is enabled

    // Input masking (Alpine.js)
    ->mask('99/99/9999')
    ->mask(RawJs::make(<<<'JS'
        $input.startsWith('34') || $input.startsWith('37')
            ? '9999 999999 99999'
            : '9999 9999 9999 9999'
    JS))
    ->mask(RawJs::make('$money($input)'))
    ->stripCharacters(',')

    // Trimming
    ->trim()  // Trims whitespace from start/end

    // Read-only (submitted but not editable)
    ->readOnly()

    // Length validation
    ->maxLength(255)
    ->minLength(2)
    ->length(10)

    // General field methods
    ->label('Full Name')
    ->hiddenLabel()              // Hide label visually (a11y-safe)
    ->default('John')
    ->disabled()
    ->disabledOn(['edit', 'view'])
    ->hidden()
    ->hiddenOn('create')
    ->placeholder('Enter name...')
    ->hint('This is a hint')
    ->helperText('Helpful context')
    ->extraAttributes(['class' => 'custom-class'])
    ->required()
    ->markAsRequired(false)       // Remove asterisk visually
    ->inlineLabel()
    ->autofocus()
    ->live()                      // Reactive (triggers server round-trip)
    ->live(onBlur: true)          // Reactive only on blur
    ->suffixAction(
        Action::make('generate')
            ->icon(Heroicon::Sparkles)
            ->action(function (Set $set) {
                $set('name', fake()->name());
            })
    );
```

### Global Defaults (Service Provider)

```php
use Filament\Forms\Components\TextInput;

TextInput::configureUsing(function (TextInput $component): void {
    $component->trim();
});
```

---

## Select

```php
use Filament\Forms\Components\Select;
use Filament\Forms\Components\MorphToSelect;
use Filament\Forms\Components\TextInput;
use Illuminate\Database\Eloquent\Builder;
```

### Basic Options

```php
Select::make('status')
    ->options([
        'draft' => 'Draft',
        'reviewing' => 'Reviewing',
        'published' => 'Published',
    ])
    ->native(false)               // Use JavaScript select (required for searchable)
    ->searchable()
    ->searchDebounce(1000)        // Default: 1000ms
    ->preload()                   // Preload options (avoid for large datasets)
    ->loadingMessage('Loading...')
    ->noSearchResultsMessage('No results found.')
    ->noOptionsMessage('No options available.')
    ->searchPrompt('Search for an option...')
    ->searchingMessage('Searching...')
;
```

### Custom Search

```php
Select::make('author_id')
    ->searchable()
    ->getSearchResultsUsing(fn (string $search): array =>
        User::query()->where('name', 'like', "%{$search}%")
            ->limit(50)->pluck('name', 'id')->all()
    )
    ->getOptionLabelUsing(fn ($value): ?string => User::find($value)?->name)
;
```

### Multi-Select

```php
Select::make('technologies')
    ->multiple()
    ->options([...])
    ->reorderable()               // Allow reordering selected options
;
```

### Grouped Options

```php
Select::make('status')
    ->options([
        'In Process' => [
            'draft' => 'Draft',
            'reviewing' => 'Reviewing',
        ],
        'Reviewed' => [
            'published' => 'Published',
            'rejected' => 'Rejected',
        ],
    ])
    ->grouped()
;
```

### Eloquent Relationship (BelongsTo)

```php
Select::make('author_id')
    ->relationship(name: 'author', titleAttribute: 'name')
    ->relationship(
        name: 'author',
        titleAttribute: 'name',
        modifyQueryUsing: fn (Builder $query) => $query->where('active', true)
    )
    ->searchable(['name', 'email'])   // Search across multiple columns
    ->preload()
    // Exclude current record (self-referencing relationships)
    ->optionsLimit(50)
;
```

### Eloquent Relationship (BelongsToMany)

```php
Select::make('technologies')
    ->multiple()
    ->relationship(titleAttribute: 'name')
    // IMPORTANT: call disabled() BEFORE relationship()
    ->disabled()
    ->relationship(titleAttribute: 'name')
    // Save extra pivot data
    ->pivotData(['is_primary' => true])
;
```

### Custom Option Labels from Relationship

```php
Select::make('author_id')
    ->relationship(name: 'author', titleAttribute: 'name')
    ->getOptionLabelFromRecordUsing(fn (User $record): string => "{$record->name} ({$record->email})")
;
```

### Create New Option in Modal

```php
Select::make('author_id')
    ->relationship(name: 'author', titleAttribute: 'name')
    ->createOptionForm([
        TextInput::make('name')->required(),
        TextInput::make('email')->required()->email(),
    ])
    // Custom creation logic
    ->createOptionUsing(function (array $data): int {
        return auth()->user()->team->members()->create($data)->getKey();
    })
    // Customize the create action button
    ->createOptionAction(fn (Action $action) => $action->modalWidth('md'))
;
```

### Edit Selected Option in Modal

```php
Select::make('author_id')
    ->relationship(name: 'author', titleAttribute: 'name')
    ->editOptionForm([
        TextInput::make('name')->required(),
    ])
    ->editOptionUsing(function (array $data, Model $record): void {
        $record->update($data);
    })
;
```

### MorphTo Relationship

```php
use Filament\Forms\Components\MorphToSelect;

MorphToSelect::make('commentable')
    ->types([
        MorphToSelect\Type::make(Post::class)->titleAttribute('title'),
        MorphToSelect\Type::make(Product::class)
            ->titleAttribute('name')
            ->modifyOptionsQueryUsing(fn (Builder $query) => $query->whereBelongsTo($this->team))
            ->getOptionLabelFromRecordUsing(fn (Product $record): string => "{$record->name} - {$record->slug}"),
    ])
    ->searchable()
    ->preload()
    // Customize the type selector
    ->modifyTypeSelectUsing(fn (Select $select) => $select->native(false))
    // Customize the key select for all types
    ->modifyKeySelectUsing(fn (Select $select) => $select->native())
;
```

### Other Select Options

```php
Select::make('status')
    ->allowHtml()                 // Allow HTML in option labels
    ->wrap()                      // Wrap option labels
    ->truncate()                  // Truncate long option labels
    ->selectablePlaceholder(false) // Disable placeholder
    ->disableOptionWhen(fn (string $value): bool => $value === 'published')
    ->prefix('€')
    ->suffixIcon(Heroicon::Check)
    ->suffixIconColor('success')
    ->loadingMessage('Loading authors...')
    ->required()
    ->exists(table: User::class, column: 'id')
    ->unique(table: User::class, column: 'email', ignoreRecord: true)
    ->scopedUnique()              // Uses Eloquent model (multi-tenant aware)
    ->in(['draft', 'published'])
    ->notIn(['archived'])
;
```

---

## Checkbox

```php
use Filament\Forms\Components\Checkbox;

Checkbox::make('is_admin')
    ->label('Administrator')
    ->inline(false)               // Place label below checkbox
    ->required()
    ->accepted()                  // Must be checked (for terms acceptance)
;
```

---

## Toggle

```php
use Filament\Forms\Components\Toggle;
use Filament\Support\Icons\Heroicon;

Toggle::make('is_active')
    ->label('Active')
    ->inline(false)               // Stacked layout (label above)
    ->onIcon(Heroicon::Bolt)
    ->offIcon(Heroicon::User)
    ->onColor('success')
    ->offColor('danger')
    ->required()
    ->accepted()                  // Must be ON
    ->declined()                  // Must be OFF
    ->default(true)
;
```

---

## CheckboxList

```php
use Filament\Forms\Components\CheckboxList;

CheckboxList::make('technologies')
    ->options([
        'tailwind' => 'Tailwind CSS',
        'alpine' => 'Alpine.js',
        'laravel' => 'Laravel',
        'livewire' => 'Livewire',
    ])
    ->descriptions([              // Add descriptions per option
        'tailwind' => 'A utility-first CSS framework.',
        'alpine' => 'A lightweight JavaScript framework.',
    ])
    ->columns(2)                  // Display in columns
    ->columns(['sm' => 1, 'md' => 2, 'lg' => 3])  // Responsive
    ->gridDirection('row')        // or 'column'
    ->bulkToggleable()            // Enable select all / select none
    ->searchable()                // Search through options
    ->noSearchResultsMessage('No technologies found.')
    ->relationship('technologies', 'name')
    ->required()
;
```

---

## Radio

```php
use Filament\Forms\Components\Radio;

Radio::make('status')
    ->options([
        'draft' => 'Draft',
        'published' => 'Published',
    ])
    ->descriptions([
        'draft' => 'Not visible to the public.',
        'published' => 'Visible to the public.',
    ])
    ->inline()                    // Inline option layout
    ->inlineLabel(false)          // Stacked label
    ->default('draft')
;
```

---

## DateTimePicker

```php
use Filament\Forms\Components\DateTimePicker;

DateTimePicker::make('published_at')
    ->label('Publish Date')
    ->native(false)               // Use JavaScript picker (Flatpickr)
    ->displayFormat('M d, Y H:i') // Display format
    ->format('Y-m-d H:i:s')       // Storage format
    ->seconds(false)              // Hide seconds selector
    ->minDate(now())              // Minimum date
    ->maxDate(now()->addYear())   // Maximum date
    ->weekStartsOnSunday()        // Sunday as first day
    ->timezone('America/New_York')// Display timezone
    ->hintIcon(Heroicon::Clock)
    ->required()
;
```

### DatePicker (alias)

```php
use Filament\Forms\Components\DatePicker;

DatePicker::make('birthday')
    ->native(false)
    ->displayFormat('M d, Y')
    ->minDate(now()->subYears(120))
    ->maxDate(now()->subYears(18))
;
```

### TimePicker (alias)

```php
use Filament\Forms\Components\TimePicker;

TimePicker::make('start_time')
    ->seconds(false)
;
```

---

## FileUpload

```php
use Filament\Forms\Components\FileUpload;
use Livewire\Features\SupportFileUploads\TemporaryUploadedFile;
```

### Storage Configuration

```php
FileUpload::make('attachment')
    ->disk('s3')                          // Storage disk
    ->directory('form-attachments')        // Storage directory
    ->visibility('public')                 // 'private' or 'public'
;
```

### Multiple Files

```php
FileUpload::make('attachments')
    ->multiple()
    ->maxFiles(5)
    ->maxParallelUploads(2)
    ->reorderable()
    ->panelLayout('grid')                 // 'grid' or 'list'
    ->appendFiles()                       // Move instead of copy on submit
    ->storeFiles(false)                   // Don't store until form submitted
;
```

### Image Processing

```php
FileUpload::make('image')
    ->image()                              // Only accept images
    ->imageEditor()                        // Enable image editor (Cropper.js)
    ->circleCropper()                      // Allow circle crop
    ->imageEditorAspectRatioOptions([null, '16:9', '4:3', '1:1']) // Free + fixed
    ->imageEditorMode(2)                   // 1, 2, or 3 (Cropper.js viewMode)
    ->imageEditorEmptyFillColor('#000000')
    ->imageEditorViewportWidth('1920')
    ->imageEditorViewportHeight('1080')
    // Auto-resize without editor
    ->imageResizeMode('cover')            // 'cover', 'contain'
    ->imageResizeTargetWidth('1920')
    ->imageResizeTargetHeight('1080')
    ->imageCropAspectRatio('16:9')
    ->orientImagesFromExif()              // Auto-orient from EXIF data
;
```

### File Naming

```php
FileUpload::make('attachment')
    // SECURITY WARNING: preserveFilenames() on local/public disks = RCE risk
    // via PHP files with deceptive mime types. Use only with S3 or review carefully.
    ->preserveFilenames()

    // Generate custom filenames
    ->getUploadedFileNameForStorageUsing(
        fn (TemporaryUploadedFile $file): string =>
            (string) str($file->getClientOriginalName())->prepend('custom-prefix-')
    )

    // Store original filename in a separate DB column
    ->storeFileNamesIn('original_filename')
;
```

### Security

```php
FileUpload::make('attachment')
    ->preventFilePathTampering()           // Prevent path tampering attacks
    ->preventFilePathTampering(
        allowFilePathUsing: fn (string $file): bool => str_starts_with($file, 'templates/')
    )
    ->acceptedFileTypes(['image/jpeg', 'image/png', 'application/pdf'])
    ->maxSize(2048)                        // Max size in KB
    ->minSize(10)                          // Min size in KB
    ->validationMessages([
        'tampered' => 'The selected attachment is not permitted.',
    ])
;
```

### File Types & Size

```php
FileUpload::make('document')
    ->acceptedFileTypes(['image/jpeg', 'image/png', 'application/pdf'])
    ->image()                              // Must be an image
    ->maxSize(2048)                        // KB
    ->minSize(10)
    ->dimensions(
        minWidth: 500,
        maxWidth: 2000,
        minHeight: 500,
        maxHeight: 2000,
    )
    ->mimes(['jpg', 'png'])
    ->mimetypes(['image/jpeg', 'image/png'])
;
```

### Avatar Mode

```php
FileUpload::make('avatar')
    ->avatar()                             // Compact circle layout
    ->image()
    ->circleCropper()
    ->imageEditor()
;
```

### UI Customization

```php
FileUpload::make('attachment')
    ->downloadable()
    ->openable()
    ->previewable()
    ->removeUploadedFileButtonPosition('right')
    ->uploadProgressIndicatorPosition('left')
    ->uploadButtonPosition('left')
;
```

### CRITICAL Security Warnings

1. **`preserveFilenames()` with `local`/`public` disks = RCE vulnerability** - PHP files with deceptive mime types can be executed
2. **Use `storeFileNamesIn()`** to store original names in DB while keeping random filesystem names
3. **S3 disk protects against RCE** (S3 won't execute PHP)
4. **`acceptedFileTypes()`** uses Laravel's `mimetypes` rule (validates mime type, NOT file extension)
5. **`preventFilePathTampering()`** is recommended for all production apps

---

## RichEditor

```php
use Filament\Forms\Components\RichEditor;
use Filament\Forms\Components\CustomBlock;
use Filament\Forms\Components\Select;
use Filament\Forms\Components\TextInput;
use Filament\Actions\Action;
use Filament\Support\Icons\Heroicon;
```

### Storage Format

```php
RichEditor::make('content')
    ->json()                               // Store as TipTap JSON (default: HTML)
;
```

### Toolbar Customization

```php
RichEditor::make('content')
    ->toolbarButtons([
        // Formatting
        'bold', 'italic', 'underline', 'strike',
        'subscript', 'superscript',

        // Headings
        'paragraph', 'h1', 'h2', 'h3', 'h4', 'h5', 'h6',

        // Lists
        'bulletList', 'orderedList',

        // Structure
        'blockquote', 'codeBlock', 'hr',

        // Links and media
        'link', 'unlink', 'table',

        // Alignment
        'alignStart', 'alignCenter', 'alignEnd', 'alignJustify',

        // History
        'undo', 'redo',

        // Clear
        'removeFormat',
    ])
    ->disableToolbarButtons(['h1', 'h2'])
;
```

### Floating Toolbars (Context-Aware)

```php
RichEditor::make('content')
    ->floatingToolbars([
        'paragraph' => ['bold', 'italic', 'underline', 'strike'],
        'heading' => ['h1', 'h2', 'h3'],
        'table' => [
            'tableAddColumnBefore', 'tableAddColumnAfter', 'tableDeleteColumn',
            'tableAddRowBefore', 'tableAddRowAfter', 'tableDeleteRow',
            'tableMergeCells', 'tableSplitCell',
            'tableToggleHeaderRow', 'tableToggleHeaderCell', 'tableDelete',
        ],
    ])
;
```

### File Attachments

```php
RichEditor::make('content')
    ->fileAttachmentsDisk('s3')
    ->fileAttachmentsDirectory('attachments')
    ->fileAttachmentsVisibility('private')
    ->imageEditor()
    ->imageEditorAspectRatios(['16:9', '4:3', '1:1'])
;
```

### Merge Tags

```php
RichEditor::make('content')
    ->mergeTags([
        '{user_name}' => 'User Name',
        '{user_email}' => 'User Email',
        '{current_date}' => 'Current Date',
    ])
    ->mergeTagsPanelColumns(2)
    ->showMergeTagsInBlocksPanel()
;
```

### Mentions

```php
RichEditor::make('content')
    ->mentionCharacters(['@', '#'])
    ->mentionItems(fn (string $query) =>
        User::where('name', 'like', "%{$query}%")
            ->limit(50)->pluck('name', 'id')->all()
    )
;
```

### Custom Blocks

```php
RichEditor::make('content')
    ->customBlocks([
        CustomBlock::make('callout')
            ->label('Callout')
            ->icon(Heroicon::Bell)
            ->previewComponent('callout-preview')
            ->fields([
                Select::make('type')
                    ->options([
                        'info' => 'Info',
                        'warning' => 'Warning',
                        'success' => 'Success',
                    ]),
                TextInput::make('content')->required(),
            ]),
        CustomBlock::make('gallery')
            ->label('Image Gallery')
            ->icon(Heroicon::Photo)
            ->previewComponent('gallery-preview')
            ->fields([
                FileUpload::make('images')->multiple()->image(),
            ]),
    ])
    ->blockPickerColumns(2)
    ->blockPickerDirection('row')
;
```

### Actions

```php
RichEditor::make('content')
    ->hintAction(
        Action::make('AI Write')
            ->icon(Heroicon::Sparkles)
            ->form([Textarea::make('prompt')->required()])
            ->action(function (array $data, RichEditor $component, Set $set) {
                $set($component->getName(), AI::write($data['prompt']));
            })
    )
;
```

### Styling

```php
RichEditor::make('content')
    ->colors([
        'primary' => '#3b82f6',
        'danger' => '#ef4444',
        'info' => '#06b6d4',
        'success' => '#22c55e',
        'warning' => '#f59e0b',
    ])
    ->prose()                             // Use Tailwind prose styles
;
```

### Rendering Rich Content

```php
// In Blade views - HTML content:
<div class="prose">
    {{ str($richEditorContent)->sanitizeHtml() }}
</div>

// JSON content:
<div class="prose">
    {{ $record->content->toHtml() }}
</div>
```

---

## Textarea

```php
use Filament\Forms\Components\Textarea;

Textarea::make('description')
    ->rows(3)
    ->cols(20)
    ->autosize()                  // Auto-resize to content
    ->maxLength(255)
    ->minLength(10)
    ->readOnly()                  // Submitted but not editable
;
```

---

## KeyValue

```php
use Filament\Forms\Components\KeyValue;

KeyValue::make('meta')
    ->keyLabel('Property name')
    ->valueLabel('Property value')
    ->keyPlaceholder('Property name')
    ->valuePlaceholder('Property value')
    ->addable()
    ->deletable()
    ->reorderable()
    ->editableKeys()
    ->editableValues()
    ->required()
;
```

---

## ColorPicker

```php
use Filament\Forms\Components\ColorPicker;

ColorPicker::make('color')
    ->hex()                       // Hex format (default)
    ->rgb()                       // RGB format
    ->rgba()                      // RGBA format
    ->hsl()                       // HSL format
    ->hsla()                      // HSLA format
    ->native(false)               // JavaScript picker
;
```

---

## Hidden

```php
use Filament\Forms\Components\Hidden;
use Illuminate\Support\Str;

Hidden::make('token')
    ->default(Str::random(32))
;
```

---

## Repeater

```php
use Filament\Forms\Components\Repeater;
use Filament\Forms\Components\TextInput;
```

### Basic Repeater

```php
Repeater::make('members')
    ->schema([
        TextInput::make('name')->required(),
        TextInput::make('email')->email()->required(),
    ])
    ->addActionLabel('Add member')
    ->reorderableWithButtons()     // Show up/down buttons
    ->reorderableWithDragAndDrop() // Enable drag-and-drop
    ->collapsed()                  // Collapsed by default
    ->collapsible()                // Allow collapse/expand
    ->cloneable()                  // Allow cloning items
    ->itemLabel(fn (array $state): ?string => $state['name'] ?? null)
    ->defaultItems(3)
    ->maxItems(10)
    ->minItems(1)
    ->grid(2)                      // Display items in grid
;
```

### Relationship Repeater

```php
Repeater::make('members')
    ->relationship('members')
    ->orderColumn('sort')          // Column for storing sort order
    ->schema([
        TextInput::make('name')->required(),
        TextInput::make('email')->email()->required(),
    ])
;
```

### Table Mode (v5)

```php
Repeater::make('lineItems')
    ->schema([
        TextInput::make('description'),
        TextInput::make('quantity')->numeric(),
        TextInput::make('price')->numeric()->prefix('$'),
    ])
    ->table()                      // Display as table
;
```

---

## Builder

```php
use Filament\Forms\Components\Builder;
use Filament\Forms\Components\FileUpload;
use Filament\Forms\Components\Select;
use Filament\Forms\Components\TextInput;
```

```php
Builder::make('content')
    ->blocks([
        Builder\Block::make('heading')
            ->label('Heading')
            ->icon(Heroicon::Bars3BottomLeft)
            ->schema([
                TextInput::make('content')->label('Heading')->required(),
                Select::make('level')->options([
                    'h1' => 'H1',
                    'h2' => 'H2',
                    'h3' => 'H3',
                ])->default('h2'),
            ]),
        Builder\Block::make('paragraph')
            ->label('Paragraph')
            ->icon(Heroicon::Bars3)
            ->schema([
                RichEditor::make('content'),
            ]),
        Builder\Block::make('image')
            ->label('Image')
            ->icon(Heroicon::Photo)
            ->schema([
                FileUpload::make('url')->image(),
                TextInput::make('alt')->label('Alt text'),
            ]),
        Builder\Block::make('quote')
            ->label('Quote')
            ->icon(Heroicon::ChatBubbleBottomCenterText)
            ->schema([
                Textarea::make('content')->required(),
                TextInput::make('author'),
            ]),
    ])
    ->collapsible()
    ->collapsed()
    ->blockNumbers(false)          // Hide block numbers
    ->blockPickerColumns(2)        // Columns in block picker
    ->blockPickerBlocksPerRow(3)   // Blocks per row
    ->blockPickerDirection('row')  // 'row' or 'column'
    ->addActionLabel('Add block')
    ->maxItems(10)
    ->minItems(1)
    ->cloneable()                  // Allow cloning blocks
;
```

---

## TagsInput

```php
use Filament\Forms\Components\TagsInput;

TagsInput::make('tags')
    ->separator(',')
    ->splitKeys(['Enter', ',', 'Tab'])
    ->reorderable()
    ->color('primary')
    ->suggestions(['tailwind', 'alpine', 'laravel', 'livewire'])
    ->placeholder('Add a tag')
    ->nestedRecursiveRules(['min:3'])  // Validation per tag
;
```

---

## ToggleButtons

```php
use Filament\Forms\Components\ToggleButtons;
use Filament\Support\Enums\GridDirection;
```

### Basic

```php
ToggleButtons::make('status')
    ->options([
        'draft' => 'Draft',
        'scheduled' => 'Scheduled',
        'published' => 'Published',
    ])
    ->colors([
        'draft' => 'info',
        'scheduled' => 'warning',
        'published' => 'success',
    ])
    ->icons([
        'draft' => Heroicon::OutlinedPencil,
        'scheduled' => Heroicon::OutlinedClock,
        'published' => Heroicon::OutlinedCheckCircle,
    ])
    ->tooltips([
        'draft' => 'Set as a draft before publishing.',
        'scheduled' => 'Schedule publishing on a specific date.',
        'published' => 'Publish now',
    ])
    ->inline()                     // Inline layout
    ->grouped()                    // Compact grouped buttons
;
```

### Boolean Toggle

```php
ToggleButtons::make('feedback')
    ->label('Like this post?')
    ->boolean(trueLabel: 'Absolutely!', falseLabel: 'Not at all!')
    // Colors and icons auto-configured for Yes/No
;
```

### Multiple Selection

```php
ToggleButtons::make('technologies')
    ->multiple()
    ->options([
        'tailwind' => 'Tailwind CSS',
        'alpine' => 'Alpine.js',
        'laravel' => 'Laravel',
        'livewire' => 'Livewire',
    ])
    ->columns(2)
    ->gridDirection(GridDirection::Row)
;
```

### Disabling Options

```php
ToggleButtons::make('status')
    ->options(['draft' => 'Draft', 'scheduled' => 'Scheduled', 'published' => 'Published'])
    ->disableOptionWhen(fn (string $value): bool => $value === 'published')
    ->in(fn (ToggleButtons $component): array => array_keys($component->getEnabledOptions()))
;
```

---

## Slider

```php
use Filament\Forms\Components\Slider;
use Filament\Forms\Components\Slider\Enums\PipsMode;
use Filament\Support\RawJs;
```

> Uses noUiSlider under the hood. Slider can never be empty/null - defaults to min range value.

### Basic

```php
Slider::make('rating')
    ->range(minValue: 1, maxValue: 5)     // Default: 0-100
    ->step(1)                              // Step increment
    ->default(3)
;
```

### Visual Features

```php
Slider::make('rating')
    ->range(minValue: 0, maxValue: 100)
    ->step(5)
    ->fillTrack()                          // Color the track before handle
    ->tooltips()                           // Show value tooltip
    ->tooltips(RawJs::make(<<<'JS'         // Custom tooltip format
        `${Math.round($value)}%`
    JS))
    ->pips(PipsMode::Steps)                // Add tick marks
    ->pips(PipsMode::Positions, density: 4)// With density
    ->pipsValues([0, 25, 50, 75, 100])     // Custom pip positions
    ->pipsFormatter(RawJs::make(<<<'JS'    // Custom pip labels
        `${$value}%`
    JS))
;
```

### Multiple Handles (Range)

```php
Slider::make('price_range')
    ->range(minValue: 0, maxValue: 1000)
    ->step(10)
    ->default([200, 800])
    ->fillTrack([false, true, false])      // Fill between handles
    ->tooltips([true, true])               // Tooltip per handle
    ->minDifference(100)                   // Min distance between handles
    ->maxDifference(500)                   // Max distance between handles
;
```

### Advanced

```php
Slider::make('volume')
    ->range(minValue: 0, maxValue: 100)
    ->rangePadding(10)                     // Behavioral padding from edges
    ->rangePadding([10, 20])               // Asymmetrical padding
    ->decimalPlaces(2)                     // Decimal precision
    ->vertical()                           // Vertical track
    ->rtl()                                // Right-to-left
;
```

### Pips Modes

| Mode | Description |
|------|-------------|
| `PipsMode::Steps` | Tick at every step |
| `PipsMode::Positions` | Ticks at percentage positions |
| `PipsMode::Values` | Ticks at specific values |
| `PipsMode::Count` | Evenly distribute N ticks |

### Non-Linear Tracks

```php
Slider::make('volume')
    ->range([
        'min' => [0],
        '25%' => [50, 10],    // At 25% of track, value 50, step 10
        '50%' => [100, 50],   // At 50%, value 100, step 50
        'max' => [1000],
    ])
;
```

### Global Configuration

```php
use Filament\Forms\Components\Slider;

Slider::configureUsing(function (Slider $slider): void {
    $slider->range(minValue: 0, maxValue: 100);
});
```

---

## CodeEditor

```php
use Filament\Forms\Components\CodeEditor;
use Filament\Forms\Components\CodeEditor\Enums\Language;
```

```php
CodeEditor::make('code')
    ->language(Language::JavaScript)
    // Available languages: C++, CSS, Go, HTML, Java, JavaScript, JSON,
    //   Markdown, PHP, Python, SQL, XML, YAML
    ->wrap()                           // Wrap long lines (default: horizontal scroll)
;
```

---

## Layout Components

### Grid

```php
use Filament\Schemas\Components\Grid;

Grid::make(2)                              // 2 columns default
    ->schema([
        // ...
    ])
    ->columns([                            // Responsive columns
        'sm' => 1,
        'md' => 2,
        'lg' => 3,
        'xl' => 4,
        '2xl' => 5,
    ])
    ->columnSpan([                         // Span across columns
        'default' => 1,
        'lg' => 2,
    ])
    ->columnSpanFull()                     // Full width
    ->columnStart([                        // Start position
        'lg' => 2,
    ])
    ->key('unique-key')                    // For conditional rendering
;
```

### Section

```php
use Filament\Schemas\Components\Section;
use Filament\Support\Enums\IconPosition;

Section::make('Rate limiting')
    ->description('Prevent abuse by limiting requests')
    ->schema([
        // ...
    ])
    ->icon(Heroicon::ShieldCheck)
    ->iconColor('primary')
    ->iconPosition(IconPosition::Before)   // Before or After
    ->aside()                              // Position heading aside (horizontal)
    ->collapsible()                        // Allow collapse
    ->collapsed()                          // Collapsed by default
    ->persistCollapsed()                   // Persist in localStorage
    ->compact()                            // Compact styling
    ->secondary()                          // Secondary/gray styling
    ->columns(2)                           // Grid columns within section
    ->headerActions([
        Action::make('edit')->icon(Heroicon::Pencil),
    ])
    ->footerActions([
        Action::make('save'),
    ])
    ->extraAttributes(['class' => 'custom-section'])
;
```

### Tabs

```php
use Filament\Schemas\Components\Tabs;
use Filament\Schemas\Components\Tabs\Tab;

Tabs::make('Tabs')
    ->tabs([
        Tab::make('General')
            ->icon(Heroicon::Cog)
            ->iconPosition(IconPosition::After)
            ->badge(5)
            ->badgeColor('primary')
            ->columns(2)
            ->schema([
                // ...
            ]),
        Tab::make('Security')
            ->icon(Heroicon::ShieldCheck)
            ->schema([
                // ...
            ]),
    ])
    ->activeTab(2)                           // Default active (1-based)
    ->contained(false)                       // Remove styled container
    ->isTabPersistedInQueryString()          // Persist in URL (?tab=security)
    ->persistTab()                           // Persist in session
    ->vertical()                             // Vertical tab layout
;
```

### Wizard

```php
use Filament\Schemas\Components\Wizard;
use Filament\Schemas\Components\Wizard\Step;

Wizard::make([
    Step::make('Order')
        ->icon(Heroicon::ShoppingBag)
        ->completedIcon(Heroicon::HandThumbUp)
        ->description('Review your order')
        ->columns(2)
        ->schema([
            // ...
        ])
        ->afterValidation(function () {
            // Runs after step validation passes
        })
        ->beforeValidation(function () {
            // Runs before step validation
        }),
    Step::make('Delivery')
        ->schema([/* ... */]),
    Step::make('Billing')
        ->schema([/* ... */]),
])
    ->startOnStep(2)                         // Default active step
    ->skippable()                            // Allow skipping steps
    ->persistStepInQueryString('step')       // Persist in URL
    ->nextAction(fn (Action $action) => $action->label('Next step'))
    ->previousAction(fn (Action $action) => $action->label('Back'))
    ->submitAction(view('forms.components.wizard.submit-button'))
    ->contained(false)
;
```

### Fieldset

```php
use Filament\Schemas\Components\Fieldset;

Fieldset::make('Address')
    ->schema([
        TextInput::make('street'),
        TextInput::make('city'),
        TextInput::make('zip'),
    ])
;
```

### Flex

```php
use Filament\Schemas\Components\Flex;

Flex::make()
    ->schema([
        // Arranged in flexbox layout
    ])
;
```

### Callout (Info Box)

```php
use Filament\Schemas\Components\Callout;

Callout::make('notice')
    ->title('Important Information')
    ->body('Please review the terms before continuing.')
    ->icon(Heroicon::InformationCircle)
    ->color('warning')
;
```

---

## Validation

### Dedicated Validation Methods

| Method | Description |
|--------|-------------|
| `->required()` | Field must be filled |
| `->requiredIf('field', 'value')` | Required when another field equals value |
| `->requiredIfAccepted('field')` | Required when field is accepted |
| `->requiredUnless('field', 'value')` | Required unless another field equals value |
| `->requiredWith(['field'])` | Required when any of fields are present |
| `->requiredWithAll(['f1', 'f2'])` | Required when all fields are present |
| `->requiredWithout(['field'])` | Required when any of fields are missing |
| `->nullable()` | Allow null values |
| `->filled()` | Must have a value when present |

| Method | Description |
|--------|-------------|
| `->email()` | Valid email format |
| `->url()` | Valid URL format |
| `->activeUrl()` | DNS-valid URL |
| `->ip()` | Valid IP address |
| `->ipv4()` | Valid IPv4 |
| `->ipv6()` | Valid IPv6 |
| `->macAddress()` | Valid MAC address |

| Method | Description |
|--------|-------------|
| `->alpha()` | Letters only |
| `->alphaDash()` | Letters, numbers, dashes, underscores |
| `->alphaNumeric()` | Letters and numbers |
| `->ascii()` | ASCII characters only |
| `->json()` | Valid JSON string |

| Method | Description |
|--------|-------------|
| `->maxLength(255)` | Max string length |
| `->minLength(2)` | Min string length |
| `->length(10)` | Exact string length |
| `->maxValue(100)` | Max numeric value |
| `->minValue(1)` | Min numeric value |
| `->size(10)` | Exact size (arrays: item count) |
| `->multipleOf(5)` | Must be multiple of value |

| Method | Description |
|--------|-------------|
| `->after('tomorrow')` | Date after given date |
| `->afterOrEqual('today')` | Date after or equal |
| `->before('2025-01-01')` | Date before given date |
| `->beforeOrEqual('today')` | Date before or equal |

| Method | Description |
|--------|-------------|
| `->confirmed()` | Must have matching `_confirmation` field |
| `->different('field')` | Must differ from another field |
| `->endsWith(['bot'])` | Must end with given string(s) |
| `->doesntStartWith(['admin'])` | Must not start with |
| `->doesntEndWith(['admin'])` | Must not end with |
| `->startsWith(['http'])` | Must start with |

| Method | Description |
|--------|-------------|
| `->enum(Status::class)` | Must be valid enum case |
| `->exists(table: User::class, column: 'id')` | Must exist in DB table |
| `->unique(table: User::class, column: 'email')` | Must be unique |
| `->unique(ignoreRecord: true)` | Unique, ignoring current record |
| `->scopedUnique()` | Unique with global scopes (multi-tenant) |
| `->in(['draft', 'published'])` | Must be in array |
| `->notIn(['archived'])` | Must not be in array |
| `->hexColor()` | Valid hex color |
| `->notRegex('/pattern/')` | Must not match regex |

| Method | Description |
|--------|-------------|
| `->prohibited()` | Field must be empty |
| `->prohibitedIf('field', 'value')` | Prohibited when condition met |
| `->prohibitedUnless('field', 'value')` | Prohibited unless condition met |
| `->prohibits(['credit_card'])` | If this has value, those must be empty |

### Conditional Validation

```php
TextInput::make('credit_card')
    ->required(fn (Get $get): bool => $get('payment_type') === 'credit_card')
    ->maxLength(16)
;
```

### Custom Rules

```php
use Illuminate\Validation\Rule;

TextInput::make('name')
    ->rules([
        'required',
        'max:255',
        new CustomRule(),
        Rule::unique('users', 'name')->ignore($this->record),
    ])
    ->rule('required')
    ->rule(fn (Get $get): string => "unique:users,name,{$get('id')}")
;
```

### Validation Messages

```php
TextInput::make('email')
    ->required()
    ->email()
    ->validationMessages([
        'required' => 'The email address is required.',
        'email' => 'Please enter a valid email address.',
    ])
    ->allowHtmlValidationMessages() // Allow HTML (XSS risk!)
;
```

### Disabling Validation for Non-Dehydrated Fields

```php
TextInput::make('name')
    ->required()
    ->saved(false)
    ->validatedWhenNotDehydrated(false)
;
```

---

## Utility Injection

Closure parameters are auto-injected by name via reflection:

```php
use Filament\Schemas\Components\Utilities\Get;
use Filament\Schemas\Components\Utilities\Set;

// Get another field's value
TextInput::make('slug')
    ->disabled()
    ->dehydrated()
    ->formatStateUsing(function ($state, Get $get) {
        return $get('title') ? Str::slug($get('title')) : $state;
    })
;

// Set another field's value
TextInput::make('title')
    ->live(onBlur: true)
    ->afterStateUpdated(function (Set $set, $state) {
        $set('slug', Str::slug($state));
    })
;

// Access the record
TextInput::make('email')
    ->default(fn (?Model $record): string => $record?->email ?? '')
;

// Operation-aware
TextInput::make('password')
    ->required(fn (string $operation): bool => $operation === 'create')
;

// Combine multiple utilities
Select::make('country')
    ->options(function (Livewire $livewire, Get $get) {
        return Country::query()
            ->where('region', $livewire->region)
            ->pluck('name', 'id');
    })
;
```

### Available Utilities

| Parameter | Type | Description |
|-----------|------|-------------|
| `$get` | `Filament\Schemas\Components\Utilities\Get` | Get other field values |
| `$set` | `Filament\Schemas\Components\Utilities\Set` | Set other field values |
| `$record` | `?Illuminate\Database\Eloquent\Model` | Current Eloquent record (null on create) |
| `$operation` | `string` | 'create', 'edit', 'view' |
| `$livewire` | `Livewire\Component` | Livewire component instance |
| `$component` | `Filament\Forms\Components\Field` | Current field instance |
| `$state` | `mixed` | Current field value (casted) |
| `$rawState` | `mixed` | Current field value (before casts) |
| `$model` | `?string` | FQN of the Eloquent model |

---

## Conditional Visibility

### hiddenOn / visibleOn

```php
// Hide on specific operations
TextInput::make('password')
    ->hiddenOn('edit')
    // Equivalent to:
    ->hidden(fn (string $operation): bool => $operation === 'edit')
;

// Hide on multiple operations
TextInput::make('id')
    ->hiddenOn(['create', 'edit'])
;

// Show only on specific operation
TextInput::make('id')
    ->visibleOn('view')
    // Equivalent to:
    ->visible(fn (string $operation): bool => $operation === 'view')
;
```

### Dynamic hidden / visible

```php
TextInput::make('company_name')
    ->hidden(fn (Get $get): bool => $get('account_type') !== 'company')
;

TextInput::make('ssn')
    ->visible(fn (Get $get): bool => in_array($get('country'), ['US', 'CA']))
;
```

### disabledOn

```php
TextInput::make('email')
    ->disabledOn(['edit', 'view'])
    // Equivalent to:
    ->disabled(fn (string $operation): bool => in_array($operation, ['edit', 'view']))
;
```

### saved(false) for Disabled Fields

```php
TextInput::make('total')
    ->disabled()
    ->dehydrated()          // Include in form data
    ->saved(false)          // But don't save to database
;
```

### Live Reactivity

```php
// Update on every keystroke
Select::make('country')
    ->live()
    ->options([...])
;

// Update only on blur (performance)
TextInput::make('title')
    ->live(onBlur: true)
    ->afterStateUpdated(fn (Set $set, ?string $state) => $set('slug', Str::slug($state)))
;
```

---

## Common Field Methods (All Fields)

```php
Field::make('name')
    // Label
    ->label('Display Name')
    ->hiddenLabel()              // Visually hide (a11y-safe)
    ->inlineLabel()              // Place label inline with input
    ->inlineLabelPosition('start') // 'start' or 'end'

    // State
    ->default('default_value')
    ->formatStateUsing(fn (string $state): string => strtoupper($state))
    ->dehydrateStateUsing(fn (string $state): string => trim($state))
    ->mutateDehydratedStateUsing(fn ($state) => $state)

    // Visibility / Editability
    ->hidden()
    ->hiddenOn('create')
    ->visible()
    ->visibleOn('view')
    ->disabled()
    ->disabledOn('edit')
    ->readOnly()
    ->dehydrated()               // Include in form submission (default: true)
    ->dehydrated(false)          // Exclude from form data
    ->saved(true)                // Save to database (default: true)
    ->saveRelationshipsUsing(fn (Model $record, $state) => ...)
    ->loadRelationshipsFrom(fn (Model $record) => ...)

    // Reactivity
    ->live()                     // Reactive (triggers on change)
    ->live(onBlur: true)         // Reactive only on blur
    ->live(condition: fn () => auth()->user()->isAdmin())
    ->afterStateUpdated(fn (Set $set, $state) => ...)
    ->afterStateHydrated(fn (Set $set, $state) => ...)

    // UI hints
    ->placeholder('Enter value...')
    ->hint('Helper text')
    ->hintIcon(Heroicon::QuestionMarkCircle)
    ->hintColor('warning')
    ->hintAction(Action::make('help')->icon(Heroicon::InformationCircle))
    ->helperText('Additional context below the field')
    ->prefix('USD ')
    ->suffix(' per month')

    // Styling
    ->extraAttributes(['class' => 'custom-class', 'data-foo' => 'bar'])
    ->extraFieldWrapperAttributes(['class' => 'col-span-2'])
    ->extraInputAttributes(['autocomplete' => 'off'])
    ->columnSpan(2)
    ->columnSpanFull()

    // Validation
    ->required()
    ->requiredIf('field', 'value')
    ->validationMessages(['required' => 'Custom message'])

    // Unique field ID (for conditional rendering)
    ->key('unique-field-key')
;
```

---

## Global Configuration (Service Provider)

```php
use Filament\Forms\Components\TextInput;
use Filament\Forms\Components\Slider;
use Filament\Schemas\Components\Section;

// Set defaults for all instances
TextInput::configureUsing(function (TextInput $component): void {
    $component->trim();
});

Section::configureUsing(function (Section $section): void {
    $section->columns(2);
});

Slider::configureUsing(function (Slider $slider): void {
    $slider->range(minValue: 0, maxValue: 100);
});

FileUpload::configureUsing(function (FileUpload $component): void {
    $component->preventFilePathTampering();
});
```

---

## Dot Notation for Nested Data

```php
// Array keys
TextInput::make('socials.github_url')
TextInput::make('socials.twitter_url')

// Relationship access
Select::make('author_id')
    ->relationship('author', 'name')

// Access via Get utility
$set('author.name', 'John');  // Not $set('author->name', ...)
$authorName = $get('author.name');
```

---

## Best Practices Summary

1. **Use dedicated validation methods** over `->rules()` for IDE autocomplete + frontend validation
2. **Hide labels properly** with `->hiddenLabel()` (not empty string) for accessibility
3. **File uploads**: Never use `preserveFilenames()` on local/public disks without security review. Use S3 or `getUploadedFileNameForStorageUsing()` + `storeFileNamesIn()`
4. **File uploads**: Enable `preventFilePathTampering()` globally in production
5. **Select with large datasets**: Use `->searchable()` + `->getSearchResultsUsing()` for 50+ options. Avoid `->preload()` for large sets.
6. **Reactive forms**: Use `->live(onBlur: true)` for better performance (fewer server round-trips)
7. **Multi-tenant**: Use `->scopedUnique()` instead of `->unique()` for tenant-aware uniqueness
8. **Rich editor**: Content is sanitized by default; use `->rawHtml()` with extreme caution
9. **Disabled fields**: Use `->disabled()` + `->dehydrated()` + `->saved(false)` to display computed values
10. **Conditional visibility**: Prefer `->hidden()`/`->visible()` closures over `if/else` in schema arrays
