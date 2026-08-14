# Building, signing, publishing and updating

## Contents
- [Before you build: security review](#before-you-build-security-review)
- [Versioning](#versioning)
- [Building](#building)
- [Cross-compilation](#cross-compilation)
- [Code signing: macOS](#code-signing-macos)
- [Code signing: Windows](#code-signing-windows)
- [Publishing](#publishing)
- [The auto-updater](#the-auto-updater)
- [What happens on the user's machine](#what-happens-on-the-users-machine)
- [Custom PHP binaries](#custom-php-binaries)

## Before you build: security review

Once a build leaves your machine you cannot take it back. Walk this list before every release.

**Everything in the project directory is copied into the bundle.** Tracked, untracked, and `.gitignore`d files alike — Git is not consulted during packaging. Working notes, scratch exports, database dumps, fixture data pulled from production, credentials in a `.txt` file someone left in the repo: all of it ships, readable by anyone who receives the app.

`.gitignore` is not an exclusion mechanism. `cleanup_exclude_files` in `config/nativephp.php` is, and it takes glob patterns:

```php
'cleanup_exclude_files' => [
    'content',
    'storage/app/framework/{sessions,testing,cache}',
    'storage/logs/laravel.log',
    'notes',                    // your additions
    'storage/app/client-data/*',
],
```

Two things make this worth more than a config edit. First, if the data is genuinely sensitive — personal data, client records, anything with a regulatory obligation attached — moving it out of the project root entirely is stronger than an exclusion rule that a typo silently defeats. Distributing personal data inside an application binary is a reportable breach under GDPR and equivalent regimes, not merely an untidy build.

Second, **verify against the artifact, not the config — and search the whole bundle, not the asar.**

The intuitive check is wrong, and wrong in the dangerous direction. Your PHP application does not live in `app.asar`; that holds the Electron shell and its `node_modules`. The build config copies your app via electron-builder's `extraResources`, which is never asar-packed, so the Laravel application sits beside the asar as **plain files**. Measured on a real project: an 8 MB asar next to 112 MB of unpacked application. An `asar list … | grep` on a build that is actively leaking returns nothing and hands you a false all-clear.

Search from the bundle root instead, since the sub-path differs per platform (macOS puts it under `Contents/Resources/build/app/`, Windows and Linux under `resources/build/app/`):

```shell
find nativephp/electron/dist/*/YourApp.app -path "*notes*"
grep -rl "a-string-that-appears-only-in-that-folder" nativephp/electron/dist/*/YourApp.app 2>/dev/null
```

A canary makes the grep reliable: put a unique marker string in the sensitive file so you're matching content rather than guessing at filenames someone may have renamed.

Note also that electron-builder's own filter on that copy excludes only `.git`. Everything else that reached the staging directory ships. `cleanup_exclude_files` is the only lever you have, which is why verifying its effect matters more than reading it back.

**The `.env` ships too.** Treat the environment as hostile. Prefer credentials generated or collected at first run over shared secrets. If your app talks to an API you control, use a real auth protocol — OAuth2 with short-lived tokens (under 48 hours) and high entropy — rather than a static key every installation shares. Always HTTPS.

Check `cleanup_env_keys` covers everything sensitive; it accepts wildcards. The defaults already strip `AWS_*`, `GITHUB_*`, `DO_SPACES_*`, `*_SECRET`, the updater path, and the Apple notarization variables.

**User-supplied API keys** — keys your users connect to third-party services — must be stored encrypted via `System::encrypt()` and decrypted only at the moment of use.

**File access.** Your PHP runs with the user's privileges and can touch anything they can. Confine reads and writes to appdata and the user's own directories via the provided disks.

**Don't defeat the internal auth.** NativePHP runs two servers — one for PHP, one for the runtime's native hooks — and bridges them with authenticated HTTP using a pre-shared key regenerated on every launch. It's what stops other local software, or a phishing page in the user's browser, from driving your app's native APIs. Bypassing it opens your app to trivial HTTP attacks. The global `PreventRegularBrowserAccess` middleware, applied automatically in builds, is part of the same defence.

**The bundled PHP binary** is executable by anything on the machine with permission to run it. That's an RCE vector inherent to shipping a runtime, largely unmitigable, and worth knowing about so you can answer users who ask.

## Versioning

Bump `version` in `config/nativephp.php` (or `NATIVEPHP_APP_VERSION`) for **every** build you distribute.

Migrations run on the user's machine only when the version differs from the installed one. A forgotten bump means a schema change that never lands, which surfaces later as inexplicable errors on user machines. Any format works; a plain incrementing number is easiest to keep straight, with a separate user-facing SemVer if you want one.

## Building

```shell
php artisan native:build
```

Builds for the current platform and architecture, compiling your app plus the Electron runtime into a single executable and attempting to sign and notarize it. Output lands in `nativephp/electron/dist` (moved from `dist` in v1).

Asset compilation belongs in the build hooks so it can't be forgotten:

```php
'prebuild' => ['npm run build', 'php artisan optimize'],
'postbuild' => ['npm run release'],
```

These run in the project root; add as many commands as needed.

Build and test on each platform you intend to support before publishing. Cross-compiled artifacts especially deserve a real run on the target OS.

## Cross-compilation

```shell
php artisan native:build win     # mac | win | linux | all
php artisan native:build win x64 # optional arch: x64 | arm64
```

Not every combination works from every host. Building Windows binaries on Linux needs wine, and NSIS wants 32-bit wine for x64 apps:

```bash
dpkg --add-architecture i386
apt-get -y update
apt-get -y install wine32
```

## Code signing: macOS

Provide these when running `native:build`; they're stripped from the bundle automatically:

```dotenv
NATIVEPHP_APPLE_ID=developer@abcwidgets.com
NATIVEPHP_APPLE_ID_PASS=app-specific-password
NATIVEPHP_APPLE_TEAM_ID=8XCUU22SN2
```

`NATIVEPHP_APPLE_ID_PASS` is an app-specific password, not your Apple ID password.

Without proper notarization the app runs only on the machine that built it; everywhere else macOS reports that it is damaged and can't be opened. That message means notarization, not corruption — it's the single most common false alarm in NativePHP distribution.

Auto-updates on macOS require a signed app.

## Code signing: Windows

Two options.

**Azure Trusted Signing** (recommended — no local certificate management):

```dotenv
AZURE_TENANT_ID=your-tenant-id
AZURE_CLIENT_ID=your-client-id
AZURE_CLIENT_SECRET=your-client-secret

NATIVEPHP_AZURE_PUBLISHER_NAME=your-publisher-name          # the CN from your Identity Validation Request
NATIVEPHP_AZURE_ENDPOINT=https://eus.codesigning.azure.net/ # region endpoint for your certificate
NATIVEPHP_AZURE_CERTIFICATE_PROFILE_NAME=your-certificate-profile
NATIVEPHP_AZURE_CODE_SIGNING_ACCOUNT_NAME=your-code-signing-account
```

Two of these are commonly confused: the certificate profile name is not the Trusted Signing Account name, and the code signing account name is the account shown in Azure Trusted Signing, not your app registration's display name.

These credentials are stripped from the built app automatically.

**Traditional certificates** follow the Electron Forge Windows code-signing guide.

The build output tells you which path was taken: "Signing with Azure Trusted Signing (beta)" versus "Signing with signtool.exe".

## Publishing

```shell
php artisan native:publish
php artisan native:publish win
```

Same as building, plus uploading the artifacts to your configured updater provider. Bump the version first.

With the GitHub provider, create a **draft release** before publishing. Set the tag to your `version` value prefixed with `v`; the title can be anything. Each `native:publish` attaches artifacts to that draft, replacing them if you rebuild before tagging.

## The auto-updater

Users update without downloading anything manually: the updater checks a remote location, downloads a newer version if there is one, and swaps the application files.

It runs only in production builds. On macOS, only for signed apps.

**This is the inverse of the usual trap, and worth pausing on.** Most NativePHP surprises are things that work in development and fail in a build. The updater is the opposite: it does not run in development at all, so none of it — the check, the download, the signature validation, your `Error` listener, your restart prompt — is exercisable on your own machine by normal means. The first observation of the update path is on a stranger's computer, and a broken one is silent (see below).

Budget for that. Test it by building two versions with different `version` values, publishing the first, installing it, then publishing the second and relaunching. That round trip is the only honest test, and it is worth doing once before your first real release rather than discovering the problem when the fix has to reach users who cannot receive updates.

Providers: GitHub Releases (`github`), Amazon S3 (`s3`), DigitalOcean Spaces (`spaces`).

### What it actually does at runtime

Verified against the Electron plugin source, because the behaviour here is easy to assume wrongly:

- The updater starts **once**, during Electron's bootstrap, guarded by `updater.enabled === true` — a strict comparison, so a truthy-but-not-boolean config value disables it with no error.
- It calls `checkForUpdatesAndNotify()`. One check per launch. **There is no periodic re-check and no retry logic in NativePHP's layer at all.** An app left open for weeks checks once, on the day it opened.
- The `AndNotify` variant posts its own OS notification when an update is downloaded. If you're building custom update UI, that's the notification you didn't write.
- `startAutoUpdater()` runs *before* the PHP app boots. Whether Laravel is listening in time for `CheckingForUpdate`, or a fast `UpdateNotAvailable` or `Error`, is a race. A listener that "never fires" is usually losing it.

### When a download fails

The `Error` event fires and nothing else happens — no retry within the session. The next launch performs a fresh check and, if the update is still available, downloads again. `electron-updater` caches partial downloads and can resume, so transient network failures usually resolve themselves on their own.

The case to design for is the one that doesn't: a corrupted cached partial, a provider misconfiguration, an expired token on a private repo, a signature that won't validate. Those fail identically every launch, **silently**, and the user sits on a stale version indefinitely while you receive no signal.

Subscribing to `Error` is therefore not optional in a shipped app:

```php
Event::listen(\Native\Desktop\Events\AutoUpdater\Error::class, function ($event) {
    Log::warning('Updater failed', ['message' => $event->message]);
    // plus something the user can see — a menu bar label, a settings badge
});
```

Pair it with `UpdateDownloaded` for a "restart to update" prompt, and give long-lived apps a manual `AutoUpdater::checkForUpdates()` behind a menu item or a schedule, since the automatic check won't come round again.

Note that the resume behaviour belongs to `electron-updater`, not NativePHP, and can change with a dependency bump. The once-per-launch check and the absence of NativePHP-level retry are properties of NativePHP itself.

### Configuration

```php
'updater' => [
    'enabled' => env('NATIVEPHP_UPDATER_ENABLED', true),
    'default' => env('NATIVEPHP_UPDATER_PROVIDER', 'spaces'),

    'providers' => [
        'github' => [
            'driver' => 'github',
            'repo' => env('GITHUB_REPO'),
            'owner' => env('GITHUB_OWNER'),
            'token' => env('GITHUB_TOKEN'),
            'vPrefixedTagName' => env('GITHUB_V_PREFIXED_TAG_NAME', true),
            'private' => env('GITHUB_PRIVATE', false),
            'autoupdate_token' => env('GITHUB_AUTOUPDATE_TOKEN'), // read-only token for private repos
            'channel' => env('GITHUB_CHANNEL', 'latest'),
            'releaseType' => env('GITHUB_RELEASE_TYPE', 'draft'),
        ],
        's3' => [
            'driver' => 's3',
            'key' => env('AWS_ACCESS_KEY_ID'),
            'secret' => env('AWS_SECRET_ACCESS_KEY'),
            'region' => env('AWS_DEFAULT_REGION'),
            'bucket' => env('AWS_BUCKET'),
            'endpoint' => env('AWS_ENDPOINT'),
            'public_url' => env('AWS_PUBLIC_URL'),
            'path' => env('NATIVEPHP_UPDATER_PATH', null),
        ],
        'spaces' => [
            'driver' => 'spaces',
            'key' => env('DO_SPACES_KEY_ID'),
            'secret' => env('DO_SPACES_SECRET_ACCESS_KEY'),
            'name' => env('DO_SPACES_NAME'),
            'region' => env('DO_SPACES_REGION'),
            'path' => env('NATIVEPHP_UPDATER_PATH', null),
        ],
    ],
],
```

Disable with `NATIVEPHP_UPDATER_ENABLED=false`.

Manual control:

```php
use Native\Desktop\Facades\AutoUpdater;

AutoUpdater::checkForUpdates();   // downloads automatically if one is found
AutoUpdater::downloadUpdate();    // absent from the website docs, but in the facade docblock
AutoUpdater::quitAndInstall();    // quits, installs, relaunches
```

Calling `checkForUpdates()` twice downloads the update twice — guard it rather than calling it on every page load. `quitAndInstall()` is optional; a downloaded update applies on the next start anyway, so prompting the user rather than forcing a restart is usually kinder.

Events under `Native\Desktop\Events\AutoUpdater\`:

| Event | Notes |
|---|---|
| `CheckingForUpdate` | |
| `UpdateAvailable` | download starts automatically |
| `UpdateNotAvailable` | |
| `DownloadProgress` | `total`, `delta`, `transferred`, `percent`, `bytesPerSecond` |
| `UpdateDownloaded` | `version`, `downloadedFile`, `releaseDate`, `releaseNotes`, `releaseName` |
| `UpdateCancelled` | |
| `Error` | `error` |

`DownloadProgress` and `UpdateDownloaded` are what you wire to a progress indicator and an "update ready, restart when convenient" prompt.

## What happens on the user's machine

First run: the appdata folder is created, named from `nativephp.app_id`; `{appdata}/database/database.sqlite` is created; migrations run.

Every subsequent run: NativePHP compares the app version to the installed one and migrates if it changed. Hence the versioning discipline above.

## The bundled PHP: which extensions you actually get

This is the single most common way a package works perfectly in development and fails in the app. Your machine's PHP is not the PHP that ships. Dev builds use the static binary too, so `native:run` catches it — but browser testing and your test suite do not.

The bundled binaries are statically compiled with a deliberately minimal, cross-platform-consistent set:

```
bcmath  bz2  ctype  curl  dom  fileinfo  filter  gd  iconv  intl
mbstring  mbregex  opcache  openssl  pdo  pdo_sqlite  phar  session
simplexml  sockets  sodium  sqlite3  tokenizer  xml  zip  zlib
```

Notable absences: **`xmlreader` and `xmlwriter`** (separate extensions from `xml`, `dom` and `simplexml` — having those three does not give you these two), `pdo_mysql` and `pdo_pgsql`, `ldap`, `imap`, `ffi`, `pcntl`, `posix`, `exif`, `soap`, `redis`, `imagick`, `xdebug`. The canonical list is `php-extensions.txt` in the `php-bin` repo; verify against the actual binary with `-m`, locating it as shown in `setup-and-configuration.md`.

### Check third-party packages before you commit to them

Read the package's `composer.json` `require` block for `ext-*` entries and compare against the list above. Composer will happily install it — it resolves against your system PHP, not the binary that ships.

Packages that do **not** work out of the box, and the usual reason:

| Package | Blocked on |
|---|---|
| `phpoffice/phpspreadsheet` | `ext-xmlreader`, `ext-xmlwriter` |
| `openspout/openspout` (and `spatie/simple-excel`, which wraps it) | `ext-xmlreader` |
| Anything needing MySQL or Postgres directly | `pdo_mysql`, `pdo_pgsql` |
| Signal handling / process control libraries | `pcntl`, `posix` |

When a package is blocked, work through these in order:

1. **Change the format.** If you control the input, CSV needs no extension at all and `dompdf`/`fpdf` cover PDF output without `xmlwriter`.
2. **Use the bundled Node runtime.** Your app already ships Node, and `ChildProcess::node()` runs a script with it — no extra binary, no lost support. This is usually the best answer; `data-and-processes.md` has a worked spreadsheet recipe, including the output-buffering and async pitfalls.
3. **Do it with what you have.** An `.xlsx` is a zip of XML, and `zip`, `dom` and `simplexml` are all present, so parsing `sharedStrings.xml` and `xl/worksheets/sheet1.xml` yourself is viable for files of known shape. Not worth it for arbitrary user uploads.
4. **Move the work off the device** to an API you control, if the app is online anyway.
5. **Build a custom binary** — last resort; see below for what it costs.

If an extension seems like it belongs in the defaults, a feature request against the `desktop` repo is a better move than a custom build.

## Custom PHP binaries

If you do need your own, build single-file executables with all dependencies statically linked (NativePHP uses `static-php-cli`) and point at them:

```dotenv
NATIVEPHP_PHP_BINARY_PATH=/path/to/your-app/bin/
```

The directory must mirror the `php-bin` layout: platform first (`linux`, `mac`, `win`), architecture as a subfolder (`x64`, `arm64`, `x86`). Binaries named `php` or `php.exe`, zipped as `php-{MAJOR}.{MINOR}.zip`:

```shell
zip php-8.3.zip php                                                         # macOS / Linux
powershell Compress-Archive -Path "php.exe" -DestinationPath "php-8.3.zip"  # Windows
```

You only need the platforms you support and the PHP version you require.

Two consequences worth stating plainly: your build pipeline becomes a supply-chain risk you now own, and apps using custom binaries are not eligible for support via GitHub Issues.
