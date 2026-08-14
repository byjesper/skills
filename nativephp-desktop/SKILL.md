---
name: nativephp-desktop
description: Build, debug, and ship desktop applications with NativePHP for Desktop 2.x (the nativephp/desktop package that wraps a Laravel app in Electron; written against 2.2.1). Use this skill whenever the work touches NativePHP, the Native\Desktop namespace, NativeAppServiceProvider, or artisan commands like native:install, native:run, native:build, native:publish or native:migrate — and also when someone wants to turn a Laravel app into a desktop app, build a macOS menu bar app in PHP, package Laravel with Electron, add native windows, tray icons, system notifications, global hotkeys, native file dialogs or auto-updates to a PHP app, or upgrade from NativePHP v1 (nativephp/electron, nativephp/laravel) to v2. Use it even when NativePHP is not named, if the goal is a distributable desktop app written in Laravel.
---

# NativePHP for Desktop 2.x

## Version this was written against

| | |
|---|---|
| `nativephp/desktop` | **2.2.1** (released 2026-05-29), plus unreleased `main` at commit `653d1867d973`, 2026-06-22 |
| `nativephp/php-bin` | **1.2.0** (released 2026-05-21) — the source of the bundled extension list |
| Documentation | nativephp.com desktop v2 docs, retrieved 2026-08-08 |

API names, class namespaces, config keys and build paths here were checked against that source, not taken from the docs alone — several published examples are stale and are corrected in place, with the correction called out where it matters.

Two consequences worth holding onto. Anything below stated as a **filesystem path, build-output location, or extension list** is the most perishable material in this skill: it is accurate for the versions above and can move under a dependency bump, so each of those claims is paired with a command to check it against the build in front of you. Prefer running the check over trusting the value. And because part of this reflects unreleased `main`, a project pinned to an older 2.x may lack something described here — check the package's own CHANGELOG when a method appears to be missing rather than assuming a typo.

NativePHP wraps a normal Laravel application in an Electron shell and ships a statically-compiled PHP binary alongside it, so the whole thing installs as one native app. The Laravel part is unremarkable — routes, Blade, Eloquent, queues, all as usual. What trips people up is that the *runtime assumptions* of a web app no longer hold, and those assumptions are baked into most people's instincts.

Work through this skill in three moves: get the mental model right, pick the reference file for the task at hand, then write code against the verified API names at the bottom of this file.

## The mental model that actually matters

A Laravel app on a server is one deployment, many users, a trusted environment, and code you can edit in place. A NativePHP app inverts all four. Most bugs people hit are a consequence of one of these:

**Your code is copied into the build, not served from your working directory.** `native:run` copies your entire app into the Electron build environment. Editing a PHP file changes the copy in your IDE, not the copy that's running. Restart the process to see PHP changes. Vite-driven JS/CSS hot reload works fine because Vite serves those separately. This also means there are two or three copies of your app on disk at any time (working dir, dev build inside `vendor`, prod build in `dist`) — when debugging, always know which one is executing.

**`storage_path()` no longer points into your project.** NativePHP rewrites it to the platform `appData` directory. That's where the SQLite database, logs and caches live on the user's machine, and it survives app updates. Anything you want to persist for the user goes there; anything you want the user to *find* goes to one of the named disks (`desktop`, `documents`, `downloads`, `user_home`, …).

**Your whole project directory ships, not just the tracked files.** The bundle is a copy of the application directory — tracked files, untracked files, and anything in `.gitignore` alike. Git plays no part in packaging, so an ignored folder of scratch data, credentials or client records travels to every user unless you exclude it explicitly via `cleanup_exclude_files`. It also ships as **plain files**, not inside `app.asar` — the asar holds only the Electron shell — so anyone can read it with a file browser, and any verification that inspects the asar will miss it entirely.

**The `.env` is the best-known case of that, not a special one.** It sits inside the bundle, readable by anyone who has the app. Treat the whole environment as hostile: secrets should be generated or collected at runtime and stored via `System::encrypt()`, not baked into a build. `cleanup_env_keys` strips matching keys at bundle time — extend it rather than trusting yourself to remember.

**Migrations run on version change, not on deploy.** In production, NativePHP compares `config('nativephp.version')` against the installed version and migrates the user's database only if it differs. Forgetting to bump the version means users silently never get the new schema. In development, migrations of the dev database are *not* automatic at all — run `php artisan native:migrate` yourself.

**The app is the server.** There is no ops team, no rollback, no shared cache. A migration that destroys data destroys the user's only copy. Test prod builds before releasing.

**The PHP that ships is not the PHP on your machine.** A statically-compiled binary with a fixed, minimal extension set runs your code in both dev builds and production. Composer resolves against your system PHP, so a package can install cleanly, pass its tests, work in the browser, and then fail under `native:run`. Before adding any dependency, check its `ext-*` requirements against the list in `references/build-and-distribute.md` — `xmlreader` and `xmlwriter` in particular are absent, which rules out the common spreadsheet libraries.

**Everything native is bootstrapped from one place.** `app/Providers/NativeAppServiceProvider::boot()` runs after Electron is up. Windows, menus, the menu bar, global hotkeys — all registered there. Calling native facades from a route or controller works too, but boot-time setup belongs in the provider.

## Getting oriented in an existing project

Before writing anything, check what you're dealing with. `composer.json` requiring `nativephp/electron` or `nativephp/laravel` means v1, and the namespace is `Native\Laravel` — read `references/upgrading-from-v1.md` first. `nativephp/desktop` means v2 and `Native\Desktop`. Then read `app/Providers/NativeAppServiceProvider.php`, which is the app's native entry point, and `config/nativephp.php`.

If someone is starting fresh, requirements are PHP 8.3+, Laravel 11 or 12, Node 22+, and Windows 10+ / macOS 12+ / Linux. macOS Catalina and Big Sur are not supported in v2.

## Which reference to read

Read the one that matches the task. They're independent; don't read all of them.

| File | Covers |
|---|---|
| `references/setup-and-configuration.md` | Installing, `config/nativephp.php`, the dev loop, hot reloading, app icons, php.ini, debugging a broken build |
| `references/windows-and-menus.md` | `Window`, `MenuBar`, `Menu`, `Dialog`, `Alert`, `Notification`, context menus, window events |
| `references/system-apis.md` | `App`, `System` (encryption, TouchID, printing, theme, timezone), `Screen`, `Shell`, `Clipboard`, `GlobalShortcut`, `PowerMonitor`, `Settings` |
| `references/data-and-processes.md` | SQLite, migrations in dev vs prod, storage disks and `extras/`, reading spreadsheets and other formats the bundled PHP can't parse, queues and workers, child processes, broadcasting to JS and Livewire |
| `references/build-and-distribute.md` | `native:build`, cross-compilation, code signing on macOS and Windows, `native:publish`, the auto-updater, security review before shipping |
| `references/testing.md` | The `::fake()` doubles and their assertions — needed because tests otherwise try to reach a running Electron process |
| `references/upgrading-from-v1.md` | Package rename, namespace rewrite, `native:serve` → `native:run`, `nodeIntegration` default change, dropped macOS versions |

## Verified API names

The published docs contain several stale namespaces. These were checked against the `nativephp/desktop` source; prefer them over anything you recall or read elsewhere. Getting one of these wrong produces a "class not found" that looks like a broken install.

Facades, all under `Native\Desktop\Facades\`:
`Alert`, `App`, `AutoUpdater`, `ChildProcess`, `Clipboard`, `ContextMenu`, `Dock`, `GlobalShortcut`, `Menu`, `MenuBar`, `Notification`, `PowerMonitor`, `Process`, `QueueWorker`, `Screen`, `Settings`, `Shell`, `System`, `Window`.

Not facades — instantiate via a static constructor on the class itself:

```php
use Native\Desktop\Dialog;        // Dialog::new()->title(...)->open()
use Native\Desktop\ProgressBar;   // ProgressBar::create($maxSteps)
```

Easily-miscited classes:

```php
Native\Desktop\Events\Settings\SettingChanged          // NOT Events\Notifications\
Native\Desktop\Events\PowerMonitor\PowerStateChanged   // NOT Events\PowerStateChanged
Native\Desktop\Events\PowerMonitor\ThermalStateChanged
Native\Desktop\Events\Menu\MenuItemClicked
Native\Desktop\DataObjects\QueueConfig                 // NOT DTOs\ or Native\DTOs\
Native\Desktop\DataObjects\Printer
Native\Desktop\Enums\SystemThemesEnum
Native\Desktop\Enums\SystemIdleStatesEnum
Native\Desktop\Enums\ThermalStatesEnum
Native\Desktop\Enums\PowerStatesEnum
Native\Desktop\Contracts\ProvidesPhpIni
```

Event namespaces group by subject: `Events\App\`, `Events\Windows\`, `Events\MenuBar\`, `Events\Menu\`, `Events\Notifications\`, `Events\Settings\`, `Events\PowerMonitor\`, `Events\ChildProcess\`, `Events\AutoUpdater\`.

## Artisan commands

```
native:install [--publish] [--force] [--installer=npm]   # also runs as post-update-cmd
native:run [--no-queue] [--no-focus] [--no-dependencies] # dev build; native:serve is the deprecated v1 name
native:build [os] [arch] [--publish]                     # os: mac|win|linux|all
native:publish [os] [arch]                               # build + upload to updater provider
native:migrate                                           # same signature as artisan migrate
native:migrate:fresh                                     # destructive
native:seed
native:reset [--with-app-data]
native:debug {output}
```

`composer native:dev` is a convenience script the installer adds, running `native:run` and `npm run dev` together.

## Working style

Prefer plain Laravel for anything that doesn't need the OS. A Livewire component and an Eloquent query are the same here as anywhere; reach for a NativePHP facade only at the boundary where you actually need a window, the filesystem outside the sandbox, a notification, or a process.

When a build fails, resist the urge to change application code first. The layers are: your Laravel app, the NativePHP commands, the bundled static PHP binary, Electron and its platform toolchain, and the OS. Run `native:run -vvv` or `native:build -vvv`, then work out which layer is actually failing — `references/setup-and-configuration.md` has the isolation steps.

When a change affects what gets shipped — a new dependency, a new secret, a migration, a version bump — say so explicitly. Those are the changes with consequences that only show up on a stranger's machine.
