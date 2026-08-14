# Windows, menus and native UI

## Contents
- [Windows](#windows)
- [Window events](#window-events)
- [Application menus](#application-menus)
- [Menu items](#menu-items)
- [Context menus](#context-menus)
- [The menu bar (tray)](#the-menu-bar-tray)
- [Dialogs](#dialogs)
- [Alerts](#alerts)
- [Notifications](#notifications)
- [Dock](#dock)

## Windows

```php
use Native\Desktop\Facades\Window;

Window::open()->width(800)->height(600);
```

Every window has an ID; the default is `main`. Pass one to `open()` to distinguish windows, then use it to address the window later. Methods that take an optional ID (`close`, `resize`, `minimize`, `maximize`, `alwaysOnTop`) will infer the window from the current route if you omit it — convenient inside a controller, ambiguous elsewhere, so be explicit when the calling context isn't a window's own request.

```php
Window::open('settings')->route('settings')->width(600)->height(400);

Window::close('settings');
Window::hide('settings');
Window::show('settings');
Window::resize(400, 300, 'settings');
Window::minimize('settings');
Window::maximize('settings');
Window::alwaysOnTop(true, 'settings');
```

### hide() and show() are real but undocumented

`Window::hide()` and `Window::show()` exist on `WindowManager` and on the test fake, but neither appears in the `Window` facade's docblock, and `show()` is missing from the `WindowManager` contract as well. They work at runtime; IDE autocomplete and PHPStan will not find them, and the published docs never mention them. `show()` is the correct way to bring back a window hidden with `hide()` — don't reach for `open()`, which is for windows that don't exist.

Hidden is not the same as closed. `show()` will not resurrect a closed window; check `Window::all()` and fall back to `Window::open()`.

### When you must pass the window ID

Omitting the ID is not a general convenience. `DetectsWindowId` resolves it from a `_windowId` query parameter on the request's `Referer` (falling back to the current URL). That works inside a controller handling a request from a window, and returns `null` everywhere else.

So pass the ID explicitly from anywhere without an originating window request — menu item event listeners, queued jobs, scheduled tasks, child process output handlers, `NativeAppServiceProvider::boot()`. A silently-null ID here is a common cause of "the method did nothing".

Retrieving windows:

```php
$window = Window::current();  // focused window: id, title, width, height, x, y, alwaysOnTop
$all = Window::all();
$settings = Window::get('settings');

Window::get('settings')->url(route('home'));
Window::get('settings')->title('Mmmm... delicious!');
```

Changing a window's URL from outside the request cycle is how you drive navigation from a menu item click, a queued job, or child process output.

### Configuring a window

Chained onto `Window::open()`:

| Concern | Methods |
|---|---|
| Content | `route(string)`, `url(string)`, `title(string)` |
| Size | `width()`, `height()`, `minWidth()`, `minHeight()`, `maxWidth()`, `maxHeight()` |
| Position | `position(int $x, int $y)`, `rememberState()` |
| State on open | `minimized()`, `maximized()`, `fullscreen()`, `kiosk()` |
| Chrome | `titleBarHidden()`, `titleBarHiddenInset()`, `trafficLightsHidden()` (macOS), `trafficLightPosition(int $x, int $y)`, `frameless()`, `hideMenu()`, `hasShadow(false)` |
| Behaviour | `resizable(false)`, `movable(false)`, `focusable(false)`, `minimizable(false)`, `maximizable(false)`, `closable(false)`, `fullscreenable(false)`, `alwaysOnTop()` |
| Visibility | `skipTaskbar()`, `hiddenInMissionControl()` (macOS) |
| Appearance | `backgroundColor('#00000050')`, `zoomFactor(1.25)` |
| Navigation | `preventLeaveDomain()`, `preventLeavePage()`, `suppressNewWindows()` |
| Web layer | `webPreferences([...])` |
| Dev | `showDevTools()`, `hideDevTools()`, `devToolsOpen()` |

`rememberState()` persists size and position across launches, but only for one window at a time.

`backgroundColor()` shows during resize before content paints — matching it to your app background removes a visible flash.

`zoomFactor()` takes a multiplier, not a percentage: 125% is `1.25`.

`hideMenu()` hides the application menu on Windows and Linux, revealing it on ALT.

### Custom title bars

With `titleBarHidden()` you supply your own title bar, and it needs an explicitly draggable region or the window can't be moved:

```html
<div style="height: 30px; -webkit-app-region: drag;">
    <!-- your title bar content -->
</div>
```

### Navigation restrictions

When a window renders content you don't control, constrain it:

```php
Window::open()->url('https://nativephp.com/')->preventLeaveDomain();
Window::open()->url('https://example.com/page')->preventLeavePage();
Window::open()->suppressNewWindows();
```

`preventLeaveDomain()` blocks navigation to a different domain, scheme or port. `preventLeavePage()` confines the user to the page as rendered, still permitting anchors and query-string changes. `suppressNewWindows()` stops `target="_blank"` and middle-clicks from spawning windows the user can "break out" into.

Never render an untrusted external site inside your own window without these.

### webPreferences

```php
Window::open()->webPreferences([
    'nodeIntegration' => true,
    'spellcheck' => true,
    'backgroundThrottling' => true,
]);
```

Defaults in v2:

```php
['sandbox' => false, 'preload' => '{preload-path}', 'contextIsolation' => true,
 'spellcheck' => false, 'nodeIntegration' => false, 'backgroundThrottling' => false]
```

`sandbox`, `preload` and `contextIsolation` are locked for security and cannot be overridden. Everything else from Electron's WebPreferences is accepted. Note that `nodeIntegration` defaults to `false` in v2 where v1 defaulted to `true` — re-enable it deliberately and only where needed, since it hands your renderer full Node access.

## Window events

All dispatch as ordinary Laravel events under `Native\Desktop\Events\Windows\` and are also broadcast to the `nativephp` channel:

`WindowShown`, `WindowHidden`, `WindowClosed`, `WindowFocused`, `WindowBlurred`, `WindowMinimized`, `WindowMaximized`, `WindowUnmaximized`, `WindowResized`.

Each carries the window ID; `WindowResized` also carries `$width` and `$height`.

```php
use Native\Desktop\Events\Windows\WindowFocused;

Event::listen(WindowFocused::class, function (WindowFocused $event) {
    // $event->id
});
```

See `data-and-processes.md` for listening in JavaScript and Livewire.

## Application menus

One unified `Menu` facade builds the application menu, context menus, menu bar menus and Dock menus.

```php
use Native\Desktop\Facades\Menu;

Menu::create(
    Menu::app(),      // macOS only, and always first
    Menu::file(),
    Menu::edit(),
    Menu::view(),
    Menu::window(),
);

// equivalently
Menu::default();
```

`Menu::create()` both builds and registers. Call it again at any time — from a listener, controller or Livewire action — to swap the menu in response to application state.

Predefined roles: `app()`, `about()`, `file()`, `edit()`, `view()`, `window()`, `help()`. Each takes an optional label. On macOS the first item always displays your application's name regardless of the label you set.

The predefined menus carry the standard hotkeys (cut, copy, paste, and so on). Build custom replacements and you inherit responsibility for wiring those shortcuts yourself — a common source of "why can't I paste in my app" bug reports.

`Menu::make(...)` returns a `Native\Desktop\Menu\Menu` instance rather than registering it. Menus are themselves menu items, so they nest as submenus:

```php
Menu::create(
    Menu::app(),
    Menu::make(
        Menu::link('https://nativephp.com', 'Documentation'),
    )->label('My Submenu'),
);
```

## Menu items

```php
Menu::checkbox(string $label, bool $checked = false, ?string $hotkey = null);
Menu::label(string $label, ?string $hotkey = null);
Menu::link(string $url, ?string $label = null, ?string $hotkey = null);
Menu::radio(string $label, bool $checked = false, ?string $hotkey = null);
Menu::route(string $route, ?string $label = null, ?string $hotkey = null);
```

Chainable on any item:

```php
Menu::route('welcome')
    ->label('Home')
    ->id('my-item')
    ->icon(public_path('icon.png'))
    ->visible(false)
    ->tooltip('Hover text')   // macOS only
    ->hotkey('Cmd+F')
    ->disabled();
```

Special items with fixed behaviour and no click event — you may only change their labels: `separator()`, `undo()`, `redo()`, `cut()`, `copy()`, `paste()`, `pasteAndMatchStyle()`, `reload()`, `fullscreen()`, `minimize()`, `close()`, `hide()`, `quit()`, `devTools()`.

### Handling clicks

```php
Menu::label('Click me!')->event(MyCustomMenuItemEvent::class);
```

The custom event class extends `Native\Desktop\Events\Menu\MenuItemClicked`. Without one, the default `MenuItemClicked` fires and is broadcast, so you can handle it in PHP, in JavaScript, or both. When triggered by a hotkey, `$combo['triggeredByAccelerator']` is `true`.

**The payload is `public array $item` and `public array $combo` — there is no window ID.** A listener therefore cannot tell which window the click came from, and omitting the ID on a subsequent `Window::` call resolves to `null` (see the window-ID detection note above), so nothing happens. For anything that acts on a window, either use the predefined role item, which the runtime handles natively, or track the target window yourself.

This is why a hand-rolled "reload the current window" menu item quietly fails while `Menu::reload()` — or the reload already inside `Menu::view()` — just works. Prefer the roles for window and edit actions; reserve custom events for application logic.

Checkbox and radio items expose their state as `$item['checked']`.

Radio items group by position, not name — all radios in a menu form one group unless separated by `Menu::separator()`, which starts a new group.

Link items navigate the focused window. To open in the user's browser instead:

```php
Menu::link('https://nativephp.com/', 'Documentation')->openInBrowser();
```

Hotkey modifiers and key codes are listed in `system-apis.md` under global hotkeys. Menu hotkeys fire only when one of your windows is focused or the relevant context menu is open — that's the difference from global hotkeys.

## Context menus

For per-element context menus, use the injected `Native` JavaScript helper. It takes Electron `MenuItem` option objects:

```js
element.addEventListener('contextmenu', (event) => {
    event.preventDefault()

    Native.contextMenu([
        { label: 'Duplicate', accelerator: 'd', click() { duplicateEntry(element.dataset.id) } },
        { label: 'Edit', accelerator: 'e', click() { showEditForm(element.dataset.id) } },
        { label: 'Delete', click() { /* ... */ } },
    ])
})
```

There is also a `ContextMenu` facade (`register(Menu $menu)` / `remove()`) for registering a PHP-built menu.

## The menu bar (tray)

```php
use Native\Desktop\Facades\MenuBar;

MenuBar::create();
```

Creating a menu bar hides the Dock icon by default, which is what you want for a standalone tray app. For an app that also has windows, keep the Dock icon:

```php
MenuBar::create()->showDockIcon();
```

Configuration chained onto `create()`, or called on the facade later to change it at runtime:

```php
MenuBar::create()
    ->route('home')                  // or ->url('https://…')
    ->icon(storage_path('app/menuBarIconTemplate.png'))
    ->label('Status: Online')
    ->tooltip('Click to open')
    ->width(400)->height(400)        // default is 400×400
    ->x(100)->y(200)
    ->alwaysOnTop()
    ->vibrancy('light')              // macOS
    ->backgroundColor('#ffffff')
    ->webPreferences([...]);

MenuBar::show();
MenuBar::hide();
MenuBar::label('');                  // empty string removes the label
MenuBar::resizable(false);
```

`alwaysOnTop()` during development saves clicking the tray icon on every restart.

### Icons

A PNG with a transparent background, 22×22, plus a 44×44 `@2x` variant alongside it — NativePHP picks the retina file up automatically from the naming convention. On macOS, name the file with a `Template` suffix (`menuBarIconTemplate.png`) and it will be rendered as a template image that adapts to light and dark menu bars.

### Context menu on the tray icon

```php
use Native\Desktop\Facades\Menu;

MenuBar::create()->withContextMenu(
    Menu::make(
        Menu::label('My Application'),
        Menu::separator(),
        Menu::link('https://nativephp.com', 'Learn more…')->openInBrowser(),
        Menu::separator(),
        Menu::quit(),
    )
);

MenuBar::showContextMenu();          // trigger it programmatically
MenuBar::onlyShowContextMenu();      // left-click shows the menu instead of a window
```

### Menu bar events

Under `Native\Desktop\Events\MenuBar\`: `MenuBarShown`, `MenuBarHidden`, `MenuBarCreated`, `MenuBarClicked`, `MenuBarDoubleClicked`, `MenuBarRightClicked`, `MenuBarDroppedFiles`.

The docs also mention a `MenuBarContextMenuOpened` event; no such class exists in the package. Use `MenuBarRightClicked` for the right-click case.

Also available on `MenuBar::create()`: `showOnAllWorkspaces()`.

A custom click event works only in combination with `onlyShowContextMenu()`:

```php
MenuBar::create()->event(MenuBarClicked::class);

class MenuBarClicked
{
    public function __construct(
        public array $combo,     // modifier keys held
        public array $bounds,    // absolute bounds of the icon
        public array $position,  // absolute cursor position
    ) {}
}
```

## Dialogs

`Dialog` is a class, not a facade:

```php
use Native\Desktop\Dialog;

$path = Dialog::new()
    ->title('Select a file')
    ->button('Select')
    ->defaultPath('/Users/username/Desktop')
    ->filter('Images', ['jpg', 'png', 'gif'])
    ->filter('Documents', ['pdf', 'docx'])
    ->open();
```

`open()` returns `null`, a path string, or an array of paths with `multiple()`. `save()` returns the path the user chose but does **not** write anything — you still have to save the file yourself.

Other modifiers: `multiple()`, `folders()`, `withHiddenFiles()`, `dontResolveSymlinks()`, `asSheet(?string $windowId = null)` to attach the dialog to a window rather than float it independently.

## Alerts

```php
use Native\Desktop\Facades\Alert;

$clicked = Alert::new()
    ->title('Pizza Order')
    ->type('question')                        // none|info|warning|error|question
    ->detail('Fun fact: pizza was first made in Naples')
    ->buttons(['Yes', 'No', 'Maybe'])
    ->defaultId(0)
    ->cancelId(1)
    ->show('Do you like pizza?');
```

`show()` returns the zero-based index of the button clicked. With no `buttons()`, the alert shows only OK. `cancelId` marks the button Escape selects; without it, the first button labelled Cancel or No is used, falling back to `0`.

Platform quirks: on Windows `question` renders the same icon as `info`; on macOS `warning` and `error` render identically.

```php
Alert::new()->error('An error occurred', 'The pizza oven is broken');
```

## Notifications

These are OS notifications, not Laravel notifications.

```php
use Native\Desktop\Facades\Notification;

Notification::title('Hello from NativePHP')
    ->message('A detail message from your Laravel app.')
    ->event(\App\Events\MyNotificationEvent::class)
    ->reference($post->id)
    ->sound('Ping')                 // system sound name, or a path to an audio file
    ->show();
```

The `reference` is the key to handling clicks meaningfully. One is generated automatically, but setting it yourself lets a listener work out *which* notification was clicked:

```php
Event::listen(PostNotificationClicked::class, function ($event) {
    $post = Post::findOrFail($event->reference);
    Window::open()->url($post->url);
});
```

Also available: `silent()`, `hasReply(?string $placeholder = null)` (macOS), and `addAction(string $label)` (macOS), which can be called repeatedly. Action clicks fire `NotificationActionClicked` with a zero-based `$index`.

Events under `Native\Desktop\Events\Notifications\`: `NotificationClicked`, `NotificationClosed`, `NotificationReply`, `NotificationActionClicked`.

Notifications interrupt. Use them for things that genuinely warrant pulling the user back to your app.

## Dock

macOS Dock control via the `Dock` facade: `bounce(string $type = 'informational')`, `cancelBounce()`, `badge(?string $type = null)`, `icon(string $path)`, `menu(Menu $menu)`, `show()`, `hide()`.
