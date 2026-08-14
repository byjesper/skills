# Setup, configuration and the development loop

## Contents
- [Requirements](#requirements)
- [Installing](#installing)
- [Publishing the Electron project](#publishing-the-electron-project)
- [The development loop](#the-development-loop)
- [App icons](#app-icons)
- [config/nativephp.php](#confignativephpphp)
- [php.ini directives](#phpini-directives)
- [The NativeAppServiceProvider](#the-nativeappserviceprovider)
- [Environment files](#environment-files)
- [Debugging](#debugging)

## Requirements

PHP 8.3 or 8.4, Laravel 11.x or 12.x, Node 22+, and Windows 10+ / macOS 12+ (Monterey) / Linux. Catalina and Big Sur were dropped in v2 along with the Electron 38 upgrade.

Developing inside a container or VM is possible but adds friction and manual steps to producing working builds. Laravel Herd is the smoothest path on macOS and Windows.

## Installing

```shell
composer require nativephp/desktop
php artisan native:install
```

`native:install` publishes `app/Providers/NativeAppServiceProvider.php` and `config/nativephp.php`, adds a `native:dev` script to `composer.json`, registers itself as a `post-update-cmd` so the environment stays current after `composer update`, and installs the Electron dependencies.

Run `native:install` again on every new machine and in CI. It is what puts the platform-specific dependencies in place; a checkout alone is not enough to build.

Before running natively, boot the app in a browser first. Exceptions during boot are far easier to read in a browser than in a window that fails to appear.

```shell
php artisan native:run
```

## Publishing the Electron project

To modify the Electron layer itself:

```shell
php artisan native:install --publish
```

This exports the Electron project to `{project-root}/nativephp/electron` and gives you full control. The trade-off is that `composer update` will want to refresh it, so your modifications need cherry-picking after upgrades. Only do this when a genuine need exists.

## The development loop

`native:run` produces a *debug build*: unsigned, unoptimised, dev tools available, terminal attached so logs stream live. Stop it with Ctrl-C; it also exits when you quit the app.

The important consequence is that your application code is **copied** into the runtime's build environment. Changes to PHP files do not appear in the running app until you restart it.

Hot reloading covers assets, not PHP. With Vite running (`npm run dev`) and the `@vite` directive in your layout, JS and CSS changes reload in the window. Which files trigger reloads depends on your Vite config.

```shell
composer native:dev   # runs native:run and npm run dev concurrently
```

Edit the script in `composer.json` if you want different behaviour.

### First run

On first run NativePHP creates the `appdata` folder (named after `APP_NAME` in development), creates `nativephp.sqlite` in your `database` folder, and migrates it. On subsequent dev runs migrations are **not** run automatically:

```shell
php artisan native:migrate
```

Changing `APP_NAME` creates a fresh `appdata` folder; the old one is left behind, not deleted.

## App icons

`native:run` and `native:build` look for these in `public/` and move them into place automatically:

| File | Use |
|---|---|
| `public/icon.png` | Main icon — desktop, Dock, app switcher. At least 512×512. |
| `public/icon.ico` | Optional Windows-specific icon |
| `public/icon.icns` | Optional macOS-specific icon |
| `public/IconTemplate.png` | Menu bar, non-retina |
| `public/IconTemplate@2x.png` | Menu bar, retina |

## config/nativephp.php

```php
return [
    'version' => env('NATIVEPHP_APP_VERSION', '1.0.0'),
    'app_id' => env('NATIVEPHP_APP_ID', 'com.nativephp.app'),
    'deeplink_scheme' => env('NATIVEPHP_DEEPLINK_SCHEME'),
    'author' => env('NATIVEPHP_APP_AUTHOR'),
    'copyright' => env('NATIVEPHP_APP_COPYRIGHT'),
    'description' => env('NATIVEPHP_APP_DESCRIPTION', 'An awesome app built with NativePHP'),
    'website' => env('NATIVEPHP_APP_WEBSITE', 'https://nativephp.com'),
    'provider' => \App\Providers\NativeAppServiceProvider::class,

    'cleanup_env_keys' => ['AWS_*', 'GITHUB_*', 'DO_SPACES_*', '*_SECRET', /* ... */],
    'cleanup_exclude_files' => [
        'content',
        'storage/app/framework/{sessions,testing,cache}',
        'storage/logs/laravel.log',
    ],

    'updater' => [ /* see build-and-distribute.md */ ],
    'queue_workers' => [ /* see data-and-processes.md */ ],

    // Optional build hooks
    'prebuild' => ['npm run build', 'php artisan optimize'],
    'postbuild' => ['npm run release'],
];
```

Notes on the fields that carry consequences:

`version` drives migrations on the user's machine. Production builds compare it against the installed version and migrate only on a change. Bump it for every build you distribute. Any format works; a monotonically incrementing number is easiest to reason about, and you can keep a separate user-facing SemVer if you prefer.

`app_id` is a reverse-domain identifier and determines the `appdata` folder name in production builds. Changing it orphans existing users' data.

`deeplink_scheme` registers a custom URL scheme (`myapp://some/path`) so other applications can open yours.

`cleanup_env_keys` accepts wildcards and is your last line of defence against shipping a secret.

`prebuild` / `postbuild` run in the project root; use them for asset compilation rather than remembering to run it manually.

## php.ini directives

Implement `ProvidesPhpIni` on the provider and return directives from `phpIni()`:

```php
namespace App\Providers;

use Native\Desktop\Contracts\ProvidesPhpIni;
use Native\Desktop\Facades\Window;

class NativeAppServiceProvider implements ProvidesPhpIni
{
    public function boot(): void
    {
        Window::open();
    }

    public function phpIni(): array
    {
        return [
            'memory_limit' => '512M',
            'max_execution_time' => '0',
            'max_input_time' => '0',
        ];
    }
}
```

## The NativeAppServiceProvider

The boot sequence is: Electron starts → `php artisan migrate` runs → `php artisan serve` starts the PHP server → `NativeAppServiceProvider::boot()` runs → the `Native\Desktop\Events\App\ApplicationBooted` event is dispatched.

`boot()` is where windows open, menus register, global hotkeys bind, and the menu bar is created. Listen for `ApplicationBooted` for work that should happen on every launch — conditional seeding, for example.

## Environment files

The `.env` is copied into the bundle. Anyone with the app can read it. See the security discussion in `build-and-distribute.md`; the short version is that per-installation credentials obtained at runtime beat shared secrets baked into a build, and anything that must be stored locally should go through `System::encrypt()`.

## Debugging

Builds fail across five layers: your Laravel app; the NativePHP commands; the bundled static PHP binary; Electron and its platform toolchain; the OS and architecture. Identify the layer before changing code.

**Verbose output.** `-v`, `-vv`, `-vvv` on `native:run` and `native:build` show what each stage is doing.

**Logs.** Builds write to `{appdata}/storage/logs/`. Artisan commands run in your development environment write to the usual `storage/logs/`.

**Know which copy is running.** Dev builds live inside `vendor`; prod builds in `dist`. The file you're editing is a third copy.

**Step outside the runtime.** Run the failing step directly — hit the app in a browser to see whether a 500 during boot is why no window appears.

**Verify the bundled PHP binary** matches your platform and runs at all. Note that the paths in the published docs describe an older layout: there is no `asarUnpack` in the v2 build config, and the binary is staged into the build path (`{desktop-package}/resources/build/php/`), which electron-builder copies to `Resources/build/` as plain files. So look under `build/php/`, not inside the asar:

```shell
# dev build — the staging directory inside the installed package
vendor/nativephp/desktop/resources/build/php/php -v

# prod build, macOS
nativephp/electron/dist/{os+arch}/YourApp.app/Contents/Resources/build/php/php -v

# prod build, Windows
nativephp\electron\dist\win-unpacked\resources\build\php\php.exe -v
```

If a path doesn't match on your version, don't guess — `find` for the binary from the bundle root, the same way you would for a leaked file:

```shell
find nativephp/electron/dist -name "php" -type f -perm -u+x
```

Running `-m` on whatever you find is also how you confirm the available extension set for the build you are actually shipping.

**Start clean.** Delete `dist`. Wipe the appdata directory. Refresh the database with `php artisan native:migrate:fresh` (destructive). Kill stray processes left behind by a failed build.

### The app starts but no window appears

A dock or taskbar icon means Electron launched, so the fault is downstream of it. Work through the lifecycle in order — Electron starts, `migrate` runs, `serve` starts, `boot()` runs — since a failure at any step before the last produces exactly this symptom, with no error dialog.

- **A failing migration** halts at step 2, before anything could open a window. Check `{appdata}/storage/logs/`, then try `native:reset --with-app-data`.
- **A 500 during boot** leaves the shell process running with nothing to render. Hit the app's entrypoint in a browser to see it.
- **`boot()` never ran**: confirm `config('nativephp.provider')` points at your actual provider class, and that it really calls `Window::open()`.
- **The window opened off-screen.** `rememberState()` restores the last position, which may be on a monitor that is no longer attached. Nothing is wrong and no log will say so. Clear appdata, or temporarily drop `rememberState()`, to rule it out — cheap enough to check early.
- **A menu bar app hides the dock icon by default**, so the reverse symptom (no icon at all) may be intentional; see `showDockIcon()`.

Appdata locations:

| Platform | Location |
|---|---|
| macOS | `~/Library/Application Support` |
| Linux | `$XDG_CONFIG_HOME` or `~/.config` |
| Windows | `%APPDATA%` |

`php artisan native:reset [--with-app-data]` automates much of the cleanup.
