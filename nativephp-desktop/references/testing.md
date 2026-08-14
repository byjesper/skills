# Testing

## Why fakes are necessary here

NativePHP facades work by making authenticated HTTP calls to the Electron process. In a test suite there is no Electron process, so any code path touching a native facade fails with an HTTP error. That failure is expected, not a sign of broken code — it's the signal to reach for a fake.

`::fake()` swaps the real implementation for one with an identical API that records calls instead of making them. Your application code needs no changes.

```php
use Native\Desktop\Facades\Window;

#[\PHPUnit\Framework\Attributes\Test]
public function example(): void
{
    Window::fake();

    $this->get('/whatever-action');

    Window::assertOpened('window-name');
}
```

Pair with `Http::fake()` when the code under test also makes outbound HTTP calls.

Fakes exist for `Window`, `GlobalShortcut`, `Shell`, `PowerMonitor`, `ChildProcess` and `QueueWorker`. Facades without a fake — `Notification`, `Menu`, `Settings`, `System` and so on — need mocking or an abstraction of your own if a test path reaches them.

## Windows

```php
use Illuminate\Support\Facades\Http;
use Native\Desktop\Facades\Window;

Http::fake();
Window::fake();

$this->get('/whatever-action');

Window::assertOpened(fn (string $windowId) => Str::startsWith($windowId, ['window-name']));
Window::assertClosed('window-name');
Window::assertHidden('window-name');
```

Assertions: `assertOpened`, `assertClosed`, `assertHidden`, `assertShown`, `assertReloaded`, `assertNotReloaded`. Each accepts a string, a closure receiving the window ID, or nothing at all.

Count variants: `assertOpenedCount`, `assertClosedCount`, `assertHiddenCount`, `assertShownCount`, each taking an integer.

The docs list only the first three. The fake implements `show()` and `hide()`, so code calling `Window::show('main')` is testable even though the method is missing from the facade docblock.

### Asserting against the window instance

When what matters is the configuration a window was opened with, not just that it opened:

```php
use Mockery;
use Native\Desktop\Facades\Window;
use Native\Desktop\Windows\Window as WindowImplementation;

Http::fake();
Window::fake();
Window::alwaysReturnWindows([
    $mockWindow = Mockery::mock(WindowImplementation::class)->makePartial(),
]);

$mockWindow->shouldReceive('route')->once()->with('action')->andReturnSelf();
$mockWindow->shouldReceive('height')->once()->with(500)->andReturnSelf();
$mockWindow->shouldReceive('width')->once()->with(775)->andReturnSelf();

$this->get(route('action'));
```

`andReturnSelf()` on every expectation is what keeps the fluent chain intact — omit it and the chain breaks on the next call.

This is precise but brittle: it couples the test to the exact chain of builder calls, so a harmless reordering fails it. Reserve it for windows whose dimensions or route genuinely matter, and use the simpler assertions elsewhere.

## Global shortcuts

```php
use Native\Desktop\Facades\GlobalShortcut;

GlobalShortcut::fake();

$this->get('/whatever-action');

GlobalShortcut::assertKey('CmdOrCtrl+,');
GlobalShortcut::assertRegisteredCount(1);
GlobalShortcut::assertEvent(OpenPreferencesEvent::class);
```

Assertions: `assertKey`, `assertRegisteredCount`, `assertUnregisteredCount`, `assertEvent`.

## Shell

```php
use Native\Desktop\Facades\Shell;

Shell::fake();

$this->get('/whatever-action');

Shell::assertOpenedExternal('https://some-url.test');
```

Assertions: `assertShowInFolder`, `assertOpenedFile`, `assertTrashedFile`, `assertOpenedExternal`.

Worth testing deliberately: `assertTrashedFile` confirms you used the recoverable path rather than deleting outright.

## Power monitor

```php
use Native\Desktop\Facades\PowerMonitor;

PowerMonitor::fake();

$this->get('/whatever-action');

PowerMonitor::assertGetSystemIdleState('...');
```

Assertions: `assertGetSystemIdleState`, `assertGetSystemIdleStateCount`, `assertGetSystemIdleTimeCount`, `assertGetCurrentThermalStateCount`, `assertIsOnBatteryPowerCount`.

The count assertions are the useful ones for catching polling loops that hammer the API.

## Child processes

```php
use Native\Desktop\Facades\ChildProcess;

ChildProcess::fake();

$this->get('/whatever-action');

ChildProcess::assertGet('background-worker');
ChildProcess::assertMessage(
    fn (string $message, ?string $alias) => $message === '{"some-payload":"for-the-worker"}' && $alias === null
);
```

Assertions: `assertGet`, `assertStarted`, `assertPhp`, `assertArtisan`, `assertNode`, `assertStop`, `assertRestart`, `assertMessage`.

## Queue workers

```php
use Native\Desktop\DataObjects\QueueConfig;
use Native\Desktop\Facades\QueueWorker;

QueueWorker::fake();

$this->get('/whatever-action');

QueueWorker::assertUp(fn (QueueConfig $config) => $config->alias === 'custom');
```

Assertions: `assertUp`, `assertDown`.

Note the namespace: `Native\Desktop\DataObjects\QueueConfig`. The docs cite `DTOs\` in places; that path does not exist.

## What to test where

Most of a NativePHP app is ordinary Laravel and should be tested as such — no fakes, no ceremony. Reach for fakes at the boundary: the controller that opens a window, the listener that starts a child process, the action that trashes a file.

Automated tests can't cover the parts most likely to break in the field: build and packaging, signing and notarization, migrations against a real user database, and behaviour on other operating systems. Those need a production build run by hand on each target platform before release.
