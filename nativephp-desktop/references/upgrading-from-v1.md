# Upgrading from NativePHP v1 to v2

v2 is an architecture overhaul and security release. The package moved repository and name, and the root namespace changed with it. Nearly all NativePHP material written before v2 — blog posts, Stack Overflow answers, tutorials, and quite possibly your own recollection — uses the v1 namespace and will produce class-not-found errors under v2.

## Change summary

| Impact | Change |
|---|---|
| High | Package renamed to `nativephp/desktop` |
| High | Root namespace `Native\Laravel` → `Native\Desktop` |
| High | macOS Catalina and Big Sur no longer supported |
| Medium | `native:serve` renamed to `native:run` |
| Medium | `nodeIntegration` now defaults to `false` |
| Low | Build output moved to `nativephp/electron/dist` |
| Low | Electron backend now published via `native:install --publish` |

## Packages

```json
"require": {
    "nativephp/electron": "^1.3",   // remove
    "nativephp/laravel": "^1.3",    // remove
    "nativephp/desktop": "^2.0"     // add
}
```

Both old packages go — `nativephp/laravel` too, if present. The new package declares a conflict with `nativephp/electron`, `nativephp/laravel` and `nativephp/mobile`, so Composer will object if any linger.

```shell
composer update
php artisan native:install
```

After this, `native:install` is registered as a `post-update-cmd` and runs itself after future `composer update`s.

## Namespace rewrite

Replace every occurrence of `Native\Laravel` with `Native\Desktop`:

```php
use Native\Laravel\Facades\Window;   // v1
use Native\Desktop\Facades\Window;   // v2
```

This is mechanical and safe to do with a find-and-replace across the project, but check these places specifically, since they're easy to miss in a grep of `app/`:

- `app/Providers/NativeAppServiceProvider.php`
- `config/nativephp.php` (the `provider` key, and any imported class references)
- Event listener registrations in `EventServiceProvider` or `AppServiceProvider`
- Livewire `#[On('native:…')]` attributes and `$listeners` arrays, where the class name may be a **string literal** rather than a `::class` reference
- JavaScript `Native.on('Native\\Laravel\\Events\\…')` handlers — these are strings, so nothing will fail loudly; the listener just silently never fires
- Tests

The Livewire and JavaScript cases deserve extra attention precisely because they fail quietly. A window event listener that stops firing looks like a NativePHP bug rather than a stale string.

While you're rewriting, note that a few namespaces the v1 docs used are wrong in current v2 docs too — see the verified list in `SKILL.md`. In particular `SettingChanged` lives under `Events\Settings\`, power events under `Events\PowerMonitor\`, and `QueueConfig` under `DataObjects\`.

## `native:serve` → `native:run`

`native:serve` is deprecated. The rename gives symmetry with the mobile package.

Update the `native:dev` script in `composer.json`, plus any CI configuration, Makefile, README, or shell alias that invokes it.

## `nodeIntegration` defaults to false

This is the security change most likely to break a working v1 app. Renderer code that reached for Node APIs — `require`, `process`, `fs` — will now fail.

Re-enable it deliberately, per window, only where genuinely needed:

```php
Window::open()->webPreferences(['nodeIntegration' => true]);
```

Before doing that, consider whether the functionality could move to PHP instead. `nodeIntegration` gives your renderer full Node access, which means any content rendered in that window — including anything remote — gets it too. Moving filesystem or process work into a Laravel route or a `ChildProcess` is usually both safer and less code.

`sandbox`, `preload` and `contextIsolation` are now locked and cannot be overridden at all.

## macOS support

Catalina (10.15) and Big Sur (11) support is gone, following Electron 38 and Apple's own supported-version policy. Check your deployment targets before upgrading; most users won't be affected, but if you have telemetry on installed OS versions, look at it first.

## Build output location

Builds now land in `nativephp/electron/dist` rather than `dist` at the project root. Update `.gitignore`, CI artifact paths, and any release scripts that glob the old location.

## Modifying the Electron backend

If your v1 setup involved editing the Electron project directly:

```shell
php artisan native:install --publish
```

This exports the Electron project to `{project-root}/nativephp/electron` and modifies your `post-update-cmd` to keep it current. Your own modifications will need cherry-picking after each `composer update` — worth confirming the customisation is still necessary before carrying it forward.

## After upgrading

Run through the checklist rather than assuming a green test suite means success:

1. `php artisan native:run` and confirm windows open, menus render, the menu bar appears.
2. Exercise every window event listener, especially the Livewire and JavaScript ones.
3. Check anything that relied on `nodeIntegration`.
4. Run a production build for each platform you support and test the built app, not just the dev build.
5. Verify migrations against a copy of a real user database — the version-change migration path is where an upgrade can destroy data.
