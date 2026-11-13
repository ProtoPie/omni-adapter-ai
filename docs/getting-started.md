# Connect your first program to a stage

This guide takes you from nothing to a program that shows up on a Connect Omni stage,
sends a message and receives messages, in any of the six languages. The language READMEs
are the reference for everything after that.

## Words used in this guide

- **Stage** — a shared Connect workspace. Prototypes, people and programs on a stage
  exchange messages on named **channels** (`temp.reading`, `door.open`).
- **Channel** — what ProtoPie calls a *message*: the Console's *Message* column and its
  **Send Message** box use the same name.
- **Session** — a stage that is running right now. Programs can only join a live session.
- **Participant** — anything connected to a live session: a person, a prototype, your
  program.
- **Backstage** — the stage's wiring canvas. Each connected program shows there as a node,
  next to the stage's prototypes; people do not appear there.
- **Console** — the stage's message log.
- **Adapter** — the library you add to your program so it can join a stage. One per
  language: TypeScript, Python, C#, Kotlin, Swift, Dart.

## The three things you set up

| What | Where it comes from | What it is |
| --- | --- | --- |
| **Adapter token** — starts with `cst_` | The stage: **Share → Adapter keys → Create key → Copy** | The credential. It already contains the server address and the stage, so it is the only connection setting you need. Keep it secret. |
| **The adapter** | Added to your project (step 2) | The library that speaks the stage protocol for your language. |
| **Adapter name** — `displayName` in your code | You choose it (step 3) | How the stage shows your program, and how screen layers find it. |

> **The key name is not the adapter name.** When you create a key, Connect asks you to
> *"Name this key (e.g. booth-3)"*. That name only labels the key in the key list, so you
> know which one to revoke later. The name the stage shows comes from `displayName` in your
> code. Using the same word for both keeps things easy to match up.

You may also see a bare key that starts with `ck_`. It is the raw key inside the token; you
don't need it.

## Before you start

- **You can edit the stage** — you are an Editor or Owner in the stage's team. Only they
  see **Share** and can create adapter tokens.
- **Your Connect supports adapters.** Open the stage, then **Share**. If there is no
  **Adapter keys** row, this Connect deployment does not offer adapters, and there is no
  other way to get a token. Ask whoever runs your Connect.
- **The stage has a live session when your program connects.** A session starts when an
  Editor or Owner opens the stage in Connect; people who join through a share link cannot
  start one. The person who created the stage holds the session: when they close the stage,
  it ends about a minute later unless they still have it open in another window — other
  Editors' windows and connected programs do not keep it alive. Your program then waits and
  rejoins the next session.
- **For unattended setups** (a kiosk, a booth, a test rig), create the key with **Let adapters
  start a session** turned on in **Adapter keys** (a key keeps the setting it was issued with) —
  then your program can start the session itself. While such a key exists, the creator is just
  another participant: the session stays open while anyone — person
  or program, the creator included — is connected, and closes by itself a few minutes (five by
  default) after the last one leaves.

## Step 1 — Create an adapter token

1. Open the stage in Connect and click **Share → Adapter keys**.
2. Type a key name, for example `line3-sensor-pc`, and click **Create key**.
3. Click **Copy**. The token is shown **only once** — the server keeps just a fingerprint.
   If you lose it, create a new key and revoke the old one.
4. Put it in an environment variable (or your secret store), not in your source code:

   ```bash
   export CONNECT_TOKEN='cst_…'        # macOS / Linux
   ```

   ```powershell
   $env:CONNECT_TOKEN = 'cst_…'        # Windows PowerShell
   ```

   Set it in the terminal you run the program from. Programs started from an IDE (Visual
   Studio, Rider, IntelliJ, Android Studio, Xcode) do not see your terminal's variables —
   add `CONNECT_TOKEN` to the run configuration instead.

   Paste the token exactly as copied. A token with a space or newline around it is rejected
   with `ConfigError: credential has leading or trailing whitespace` — this usually happens
   when the token is read from a file that ends with a newline.

**Use one token per program and machine** — except a standby copy (step 3), which must share
its active program's token. Revoking a token disconnects every program that uses it, so one
token per program lets you cut off exactly one.

## Step 2 — Add the adapter to a project

Each adapter is on its language's public registry. Set up a project in your language; each
block ends with the command that runs the example from step 4.

**TypeScript / JavaScript** (Node 22+)

```bash
mkdir my-adapter && cd my-adapter
npm init -y && npm pkg set type=module                # the example uses top-level await
npm install @studioxid/omni-adapter tsx
npx tsx main.ts
```

**Python** (3.10+)

```bash
python3 -m venv .venv && . .venv/bin/activate         # Windows: .venv\Scripts\activate
pip install omni-adapter
python main.py
```

**C# / .NET** (.NET 8 SDK — the example is a console app)

```bash
dotnet new console -o MyAdapter && cd MyAdapter
dotnet add package StudioXid.OmniAdapter --prerelease   # only beta versions are published so far
dotnet run                                            # with the example in Program.cs
```

For **Unity** the setup is different (plugin DLLs and a main-thread dispatcher) — follow
the **Unity** section of the C# package's README (shown on its nuget.org page).

**Kotlin / JVM** (JDK 17–21 to build, Gradle; the library itself targets JVM 11). A project
folder with these two files — start from `gradle init` → application → Kotlin, which also
gives you the `./gradlew` wrapper, and replace its two build files with:

```kotlin
// settings.gradle.kts
rootProject.name = "my-adapter"
include("app")

// app/build.gradle.kts
plugins {
    kotlin("jvm") version "2.1.0"
    application
}

repositories {
    mavenCentral()                                    // the adapter and its own dependencies come from here
}

dependencies {
    implementation("com.studioxid:omni-adapter:0.1.0-beta.2")
}

application {
    mainClass = "MainKt"                              // the class Kotlin makes from Main.kt
}
```

Put the example in `app/src/main/kotlin/Main.kt`, delete the sample code `gradle init` created
(`app/src/main/kotlin/<package>/` and `app/src/test/` — the build file above declares no test
library), and run `./gradlew run`. If the build fails with a configuration-cache error, remove
`org.gradle.configuration-cache=true` from `gradle.properties`.

**Swift** (Swift 6.0+ / Xcode 16+, macOS 10.15+ / iOS 13+). `Package.swift` — the
`swift-tools-version` comment must stay its first line:

```swift
// swift-tools-version:5.9
import PackageDescription

let package = Package(
    name: "MyAdapter",
    platforms: [.macOS(.v10_15), .iOS(.v13)],         // required — the adapter needs these
    dependencies: [.package(url: "https://github.com/ProtoPie/omni-adapter-swift", from: "0.1.0-beta.2")],
    targets: [
        .executableTarget(name: "MyAdapter", dependencies: [
            .product(name: "OmniAdapter", package: "omni-adapter-swift"),  // package = repository name
        ]),
    ]
)
```

Put the example in `Sources/MyAdapter/main.swift` and run `swift run`.

**Dart** (Dart 3.4+)

```yaml
# pubspec.yaml
name: my_adapter
environment:
  sdk: ^3.4.0
dependencies:
  omni_adapter: ^0.1.0-beta.2   # only beta versions are published so far
```

Put the example in `bin/main.dart` and run `dart pub get && dart run bin/main.dart`.

## Step 3 — Choose the adapter's name

`displayName` is how the stage shows your program — the node on the Backstage canvas, the
*Source* column in the Console — and how an Adapter Stream layer finds the program whose
screen it shows.

- **1–64 characters.** If you leave it out, the adapter is called `Omni adapter`, so every
  unnamed program looks the same. Always set it.
- **Keep it stable.** Use the same name on every run. The participant id changes every time
  the program connects; the name is what people and layers recognise.
- **Make it unique per program.** Your program keeps its name across restarts as long as it
  uses the same token. While another program with a **different** token is connected under
  the same name, yours joins as `line3-temperature (2)` instead — and a screen program
  (`providesView: 'stream'`) also gets `(2)` when a screen layer with that name belongs to
  another token that is still valid; if you replaced a token, revoke the old one. A person who
  joins under your program's name is the one renamed, not your program. Print
  `client.me.displayName` after connecting to see the name you actually got.
- **Stop it cleanly.** A program that exits without leaving the stage (it crashed, or it
  was killed) keeps its place in the member list for a few seconds (eight by default) in case
  it reconnects. The examples in step 4 leave the stage when you press Ctrl+C or a service
  manager stops them, so the list is right at once. A restart within those seconds keeps its
  name (older servers, from before same-key name sharing, renamed it to `(2)`).
- **A pattern that works:** `<place>-<role>` — `line3-temperature`, `booth2-kiosk`,
  `lab-robot-arm`.
- **Running two copies of the same program on purpose** (a standby for failover)? Give both
  the same name and the same token, and turn on `exclusiveName` (`exclusiveName: true`;
  Python `exclusive_name=True`, C# `ExclusiveName = true`). The newest one is active; the
  other keeps receiving, but its sends are held back: a plain `send` is
  dropped, and `send(..., ack: true)` fails with `Standby` (whatever it queued while
  disconnected is dropped the same way when it turns out to be standby). Check
  `client.standby` before sending. A program with a
  different token still joins as `(2)`.

## Step 4 — Connect

Each example reads the token from `CONNECT_TOKEN`, joins under a name, prints what it
receives, sends `hello.world` and waits for the server to acknowledge it. Then it runs
until you press Ctrl+C or a service manager stops it (SIGTERM) — it leaves the stage and
exits — or until the session is over for good (for example, the token was revoked): it
prints `session over: <reason>` and exits with status 1. If the session merely closes, the
example waits and rejoins the next one. When something fails, it prints the error code and
its **hint** — a one-line next step — or the message when the error has no hint (just the code
when there is nothing more: `AckTimeout` means the server did not confirm `hello.world` in
time), leaves the stage if it had joined, and exits with status 1. Replace `line3-temperature` with your
name.

**TypeScript** (`main.ts`)

```ts
import { connect, isAdapterError } from '@studioxid/omni-adapter';

try {
  const client = await connect({
    secretKey: process.env.CONNECT_TOKEN ?? '',
    displayName: 'line3-temperature',
  });
  // Run until the session is over (its reason) or you stop the program (null).
  const stopped = new Promise<string | null>((resolve) => {
    client.on('ended', (reason) => resolve(reason ?? 'ended'));
    process.once('SIGINT', () => resolve(null)); // Ctrl+C
    process.once('SIGTERM', () => resolve(null)); // stopped by a service manager
  });
  try {
    console.log(`joined as ${client.me?.displayName}`);
    client.on('*', (msg) => console.log(`${msg.channel} = ${msg.value}`));
    await client.send('hello.world', 'hi', { ack: true });
    const reason = await stopped;
    if (reason !== null) {
      console.error(`session over: ${reason}`);
      process.exitCode = 1;
    }
  } finally {
    await client.close(); // leave the stage, so the next run keeps the name
  }
} catch (err) {
  if (!isAdapterError(err)) throw err;
  const why = err.hint ?? err.message; // some codes (AckTimeout) carry nothing more than the code
  console.error(why === err.code ? err.code : `${err.code}: ${why}`);
  process.exitCode = 1;
}
```

**Python** (`main.py`)

```python
import asyncio
import os
import sys
from typing import Optional

import omni_adapter


async def main() -> int:
    try:
        client = await omni_adapter.connect(
            secret_key=os.environ.get("CONNECT_TOKEN", ""),
            display_name="line3-temperature",
        )
    except omni_adapter.AdapterError as err:
        print(f"{err.code}: {err.hint or err.message}", file=sys.stderr)
        return 1
    # Run until the session is over (its reason) or you stop the program (None).
    stopped = asyncio.get_running_loop().create_future()

    def stop(reason: Optional[str] = None) -> None:
        if not stopped.done():
            stopped.set_result(reason)

    client.on("ended", lambda reason: stop(reason or "ended"))
    # Ctrl+C, or a service manager's stop: the first signal ends the wait, a second exits at once.
    with omni_adapter.stop_on_signals(stop):
        try:
            print(f"joined as {client.me.display_name if client.me else '?'}")

            @client.on("*")
            def on_any(msg: omni_adapter.StageMessage) -> None:
                print(f"{msg.channel} = {msg.value!r}")

            await client.send("hello.world", "hi", ack=True)
            reason = await stopped
            if reason is not None:
                print(f"session over: {reason}", file=sys.stderr)
                return 1
            return 0
        except omni_adapter.AdapterError as err:
            why = err.hint or err.message  # some codes (AckTimeout) carry nothing more than the code
            print(err.code if why == err.code else f"{err.code}: {why}", file=sys.stderr)
            return 1
        finally:
            await client.close()  # leave the stage, so the next run keeps the name


try:
    sys.exit(asyncio.run(main()))
except KeyboardInterrupt:
    pass  # Ctrl+C before the handlers are installed, or a second Ctrl+C
```

**C#** (`Program.cs`)

```csharp
using System.Runtime.InteropServices;
using StudioXid.OmniAdapter;

ConnectClient client;
try
{
    client = await Connect.ToStageAsync(new ConnectOptions
    {
        SecretKey = Environment.GetEnvironmentVariable("CONNECT_TOKEN") ?? "",
        DisplayName = "line3-temperature",
    });
}
catch (AdapterError err)
{
    Console.Error.WriteLine($"{err.Code}: {err.Hint ?? err.Message}");
    return 1;
}

// Run until the session is over (its reason) or you stop the program (null).
var stop = new TaskCompletionSource<string?>(TaskCreationOptions.RunContinuationsAsynchronously);
client.Ended += reason => stop.TrySetResult(reason ?? "ended");
Console.CancelKeyPress += (_, e) => { e.Cancel = true; stop.TrySetResult(null); };   // Ctrl+C
using var sigterm = PosixSignalRegistration.Create(PosixSignal.SIGTERM,                // a service manager's stop
    ctx => { ctx.Cancel = true; stop.TrySetResult(null); });
try
{
    Console.WriteLine($"joined as {client.Me?.DisplayName}");
    client.On("*", msg => Console.WriteLine($"{msg.Channel} = {msg.Value}"));
    await client.SendAsync("hello.world", "hi", ack: true);
    var ended = await stop.Task;
    if (ended == null) return 0;
    Console.Error.WriteLine($"session over: {ended}");
    return 1;
}
catch (AdapterError err)
{
    var why = err.Hint ?? err.Message;   // some codes (AckTimeout) carry nothing more than the code
    Console.Error.WriteLine(why == err.Code ? err.Code : err.Code + ": " + why);
    return 1;
}
finally
{
    await client.CloseAsync(); // leave the stage, so the next run keeps the name
}
```

**Kotlin** (`Main.kt`)

```kotlin
import com.studioxid.omniadapter.AdapterError
import com.studioxid.omniadapter.ClientEvent
import com.studioxid.omniadapter.ConnectOptions
import com.studioxid.omniadapter.connect
import kotlinx.coroutines.CoroutineStart
import kotlinx.coroutines.async
import kotlinx.coroutines.flow.filterIsInstance
import kotlinx.coroutines.flow.first
import kotlinx.coroutines.runBlocking
import kotlin.system.exitProcess

fun main(): Unit = runBlocking {
    val client = try {
        connect(
            ConnectOptions(
                secretKey = System.getenv("CONNECT_TOKEN") ?: "",
                displayName = "line3-temperature",
            ),
        )
    } catch (err: AdapterError) {
        System.err.println("${err.code}: ${err.hint ?: err.message}")
        exitProcess(1)
    }
    // Ctrl+C and a service manager's stop (SIGTERM) run shutdown hooks:
    // leave the stage, so the next run keeps the name.
    Runtime.getRuntime().addShutdownHook(Thread { runBlocking { client.close() } })
    // Watch for the end of the session from now on (events are not replayed).
    val ended = async(start = CoroutineStart.UNDISPATCHED) {
        client.events.filterIsInstance<ClientEvent.Ended>().first()
    }
    try {
        println("joined as ${client.me?.displayName}")
        client.on("*") { msg -> println("${msg.channel} = ${msg.value}") }
        client.send("hello.world", "hi", ack = true)
    } catch (err: AdapterError) {
        val why = err.hint ?: err.message // some codes (AckTimeout) carry nothing more than the code
        System.err.println(if (why == err.code) err.code else "${err.code}: $why")
        exitProcess(1) // runs the shutdown hook, which leaves the stage
    }
    val reason = ended.await().reason // runs until the session ends or the program is stopped
    if (reason != "client-closed") {
        System.err.println("session over: $reason")
        exitProcess(1)
    }
}
```

**Swift** (`main.swift`)

```swift
import OmniAdapter
import Foundation

let client: ConnectClient
do {
    client = try await connect(ConnectOptions(
        secretKey: ProcessInfo.processInfo.environment["CONNECT_TOKEN"] ?? "",
        displayName: "line3-temperature"
    ))
} catch let error as AdapterError {
    print("\(error.code): \(error.hint ?? error.message)")
    exit(1)
}

// Ctrl+C and a service manager's stop (SIGTERM): leave the stage, so the next run keeps the name.
var signalSources: [DispatchSourceSignal] = []
for sig in [SIGINT, SIGTERM] {
    signal(sig, SIG_IGN)
    let source = DispatchSource.makeSignalSource(signal: sig, queue: .main)
    source.setEventHandler { Task { await client.close() } }
    source.resume()
    signalSources.append(source)
}
let events = await client.events() // watch for the end of the session from now on
do {
    print("joined as \(await client.me?.displayName ?? "?")")
    await client.on("*") { message in print("\(message.channel) = \(message.value.jsonValue)") }
    try await client.send("hello.world", "hi", ack: true)
} catch let error as AdapterError {
    let why = error.hint ?? error.message // some codes (AckTimeout) carry nothing more than the code
    print(why == error.code ? error.code : "\(error.code): \(why)")
    await client.close()
    exit(1)
}
// Run until the session ends or you stop the program.
for await case .ended(let reason) in events {
    await client.close() // lets the leave started by a signal finish (a no-op otherwise)
    if reason == "client-closed" { exit(0) }
    print("session over: \(reason ?? "ended")")
    exit(1)
}
```

**Dart** (`bin/main.dart`)

```dart
import 'dart:async';
import 'dart:io';

import 'package:omni_adapter/omni_adapter.dart';

Future<void> main() async {
  final ConnectClient client;
  try {
    client = await connect(ConnectOptions(
      secretKey: Platform.environment['CONNECT_TOKEN'] ?? '',
      displayName: 'line3-temperature',
    ));
  } on AdapterError catch (err) {
    stderr.writeln('${err.code}: ${err.hint ?? err.message}');
    exit(1);
  }
  // Run until the session is over (its reason) or you stop the program (null).
  final stopped = Completer<Object?>();
  client.onEvent('ended', (reason) {
    if (!stopped.isCompleted) stopped.complete(reason ?? 'ended');
  });
  final signals = [
    ProcessSignal.sigint, // Ctrl+C
    if (!Platform.isWindows) ProcessSignal.sigterm, // a service manager's stop
  ].map((s) => s.watch().listen((_) {
        if (!stopped.isCompleted) stopped.complete(null);
      })).toList();
  var code = 0;
  try {
    print('joined as ${client.me?.displayName}');
    client.on('*', (msg) => print('${msg.channel} = ${msg.value}'));
    await client.send('hello.world', 'hi', ack: true);
    final reason = await stopped.future;
    if (reason != null) {
      stderr.writeln('session over: $reason');
      code = 1;
    }
  } on AdapterError catch (err) {
    final why = err.hint ?? err.message; // some codes (AckTimeout) carry nothing more than the code
    stderr.writeln(why == err.code ? err.code : '${err.code}: $why');
    code = 1;
  } finally {
    await client.close(); // leave the stage, so the next run keeps the name
    for (final s in signals) {
      await s.cancel();
    }
  }
  exit(code);
}
```

### In an app rather than a script

The examples are command-line programs. In an app, keep the same `connect` / `on` / `send`
calls, and call `close()` (`CloseAsync()` in C#) when the app is done with the stage — a
screen closes, the app quits. It leaves the stage at once, so the next start keeps the name.
Adjust the rest:

- **Android** — add the `INTERNET` permission (and allow cleartext traffic if your server
  is plain `http://`). Connect from `lifecycleScope`, not `runBlocking`; callbacks do not
  run on the main thread. See the Kotlin package's README.
- **iOS / macOS apps** — a plain `http://` server addressed by host name needs an App Transport
  Security exception, and a sandboxed macOS app needs the `com.apple.security.network.client`
  entitlement.
  Callbacks run off the main actor; hop with `Task { @MainActor in … }` before touching
  UI (see the Swift package's README).
- **Flutter** — `dart:io` (`Platform.environment`, `exit`, `ProcessSignal`) is not
  available on the web; pass the token with `--dart-define=CONNECT_TOKEN=cst_…` and read it
  with `const String.fromEnvironment('CONNECT_TOKEN')`. Anything you build into an app ships
  inside it — in a Flutter **web** app every visitor can read the token, so use a token
  you are ready to revoke. See the Dart package's README on pub.dev.
- **WPF / WinForms / Unity** — callbacks run on the adapter's receive thread; post UI work
  to the UI thread and never block inside a callback. See the
  C# package's README.
- **A service** (systemd, Docker, Kubernetes, a Windows service) is stopped with a terminate
  signal, not Ctrl+C — call `close()` on that signal too, or each restart leaves the old
  run in the member list for a few seconds.

## Step 5 — Check that it worked

- Your program prints `joined as line3-temperature`.
- On the stage's **Backstage** canvas a node with that name appears.
- The stage **Console** shows `hello.world` with `line3-temperature` in the *Source* column.
- Send something to the stage from the Console or a prototype — your program prints it.

If it does not connect, run the **doctor** before changing code. It checks the token, the
server address, the server, the live session and a real join, and names the step that
failed with a fix:

```bash
npx --no-install omni-adapter doctor "$CONNECT_TOKEN"   # in the TypeScript project from step 2
python -m omni_adapter doctor "$CONNECT_TOKEN"          # in the Python virtual environment from step 2
```

Only lines marked ✖ need fixing; lines marked △ or · are information.

C#, Kotlin, Swift and Dart have no command — there, and in your own diagnostics screen, use
the library call and print its report when `connect` fails:

| Language | Run | Format |
| --- | --- | --- |
| TypeScript, Dart | `await runDoctor(token)` | `formatDoctorReport(results)` |
| Kotlin | `runDoctor(token)` (from a coroutine) | `formatDoctorReport(results)` |
| Python | `await run_doctor(token)` | `format_doctor_report(results)` |
| C# | `await Doctor.RunAsync(token)` | `Doctor.FormatReport(results)` |
| Swift | `await runDoctor(credential: token)` | `formatDoctorReport(results)` |

The doctor checks the **connection**, not delivery: if it passes and a peer still does not
react, look at the channel names and at the receiving side.

## When it does not connect

| What you see | Why | What to do |
| --- | --- | --- |
| No **Share** button on the stage | You can view the stage but not edit it | Ask an Editor or Owner of the stage's team to create the token |
| **Share** has no **Adapter keys** row | This Connect deployment does not offer adapters | Ask whoever runs your Connect |
| `ConfigError: … requires a credential` | `CONNECT_TOKEN` is not set in the process that ran your code | Set it in the same terminal, or in the IDE's run configuration (step 1) |
| `ConfigError: credential has leading or trailing whitespace` | The token was copied with a space or newline | Remove it — trim the value you read from a file or variable |
| `ConfigError: an adapter key (ck_…) is not self-describing` | You used the bare `ck_…` key | Use the full `cst_…` token. **Copy** appears only right after **Create key** — if you no longer have the token, create a new key |
| `AuthRevoked` | The token was revoked, or it was issued for another stage or server | Create a new one: **Share → Adapter keys → Create key** |
| `AuthRevoked` mentioning `stage-locked` | The stage is closed to guests (`allowAnonymousGuests` is false), and adapters join as guests | Ask the stage owner or whoever runs your Connect to reopen it to guests — no screen in the Connect web app has this switch |
| `RoomEnded` (404) | No live session — nobody with edit rights has the stage open | Open the stage in Connect, or create a key with **Let adapters start a session** turned on (**Share → Adapter keys** — a key keeps the setting it was issued with) and use that token |
| `StageArchived` | The stage was archived | Ask an Editor or Owner of the stage's team to restore it: **Archived** page → right-click the stage → **Restore** |
| `StageFull` | The stage's participant limit is reached — every connected program takes a seat | Stop programs you don't need, and reuse one client instead of connecting per message |
| `KeyMaterialUnavailable` from `connect()` (503) | The server cannot read the stored material of an adapter key on this stage. One such attempt alone fails with a plain 500 (`Internal`); after 30 of these from one address within a minute for one stage, every token — a correct one too — gets this 503 until the minute ends. The error's `retryAfterMs` and hint say how long | Ask whoever runs your Connect to revoke the stage's old keys (**Share → Adapter keys**) or restore the key material. A program that had already joined keeps retrying by itself, waiting at least `retryAfterMs` |
| `TooManyRequests` from `connect()` | The server refused the join for now: this network address made too many failed join attempts — 20 within 5 minutes, usually another program on the same machine or network that still uses a revoked or mistyped token, or keeps restarting with one. Or one token joined more than 60 times in a minute — programs restarting in a loop | Wait as long as the hint says (the server's `retryAfterMs`, up to 60 s for the per-key limit) or up to 5 minutes, then retry. Find the program that causes it — give it a current token, or stop the restart loop — or it happens again. (Older adapter-enabled servers also refused a sixth join from one address within a minute — update the server) |
| `NotConnected` | The server cannot be reached — the address changed, or a firewall or VPN is in the way. From `members()` or `ping()` it can also mean the connection is down for a moment | Check the network; copy the token again if the server moved. From `members()` or `ping()`, read the error's hint — it says whether to wait for the reconnect or create a new client |
| `JoinTimeout` | Reached the server but the join never completed — often a proxy that blocks WebSocket | Switch to HTTP long-polling: `transports: ['polling']` (TypeScript), `transports=["polling"]` (Python), `UsePolling = true` (C#), `usePolling = true` (Kotlin, Swift). In Dart it works on Flutter web only |
| `ProtocolVersionMismatch` or `ClientAheadOfServer` | The adapter and the server speak different protocol versions | Update the adapter, or ask whoever runs your Connect to update the server |
| The name shows up as `line3-temperature (2)` | Another program with a different token is connected under that name — or, for a screen program, a screen layer with that name belongs to another token that is still valid (a replaced token that was never revoked) | Stop the other program, revoke the token you replaced, or pick another name. Two copies on the same token are never renamed — for a deliberate standby copy, `exclusiveName` (step 3) decides which one sends |
| `Standby` from `send(..., ack: true)`, or `client.standby` is `true` | `exclusiveName: true`, and another copy of the program is the active one | Expected for a standby copy — or turn `exclusiveName` off |

## What next

- Declare what your program sends and receives (`sends`, `receives`) — the
  Backstage node then shows the channels as ports. See your language README.
- Structured values travel as JSON strings: `sendJSON` / `onJSON` (TypeScript, Swift),
  `send_json` / `on_json` (Python), `SendJsonAsync` / `OnJson` (C#), `sendJson` / `onJson`
  (Kotlin, Dart).
- Values from prototypes and from the Console's **Send Message** box arrive as **strings**
  (`"42"`, not `42`) — convert them before comparing numbers.
- Stay under **60 messages a second** per program — the server drops what is above that.
  Coalesce fast sources with `throttle()`.
- Subscribe to errors and the end of the session in any program that runs unattended. By
  default TypeScript, Python and Dart print their own log lines to the console (each README
  says how to redirect or silence them), while C#, Kotlin and Swift print nothing — there, an
  error nobody listens to leaves no trace except `client.diagnostics()`.
- Feature support per language and per server version: [COMPATIBILITY.md](COMPATIBILITY.md).
- Writing the integration with an AI assistant: point it at [llms.txt](../llms.txt), or copy
  the skill folder [skills/using-omni-adapter/](../skills/using-omni-adapter/)
  into your project's `.claude/skills/` for Claude Code.
