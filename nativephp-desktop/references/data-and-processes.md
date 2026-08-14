# Data, files, background work and events

## Contents
- [Databases](#databases)
- [Files and storage disks](#files-and-storage-disks)
- [Bundling extra resources](#bundling-extra-resources)
- [Choosing a background mechanism](#choosing-a-background-mechanism)
- [Queues](#queues)
- [Child processes](#child-processes)
- [File formats the bundled PHP can't parse](#file-formats-the-bundled-php-cant-parse)
- [Broadcasting](#broadcasting)

## Databases

SQLite only. Anything else would either bloat the bundle or assume something about the user's machine you can't assume. Use PDO or Eloquent exactly as usual.

NativePHP handles configuration itself: it switches the app to SQLite when building, creates the database file in the user's appdata directory, points the app at it, and migrates on version change.

### Development vs production

In development the app uses `nativephp.sqlite` in the build directory. NativePHP forces this while running under Electron so your other local SQLite databases are untouched. Migrations here are **not** automatic:

```shell
php artisan native:migrate        # same signature as artisan migrate
php artisan native:migrate:fresh  # destructive
php artisan native:seed
```

Run these before starting the app rather than while it's running, unless you know the app tolerates it.

In production the database is `{appdata}/database/database.sqlite`. On each launch NativePHP compares the app version against the installed one and migrates if they differ. **This means a migration that isn't accompanied by a version bump never runs on a user's machine**, and it also means a bad migration runs against the user's only copy of their data. Test migrations against a production build before releasing.

Note the SQLite specifics that apply to migrations — foreign key constraints, limited `ALTER TABLE` support, and so on. Laravel's SQLite caveats all apply.

### Seeding on first run

Production builds don't seed. Do it from something that runs on every boot, guarded by a check:

```php
use App\Models\Config;
use Illuminate\Support\Facades\Artisan;

if (Config::where('seeded', true)->count() === 0) {
    Artisan::call('db:seed');
}
```

### When not to use the database

For small amounts of simple state, a file is often better. Critically, keep *bootstrap* state — the flags your app needs to start at all — out of the database. If the database corrupts and your startup path depends on it, the app won't open and the user can't recover. In a file, you can tell them to delete it and restart without losing their data.

## Files and storage disks

NativePHP rewrites `Application::storagePath()` — and therefore `storage_path()` and `app()->storagePath()` — to Electron's `appData` path, which differs per OS. The `local` disk follows it, landing in `{appdata}/storage/app`.

Use appdata for anything that must survive updates and uninstall/reinstall cycles, and that the user doesn't need to open directly: configuration, settings, caches, logs, the SQLite database.

For files the user *should* be able to find, NativePHP registers disks pointing at the platform-specific user directories:

```php
Storage::disk('user_home')->get('file.txt');
Storage::disk('desktop')->get('file.txt');
Storage::disk('documents')->get('file.txt');
Storage::disk('downloads')->get('file.txt');
Storage::disk('music')->get('file.txt');
Storage::disk('pictures')->get('file.txt');
Storage::disk('videos')->get('file.txt');
Storage::disk('recent')->get('file.txt');
Storage::disk('extras')->get('file.txt');
```

If your app already defines disks with these names, NativePHP overrides them.

Your PHP process runs with the logged-in user's privileges, so it can read and write anywhere they can. Restrict yourself to appdata and the user's own directories. Modern operating systems prompt the user to grant access to home subdirectories on first touch, and an app reaching somewhere unexpected reads as malware.

Cloud disks still work, but a desktop app loses connectivity far more often than a server does. Check for a connection before making network calls and handle failures without breaking the UI.

### Symlinks

Avoid them. The whole application is already accessible to the user, so the usual reasons don't apply, and Unix/Windows differences in symlink handling break packaging. Put files where they belong, or mount directories through the Storage facade.

## Bundling extra resources

Files placed in an `extras/` directory at the project root are bundled with the app:

```
your-project/
├── extras/
│   ├── my-tool.sh
│   ├── my-tool.exe
│   └── sample.csv
```

Access them through the read-only `extras` disk:

```php
$toolPath = Storage::disk('extras')->path('my-tool.exe');
```

Anything written there is overwritten on update, so treat it as immutable. This is the right home for helper binaries you spawn as child processes.

## Choosing a background mechanism

Four options, in rough order of preference:

**Do it synchronously.** On a desktop app the database is local and there's no network hop. Many things that would warrant a queue on a server are fast enough inline, and a spinner is clearer to the user than a job that silently fails somewhere. Start here.

**Queues** for genuinely slow Laravel work — heavy processing, network calls. Jobs persist in SQLite and resume when the app next starts.

**The scheduler** runs every minute inside the app, exactly as usual. Tasks scheduled for times when the app was closed are skipped, not caught up. Both the scheduler and the queue worker are tied to your app's lifetime, not the OS.

**Child processes** for long-running non-PHP programs you want to talk to over their lifetime.

`shell_exec` and `proc_open` still work but come with two hazards: they may block the calling script, and they may outlive your app as orphans, which is unpleasant for the user and hard to clean up without intervention. Prefer `ChildProcess`, which the runtime shuts down with the app.

## Queues

Nothing special is required to dispatch. The `jobs` table migration is created and migrated for you. One worker consuming the `default` queue boots automatically.

Configure additional workers in `config/nativephp.php`:

```php
'queue_workers' => [
    'one' => [],
    'two' => [],
    'three' => [
        'queues' => ['high'],
        'memory_limit' => 1024,
        'timeout' => 600,
        'sleep' => 3,
    ],
],
```

Defaults per worker: `queues => ['default']`, `memory_limit => 128`, `timeout => 60`, `sleep => 3`.

Each entry becomes a persistent child process whose alias is the array key. `sleep` is the pause when no jobs are waiting — lower is more responsive and burns more CPU, which on a laptop the user is holding is a real cost. Don't drop it below the default without a reason.

Managing workers at runtime:

```php
use Native\Desktop\DataObjects\QueueConfig;   // NOT DTOs\ — the docs are wrong about this
use Native\Desktop\Facades\QueueWorker;

$config = new QueueConfig(
    alias: 'manual',
    queuesToConsume: ['default'],
    memoryLimit: 1024,
    timeout: 600,
    sleep: 5,
);

QueueWorker::up($config);
QueueWorker::up(config: 'manual');   // by alias, if already in config
QueueWorker::down(alias: 'manual');
```

## Child processes

Spawning a child process is like running a shell command, except the runtime manages it and shuts it down gracefully when the app quits.

```php
use Native\Desktop\Facades\ChildProcess;

ChildProcess::start(
    cmd: 'tail -f storage/logs/laravel.log',
    alias: 'tail',
);
```

Every process needs a unique alias — that's how you address it later. `cmd` accepts a string or an array of arguments.

`start()` returns a `Native\Desktop\ChildProcess` instance immediately, but that does **not** mean the process started. Spawning is the OS's business and can fail. The return value is not a success signal and there is nothing useful to inspect on it — listen for `ProcessSpawned` to know it launched, and `StartupError` and `ErrorReceived` to find out why it didn't. A process that "silently does nothing" is almost always one whose failure nobody subscribed to.

### PATH is the usual culprit, and it lies to you in development

Reference binaries by absolute path unless you're certain they're on the user's `PATH` — and be aware that "on the PATH" means something different in a built app than under `native:run`.

Launched from a terminal, your app inherits that shell's environment, so `python3`, `ffmpeg` or anything else your dotfiles put on the `PATH` resolves fine. Launched by double-clicking in Finder or from the Start menu, a GUI application gets a minimal environment that does not include your shell's additions. The same `cmd` string therefore works all through development and fails for every user.

Absolute paths solve resolution but not the underlying assumption, which is that a particular interpreter exists on a stranger's machine at a version you didn't choose. Stronger options, in order:

1. **Use a runtime you already ship.** `ChildProcess::php()`, `ChildProcess::artisan()` and `ChildProcess::node()` use the bundled PHP and Node, which are guaranteed present and version-pinned.
2. **Bundle the tool** as a self-contained executable in `extras/` and spawn it by `Storage::disk('extras')->path(...)`.
3. **Depend on a system binary** only when it's genuinely universal, and detect it explicitly with a clear error rather than failing silently.

Remember too that arguments and quoting often differ between Windows and macOS/Linux.

```php
ChildProcess::start(
    cmd: ['tail', '-f', 'logs/laravel.log'],
    alias: 'tail',
    cwd: storage_path(),
    env: ['CUSTOM_ENV_VAR' => 'custom value'],
    persistent: true,   // restarted automatically if it crashes, supervisord-style
);
```

Convenience constructors that use the bundled runtimes:

```php
ChildProcess::php('path/to/script.php', alias: 'script');
ChildProcess::artisan('smtp:serve', alias: 'smtp-server');
ChildProcess::node(cmd: 'resources/js/watcher.js', alias: 'watcher');
```

`node()` uses the Node runtime shipped with your app, so JS files need no pre-compilation and dependencies don't have to be browser-bundled.

Managing them:

```php
$tail = ChildProcess::get('tail');
$all = ChildProcess::all();

$tail->stop();       ChildProcess::stop('tail');
$tail->restart();    ChildProcess::restart('tail');
$tail->message('Hello, world!');   ChildProcess::message('Hello, world!', 'tail');
```

`message()` writes to the process's STDIN; the format is whatever the program expects.

Stopping a persistent process stops it permanently until `start()` is called again — use `restart()` if you just want to bounce it.

### Child process events

Under `Native\Desktop\Events\ChildProcess\`:

| Event | Payload |
|---|---|
| `ProcessSpawned` | `$alias`, `$pid` (in Electron this is the helper process that spawns the real one) |
| `ProcessExited` | `$alias`, `$code` |
| `MessageReceived` | `$alias`, `$data` — from STDOUT |
| `ErrorReceived` | `$alias`, `$data` — from STDERR |
| `StartupError` | — |

```php
use Native\Desktop\Events\ChildProcess\MessageReceived;

Event::listen(MessageReceived::class, function (MessageReceived $event) {
    if ($event->alias === 'tail') {
        // …
    }
});
```

## File formats the bundled PHP can't parse

The static binary ships a fixed extension set (enumerated in `build-and-distribute.md`). The absences that bite hardest are **`xmlreader` and `xmlwriter`**, which rule out `phpoffice/phpspreadsheet` and `openspout/openspout` — and therefore `spatie/simple-excel`, which wraps openspout. Reading or writing `.xlsx` is the case that comes up constantly.

Composer resolves against your system PHP, so the package installs cleanly and works when you hit the app in a browser. It fails under `native:run` and in the build, because both use the static binary. Confirm what you actually have by running `-m` on the binary in your own build (see `setup-and-configuration.md` for locating it) before designing around this.

Four ways out, best first.

### 1. Use the bundled Node runtime

Your app already ships Node, and `ChildProcess::node()` runs a script with it — no extra binary, no custom PHP build, no lost support. Vendor SheetJS's standalone bundle next to a small script in `extras/`, which is bundled with the app:

```
extras/xlsx/xlsx.full.min.js
extras/xlsx/read.js
```

```js
// extras/xlsx/read.js
const XLSX = require('./xlsx.full.min.js')

try {
    const workbook = XLSX.readFile(process.argv[2])
    const sheet = workbook.Sheets[workbook.SheetNames[0]]
    process.stdout.write(JSON.stringify({
        ok: true,
        rows: XLSX.utils.sheet_to_json(sheet, { defval: null }),
    }))
} catch (error) {
    process.stdout.write(JSON.stringify({ ok: false, error: error.message }))
    process.exitCode = 1
}
```

```php
use Illuminate\Support\Facades\Storage;
use Native\Desktop\Facades\ChildProcess;

ChildProcess::node(
    cmd: [Storage::disk('extras')->path('xlsx/read.js'), $uploadedPath],
    alias: 'xlsx-read',
);
```

Three things determine whether this works in practice:

**It's asynchronous.** `node()` returns immediately and output arrives as events, so you cannot read a spreadsheet inline in a controller and render the result in the same response. Either dispatch the work and broadcast a custom event to update the UI when it finishes, or have the script write JSON to a temp file and read it on `ProcessExited`.

**Buffer STDOUT.** `MessageReceived` fires per chunk, and anything larger than a small sheet arrives in several. Accumulate keyed by alias and decode only on `ProcessExited`:

```php
Event::listen(MessageReceived::class, function ($event) {
    if ($event->alias === 'xlsx-read') {
        Cache::put('xlsx-read.buffer', Cache::get('xlsx-read.buffer', '').$event->data);
    }
});

Event::listen(ProcessExited::class, function ($event) {
    if ($event->alias === 'xlsx-read') {
        $result = json_decode(Cache::pull('xlsx-read.buffer', ''), true);
        // $event->code is the exit status; ErrorReceived carried anything on STDERR
    }
});
```

**Aliases must be unique**, so concurrent reads need distinct ones — suffix with a job or upload ID rather than reusing `xlsx-read`.

Note that `node()` forces `cwd` to `base_path()`; it isn't a parameter. Resolve paths through `Storage::disk('extras')->path()` rather than assuming a working directory.

### 2. Parse it yourself

An `.xlsx` is a zip of XML, and `zip`, `dom` and `simplexml` are all present. For files of known shape, reading `xl/sharedStrings.xml` and `xl/worksheets/sheet1.xml` directly is perfectly viable:

```php
$zip = new ZipArchive;
$zip->open($path);

$strings = [];
if ($xml = $zip->getFromName('xl/sharedStrings.xml')) {
    foreach (simplexml_load_string($xml)->si as $item) {
        $strings[] = (string) $item->t;
    }
}

$sheet = simplexml_load_string($zip->getFromName('xl/worksheets/sheet1.xml'));
foreach ($sheet->sheetData->row as $row) {
    foreach ($row->c as $cell) {
        $value = (string) $cell->v;
        $cells[] = ((string) $cell['t'] === 's') ? $strings[(int) $value] : $value;
    }
}
```

Good for a fixed export you control. Don't take this route for arbitrary user uploads — dates, formulas, inline strings, multiple sheets and styling all need handling, and reimplementing a spreadsheet reader is not the problem you set out to solve.

### 3. Change the format

If you control the input, CSV needs no extension at all — `fgetcsv` or `league/csv`. For output, `dompdf` produces PDFs without `xmlwriter`, and `System::printToPDF()` renders HTML natively. Asking an internal user for CSV is often cheaper than any of the above.

### 4. Move the work off the device

If the app is online anyway, an API you control can do the conversion with a normal PHP install. This trades an offline capability for a much simpler client, which is a poor deal for a desktop app whose selling point is working offline — weigh it honestly.

A custom PHP binary with `xmlreader` compiled in also works, but you take on the build pipeline as a supply-chain risk and forfeit support via GitHub Issues. See `build-and-distribute.md`.

## The `Process` facade is not a process runner

Despite the name, `Native\Desktop\Facades\Process` has nothing to do with `ChildProcess`. It reports system information, and is undocumented on the website:

```php
Process::arch();      // "x64", "arm64"
Process::platform();  // "darwin", "win32", "linux"
Process::uptime();
Process::fresh();
```

Use it for platform branching — selecting the right bundled binary for the current OS, say. To *run* something, you want `ChildProcess`.

## Broadcasting

NativePHP broadcasts both its own native events and any of your events you opt in, over IPC — no websocket server needed.

### Listening in JavaScript

A `window.Native` object is injected into every window. Register listeners inside a `native:init` handler; before that event the object may not exist yet.

```js
window.addEventListener('native:init', () => {
    Native.on('Native\\Desktop\\Events\\Windows\\WindowBlurred', (payload, event) => {
        //
    })
})
```

### Listening in Livewire

Prefix the event name with `native:`:

```php
use Native\Desktop\Events\Windows\WindowBlurred;
use Native\Desktop\Events\Windows\WindowFocused;

class AppSettings extends Component
{
    public $windowFocused = true;

    #[On('native:'.WindowFocused::class)]
    public function windowFocused() { $this->windowFocused = true; }

    #[On('native:'.WindowBlurred::class)]
    public function windowBlurred() { $this->windowFocused = false; }
}
```

### Broadcasting your own events

Implement `ShouldBroadcastNow` and include the `nativephp` channel:

```php
use Illuminate\Broadcasting\Channel;
use Illuminate\Contracts\Broadcasting\ShouldBroadcastNow;

class JobFinished implements ShouldBroadcastNow
{
    public function broadcastOn(): array
    {
        return [new Channel('nativephp')];
    }
}
```

This is the clean way to have a queued job or child process update the UI without polling.

The full list of native events lives in the package's `src/Events` directory, organised by subject.
