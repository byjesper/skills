# System APIs

Facades covered here: `App`, `System`, `Screen`, `Shell`, `Clipboard`, `GlobalShortcut`, `PowerMonitor`, `Settings`. All under `Native\Desktop\Facades\`.

Several methods are platform-specific. NativePHP degrades gracefully rather than fataling, but check the platform notes before building a feature that only exists on one OS.

## Contents
- [App](#app)
- [System](#system)
- [Settings](#settings)
- [Global hotkeys](#global-hotkeys)
- [Clipboard](#clipboard)
- [Shell](#shell)
- [Screen](#screen)
- [Power monitor](#power-monitor)

## App

```php
use Native\Desktop\Facades\App;

App::quit();
App::relaunch();          // quit and start again
App::focus();
App::version();           // from config('nativephp.version')
App::isRunningBundled();  // true in a built app, false in dev
```

Locale, useful for suggesting a language on first launch:

```php
App::getLocale();             // "de", "fr-FR" — the locale the app is using
App::getLocaleCountryCode();  // "US", "DE" — ISO 3166, from native OS APIs, "" if undetectable
App::getSystemLocale();       // "it-IT" — the OS-level setting, not necessarily the app's
```

macOS only:

```php
App::hide();          // hides all windows without minimising
App::isHidden();      // e.g. after Cmd-H
```

macOS and Linux (Unity launcher only on Linux):

```php
App::badgeCount(5);
App::badgeCount(0);   // clear
$count = App::badgeCount();
```

macOS and Windows:

```php
App::addRecentDocument('/path/to/document');
App::recentDocuments();
App::clearRecentDocuments();

App::openAtLogin(true);
App::openAtLogin(false);
$enabled = App::openAtLogin();
```

Also: `App::isEmojiPanelSupported()`, `App::showEmojiPanel()`.

## System

### Encryption

This is the mechanism for storing anything sensitive on a user's machine. NativePHP generates and stores the key in the OS keychain equivalent; you only handle the plaintext and ciphertext.

```php
use Native\Desktop\Facades\System;

if (System::canEncrypt()) {
    $encrypted = System::encrypt('secret_key_a79hiunfw86...');
    // store $encrypted in the database or a file
}

if (System::canEncrypt()) {
    $decrypted = System::decrypt($encrypted);
}
```

Always guard with `canEncrypt()` — it can be unavailable, particularly on Linux without a keyring. Decrypt at the point of use and don't hold plaintext longer than needed.

### TouchID

```php
if (System::canPromptTouchID() && System::promptTouchID('access your Contacts')) {
    // gated action
}
```

The `$reason` string is required and appears in the system dialog, so phrase it as the user-facing justification.

TouchID raises confidence that the person at the keyboard is the person who unlocked the device. It does not identify the user and grants no privileges. Don't treat it as authentication against a remote service.

### Printing

```blade
@use(Native\Desktop\Facades\System)
@foreach(System::printers() as $printer)
    {{ $printer->displayName }}
@endforeach
```

Each entry is a `Native\Desktop\DataObjects\Printer` carrying device details and default configuration.

```php
System::print('<html>...', $printer);            // omit $printer for the default
System::print('<html>...', $printer, $settings);
```

Settings map to Electron's `webContents.print()` options:

```php
$settings = [
    'pageSize' => 'A4',
    'landscape' => true,
    'copies' => 3,
    'duplexMode' => 'longEdge',   // simplex | shortEdge | longEdge
    'color' => false,
];
```

You can also mutate a printer's own options: `$printer->options['copies'] = 5;`

Printing to PDF returns base64 — decode before writing:

```php
$pdf = System::printToPDF('<html>...');
Storage::disk('desktop')->put('My Awesome File.pdf', base64_decode($pdf));
```

### Timezone

Your Laravel app is probably configured for UTC; the user thinks in their OS timezone.

```php
$timezone = System::timezone();   // "Europe/London"
```

This maps cross-platform timezone identifiers to ones PHP understands and responds to the user moving between zones. It's an approximation — overlapping zones mean it may not pick the exact one — so use it for display, not for anything where an hour's error matters legally.

### Theme

```php
use Native\Desktop\Enums\SystemThemesEnum;

$theme = System::theme();                      // LIGHT, DARK or SYSTEM
System::theme(SystemThemesEnum::DARK);         // override
System::theme(SystemThemesEnum::SYSTEM);       // remove the override
```

Default is `SYSTEM`.

## Settings

Small key/value persistence in `config.json` in the appdata directory. Good for preferences; not a substitute for the database.

```php
use Native\Desktop\Facades\Settings;

Settings::set('key', 'value');
Settings::get('key');
Settings::get('key', 'default');
Settings::get('key', fn () => 'default');
Settings::forget('key');
Settings::clear();
```

Missing keys with no default return `null`.

`Native\Desktop\Events\Settings\SettingChanged` fires on every change and carries `$key` and `$value`. (The published docs cite this under `Events\Notifications\` — that namespace is wrong.)

```php
use Livewire\Component;
use Native\Desktop\Events\Settings\SettingChanged;

class Settings extends Component
{
    protected $listeners = ['native:'.SettingChanged::class => '$refresh'];
}
```

## Global hotkeys

Global hotkeys fire even when your app is in the background and unfocused — that's the distinction from menu-item hotkeys, which need one of your windows focused. Register them in `NativeAppServiceProvider::boot()`.

```php
use Native\Desktop\Facades\GlobalShortcut;

GlobalShortcut::key('CmdOrCtrl+Shift+A')
    ->event(\App\Events\MyShortcutEvent::class)
    ->register();

GlobalShortcut::key('CmdOrCtrl+Shift+A')->unregister();
```

Each combination registers once, so `unregister()` needs only the key.

Modifiers: `Command`/`Cmd`, `Control`/`Ctrl`, `CommandOrControl`/`CmdOrCtrl`, `Alt`, `Option`, `AltGr`, `Shift`, `Super`, `Meta`.

Key codes: `0`–`9`, `A`–`Z`, `F1`–`F24`, `Backspace`, `Delete`, `Insert`, `Return`/`Enter`, `Up`, `Down`, `Left`, `Right`, `Home`, `End`, `PageUp`, `PageDown`, `Escape`/`Esc`, `VolumeUp`, `VolumeDown`, `VolumeMute`, `MediaNextTrack`, `MediaPreviousTrack`, `MediaStop`, `MediaPlayPause`, `PrintScreen`, `Numlock`, `Scrolllock`, `Space`, `Plus`.

Prefer `CmdOrCtrl` over `Cmd` unless the app is macOS-only. Global hotkeys are a shared resource — a common combination stolen system-wide is a genuine annoyance, so pick something distinctive.

## Clipboard

```php
use Native\Desktop\Facades\Clipboard;

Clipboard::text();                              // read
Clipboard::text('Some copied text');            // write
Clipboard::html();
Clipboard::html('<div>Some copied HTML</div>');
Clipboard::image();
Clipboard::image('path/to/image.png');          // takes a path, not image data
Clipboard::clear();
```

`clear()` is worth using when you've written something sensitive and want it to expire.

## Shell

Delegates to the OS's default handlers.

```php
use Native\Desktop\Facades\Shell;

Shell::showInFolder($path);        // reveal in Finder / Explorer
$result = Shell::openFile($path);  // "" on success; an error message otherwise
Shell::trashFile($path);           // to the system trash, not deleted outright
Shell::openExternal($url);         // default browser for http/https
```

`trashFile()` over `unlink()` for anything the user created — recoverable beats gone.

`openExternal()` is the right answer for external links; opening them in your own window makes your app a browser you now have to secure.

## Screen

```php
use Native\Desktop\Facades\Screen;

$screens = Screen::displays();
$position = Screen::cursorPosition();   // (object) ['x' => 627, 'y' => 168]
```

`displays()` returns an array of display descriptors with `bounds` (x, y, width, height), `size`, `workArea`, `id`, `label`, `internal`, `detected`. Only displays actually in use appear — a laptop screen closed behind an external monitor won't be listed.

Coordinates are relative to the top-left of the primary display and can be negative when a secondary display sits to the left or above. Bounds are the full screen; `workArea` excludes menu bars and docks, so use `workArea` when positioning windows.

## Power monitor

```php
use Native\Desktop\Enums\SystemIdleStatesEnum;
use Native\Desktop\Enums\ThermalStatesEnum;
use Native\Desktop\Facades\PowerMonitor;

$state = PowerMonitor::getSystemIdleState(60);   // seconds threshold
// ACTIVE | IDLE | LOCKED | UNKNOWN

$seconds = PowerMonitor::getSystemIdleTime();

$thermal = PowerMonitor::getCurrentThermalState();
// UNKNOWN | NOMINAL | FAIR | SERIOUS | CRITICAL

if (PowerMonitor::isOnBatteryPower()) { /* … */ }
```

Events under `Native\Desktop\Events\PowerMonitor\` (not `Events\` directly, as some docs suggest):

| Event | Payload |
|---|---|
| `PowerStateChanged` | `$state` — a `PowerStatesEnum` |
| `SpeedLimitChanged` | `$limit` — percentage of max CPU speed currently allowed |
| `ThermalStateChanged` | `$state` — a `ThermalStatesEnum` |
| `ScreenLocked`, `ScreenUnlocked` | — |
| `UserDidBecomeActive`, `UserDidResignActive` | — |
| `Shutdown` | — |

These are what you use to be a good citizen on a laptop: pause polling on battery, back off when thermal state degrades, stop background work when the screen locks.
