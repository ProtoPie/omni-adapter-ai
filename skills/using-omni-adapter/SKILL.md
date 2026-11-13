---
name: using-omni-adapter
description: Use when writing or debugging code that joins a ProtoPie Connect Omni stage through the official adapter (TypeScript/Python/C#/Kotlin/Swift/Dart) — cst_/ck_ adapter tokens, sending sensor or hardware data to prototypes, receiving stage channel messages, wiring several programs or PCs together, roster/presence, screen broadcast to an Adapter Stream layer (Unity, simulators, canvas) and remote input, MemoryStage/replay/simulate tests, and "connected but nothing happens" diagnosis. Not for ProtoPie Studio authoring or the legacy ProtoPie Connect desktop app.
---

# Using the Omni Adapter

A **Connect Omni stage** is a real-time hub: prototypes, hardware bridges, web pages and
services exchange values over dotted channels (`sensor.temperature`). The usual shape is
**several programs, each on its own machine, joined to one stage and triggering each other** —
a prototype is one spoke, not the centre.

**The full guide** is `llms.txt` — in this skill at `references/llms.txt` (a byte-identical
copy of the repo's root `llms.txt`), in a plugin install at `${CLAUDE_PLUGIN_ROOT}/llms.txt`,
in an installed npm or NuGet package at the package root (`node_modules/@studioxid/omni-adapter/`
· `~/.nuget/packages/studioxid.omniadapter/<version>/`), in the adapter repo at its root. This file is the checklist; read the guide for API details.

## Step 0 — the credential, before anything else

Ask for (or find in the env) a **`cst_…` adapter token**: in Connect, *open the stage → Share →
Adapter keys → Create key, then Copy* (Copy appears only right after Create key; the token is
shown once). The only other accepted form is a bare **`ck_…` key with explicit `url` + `stageId`**.
Nothing else — a share URL or share token is a `ConfigError` before any network call.

- Put it in an env var (`CONNECT_TOKEN`), never in source. Whitespace around it (a token file's
  trailing newline) → `ConfigError: credential has leading or trailing whitespace` — trim it.
- No **Share** button → the user is not an Editor/Owner. No **Adapter keys** row → that
  deployment does not offer adapters. No code change fixes either.
- The key *name* typed at Create key is only a label; the stage shows `displayName` from code.
- First-time walkthrough for a human: `docs/getting-started.md` — shipped in the npm and NuGet
  packages (from the package root), and at that path in the adapter repo.

## Step 1 — pick the package, then read that language's section of `references/llms.txt`

| Language | Package | Read in `references/llms.txt` |
| --- | --- | --- |
| TypeScript / Node / browser | `@studioxid/omni-adapter` (ESM; `/react` subpath) | the TypeScript (and React) section |
| Python ≥ 3.10 (asyncio) | `omni-adapter`, module `omni_adapter` | the Python section |
| C# / .NET, Unity | `StudioXid.OmniAdapter` (netstandard2.0 · net8.0) | the C# section (+ Unity checklist) |
| Kotlin / JVM / Android | `com.studioxid:omni-adapter` (coroutines) | the Kotlin section |
| Swift (iOS · macOS) | `OmniAdapter` (SwiftPM, actor) | the Swift section |
| Dart / Flutter (incl. web) | `omni_adapter` | the Dart / Flutter section |

Always read the credential section, the rules section, the concepts section (name map,
lifecycle events per language, sessions and names, delivery, latest/history) and the
errors/troubleshooting section too. Shell checks: the `omni-adapter` CLI ships with the TS and
Python packages only.

**Install — from the public registries.** The Swift package is served from the public mirror
repository `ProtoPie/omni-adapter-swift`.

| Language | Install |
| --- | --- |
| TS | `npm install @studioxid/omni-adapter` (ESM-only; `package.json` needs `"type": "module"`) |
| Python | `pip install omni-adapter` (module `omni_adapter`) |
| C# | `dotnet add package StudioXid.OmniAdapter --prerelease` (beta only so far) |
| Kotlin | module: `implementation("com.studioxid:omni-adapter:0.1.0-beta.2")` with `mavenCentral()` in `repositories` |
| Swift | `.package(url: "https://github.com/ProtoPie/omni-adapter-swift", from: "0.1.0-beta.2")` + `.product(name: "OmniAdapter", package: "omni-adapter-swift")` + `platforms: [.macOS(.v10_15), .iOS(.v13)]` |
| Dart | `dart pub add omni_adapter` (Flutter: `flutter pub add omni_adapter`) |

## Step 2 — connect with declarations

```ts
const client = await connect({
  secretKey: process.env.CONNECT_TOKEN!,   // cst_…
  displayName: 'line3-temperature',        // 1–64 chars; what the stage shows
  receives: ['line.cmd'],                  // exact names, max 32 — omit, never []
  sends: ['line.temp'],
});
```

**Declare every channel the program uses** — each channel it sends on in `sends`, each channel it
subscribes to in `receives` — even without `strictRouting`, and add a channel there in the same
change that adds its send or handler. Connect knows a program's channels only from this
declaration (Backstage node chips and flow pulses, `listeners()`, and — for a page in a Web View
layer — the layer's Stage messages Receive / Send rules: an editor opening the page turns the
declared channels on, while anything undeclared waits under "This page asked for" for **Add**). An undeclared channel still flows, so a
gap shows up later as wiring nobody can see. If a list would pass 32, tell the user instead of declaring part of it.

Same options everywhere: Python snake_case (`secret_key`, `display_name`, `strict_routing`), C#
PascalCase on `ConnectOptions`, Kotlin/Swift/Dart camelCase on `ConnectOptions`. Declarations
draw the program as a node with port chips on the stage's Backstage canvas. `osHostname`: pass
a real name or omit it (Dart on the VM fills it for you; the others leave it unset).

## Step 3 — the rules that matter

- **Values are primitives** (`string | number | boolean | null`). Objects go through the JSON
  helpers (`sendJSON`/`onJSON`, `send_json`, `SendJsonAsync`, `sendJson`) — a value they cannot
  encode fails with `ValidationError` at the call (TS: the returned Promise rejects; the other
  five throw from the call — Dart and C# synchronously, before any `Future`/`Task` exists). Kotlin `sendJson` takes `Map`/`List`/`JSONObject`/`JSONArray`/
  arrays/primitives, not data classes; Swift `sendJSON` takes JSON-shaped values, not
  `Encodable` structs. **Prototypes and the Console send every value as a string** (`"42"`,
  not `42`; `null` when there is none) — convert before comparing.
- **No self-echo** — your own sends never come back to `on()`. `latest()` folds them in, so
  **`latest()` is not a delivery check**: confirm a publish with `ack: true` (a *server* ack —
  `AckTimeout` after 15 s by default, which does not prove the server missed it — nor do
  `QueueOverflow`/`Standby`/`SessionEnded` on a queued acked send that already went out once)
  or by watching the channel from a second connection. Fire-and-forget is **at-most-once**, queued or not: a
  server rejection, or a drop right after the send went out, is never seen or retried; only
  acked sends see the server's answer (and only they trigger the `TooManyRequests` backoff).
- **60 messages a second per connection.** The server drops fire-and-forget messages over that,
  so the adapter paces every send with a token bucket (60/s, bursts of 30); with no token the
  send queues. A source above 60/s fills the queue and loses its oldest values
  (`queue-overflow`, once per episode) — `throttle()` it. The queue drains at 60/s (a full
  default queue of 1000 takes about 17 s), so after a disconnect a ≥ 60/s sender stays up to
  `queueLimit` ÷ 60 s behind. Acked sends go one per round trip — never `ack` a stream.
- **History is state, not triggers**: on (re)join the server snapshot (≤1000 messages) re-seeds
  `latest()`/`history()` and fires the history event, but never re-fires channel handlers.
  Trigger channels are at-most-once across a disconnect; state channels read `latest()` on
  `connected`. `latest()` keeps the 1000 most recently updated channels.
- **Callbacks run in arrival order** (all six; Swift skips the oldest *message* callbacks once
  1000 are waiting — `latest()` stays exact). Never block inside one.
- `on('robot.*')` matches one or more segments; `on('*')` is everything. Wildcards are allowed
  in `receives` only without `strictRouting`: with `strictRouting: true` the server delivers
  only the exact declared `receives` — turn it on once more than two programs share a stage.
- `throttle(channel, hz)` coalesces a high-rate source; its push function checks the channel
  and value and throws `ValidationError` at the call site (a push after `close()` does not
  throw); delivery stays fire-and-forget.
- Errors are `AdapterError` with `.code`; print the code with **`hint ?? message`** —
  `AckTimeout`, `ValidationError`, `SessionEnded` and some `ConfigError`s carry no hint, and an
  error without a message of its own has the code as its message (`AckTimeout`), so also say
  which call failed.
- **Close on exit** (SIGINT *and* SIGTERM, `dispose()`, app quit). `close()` ends the client at
  the call and waits up to 1 s for the server to confirm the leave (await it before exiting); an
  unclean exit leaves the old seat in the roster for a few seconds (server grace, default 10 s — 8 s on servers before 2026-10-19).
  `close()` **drops queued fire-and-forget sends** — await an acked send
  or wait for `queuedCount == 0` first when the last values matter.

### Lifecycle events — the names differ per language

In C#, Kotlin, Swift and Dart, `on("error", …)` subscribes to a **channel** named `error` and
never fires for errors. Use exactly these:

| Language | How | Names |
| --- | --- | --- |
| TS | `client.on(name, cb)` (reserved names win over a channel of that name) | `stateChange` · `error` · `memberJoin` · `memberLeave` · `history` (`readonly StageMessage[]`) · `ended` (`string \| undefined`) · `queueOverflow` |
| Python | `client.on(name, cb)` — the same **camelCase** strings, not snake_case | same seven names |
| C# | .NET events | `StateChanged` · `Error` · `MemberJoined` · `MemberLeft` · `HistoryReplaced` · `Ended` (`string?`) · `QueueOverflow` |
| Kotlin | `client.state: StateFlow<String>` + `client.events: SharedFlow<ClientEvent>` | `StateChange` · `Failure` · `MemberJoin` · `MemberLeave` · `HistoryReplaced` · `Ended` · `QueueOverflow` |
| Swift | `for await e in await client.events()` | `.stateChange` · `.failure` · `.memberJoin` · `.memberLeave` · `.historyReplaced` · `.ended` · `.queueOverflow` |
| Dart | `client.onEvent(name, (Object? payload) {…})` | `stateChange` · `error` · `memberJoin` · `memberLeave` · `history` · `ended` · `queueOverflow` |

Subscribe to the error event in every headless program — an unobserved server error otherwise
leaves nothing but `client.diagnostics()`. `ended` is final: build a new client to reconnect.

### Per-language pitfalls

| Language | What bites |
| --- | --- |
| TS | An open client **holds the Node event loop** — a one-shot script must `await client.close()` or it never exits. `close()` returns a `Promise<void>` that never rejects and settles once the server confirms the leave (≤1 s); then Node exits on its own — never `process.exit()` before it settles. Attach handlers to acked sends before `await close()` (queued ones reject with `SessionEnded` at the call). React cleanup: `return () => { void client.close(); }`. `send()`/`sendJSON()` report failures by rejecting — an un-awaited rejection is unhandled and ends the process, so `.catch()` it. ESM-only: run with `npx tsx`. |
| Python | A **sync callback cannot `await`** — `client.send(...)` there sends nothing (coroutine never awaited); use `asyncio.create_task(client.send(...))` or an `async def` callback. Async callbacks start in order but interleave after their first `await`. Wrap the run in `with omni_adapter.stop_on_signals(stop.set):` and `await client.close()` in a `finally` inside it — with a bare `try/finally` under `asyncio.run`, Ctrl+C on 3.10 cannot send the leave; the helper works on Windows too and a second Ctrl+C exits at once. |
| C# | Server events arrive on the socket receive thread one at a time — never block (`.Result`, `.Wait()`, `Dispatcher.Invoke` → false `AckTimeout`). Lifecycle events can come from other threads (`Dispose()` raises `Ended` on the caller's). Marshal UI work with `BeginInvoke`/`Post`/a Unity main-thread dispatcher. An `async` lambda on an event is `async void` — wrap it in try/catch or an exception kills the process. |
| Kotlin | Callbacks run on socket.io's EventThread — never `runBlocking { client.send(..., ack = true) }` there (false `AckTimeout`); `launch` instead (an `Unconfined` collector runs there too). `events` has replay 0 and a 256-event drop-oldest buffer — nothing can subscribe before `connect` returns, so launch collectors with `CoroutineStart.UNDISPATCHED` right after it and read `state.value`/`history()` for what came before; one `launch` per collector (`collect` never returns). Android: `lifecycleScope`, no `runBlocking` on main, `Dispatchers.Main` for UI. A live client keeps the JVM alive until `close()` — `dispose()` alone does not release it. `sendJson` takes `Map`/`List`/`JSONObject`/`JSONArray`/primitives, not data classes. |
| Swift | `ConnectClient` is an actor — await every call. Callbacks are synchronous, off the main actor: hop with `Task { @MainActor in … }`. `message.value` prints as `string("go")` — use `.stringValue`/`.doubleValue`/`.jsonValue`; pass variables as `.double(x)`. The `throttle` push throws: `try push(.double(x))`. Sandboxed macOS apps need `com.apple.security.network.client`; in an app, plain `http://` to a host name needs an ATS exception (`URLSession`). |
| Dart / Flutter | Callbacks are synchronous on the caller's zone — heavy work janks the UI. `send`/`sendJson` throw **synchronously** for validation and `SessionEnded` (wrap the call in try; `catchError` never sees it). `--dart-define` values are compiled into the app (Flutter web: readable in `main.dart.js`) — read with `const String.fromEnvironment('CONNECT_TOKEN')`, never `Platform.environment` on web. `dispose()` is synchronous: `unawaited(_client?.close()); super.dispose();`. Sandboxed macOS apps need `com.apple.security.network.client`. `usePolling` works on Flutter web only. |

## Step 4 — several programs wired together

| Question | Answer with |
| --- | --- |
| Who is out there? | `members()` → `Member[]` (includes you; an async server round trip that rejects with `NotConnected` while not connected). Re-read it on every `connected` — joins and leaves missed while disconnected are not replayed. `displayName` is the stable contract; `participantId` changes every run; `clientType` `'adapter'`/`'player'`/absent (browser); `osHostname` says which PC. |
| Will anyone get what I publish? | `listeners(ch)` → `{ channel, declared, broadcast }` (two `Member` lists), self excluded. Prototypes never declare, so they appear in `broadcast`, never `declared`. Both empty = nobody receives it (typo, or the peer is not up). Call it once at wiring time with a concrete channel. |
| Did someone join or drop? | The memberJoin/memberLeave events — not polling (plus a fresh `members()` after each reconnect). |
| Is my link healthy? | `state` **and** `queuedCount` together, plus the error event. |
| Is that peer alive? | Presence ≠ liveness — have the peer publish a periodic tick, or watch its channel. |

- **Names**: the same adapter key is never renamed — a restart keeps its name. Another
  *program* on the same name with a **different** key makes yours join as `name (2)`, and for a
  `providesView: 'stream'` adapter so does an Adapter Stream layer with that name bound to another
  key that is still valid (a rotated key whose old copy was never revoked → revoke it). Check
  `client.me.displayName` after connecting. A person who takes a program's name is the one
  renamed. (Older servers, from before same-key name sharing, renamed a quick restart after an
  unclean exit to `name (2)`.)
- **One active per name**: `exclusiveName: true` keeps the newest instance active and the rest
  standby — only between instances on the **same adapter key** (a different key joins as
  `name (2)` and the flag is dropped). While standby, new sends are dropped (fire-and-forget)
  or rejected with `Standby` (`ack`) — never deferred. When an instance **becomes** standby (at
  a reconnect or on a presence change), its queue is dropped the same way (the `standby` ring
  entry says `— dropped N queued sends`), so a later promotion sends nothing stale. Sends made in
  the one roster round trip after each (re)connect, while standby is being judged, are held:
  active (or a failed judgment) → queued; standby → the standby rule. There is no standby event
  — read `client.standby` (or ring codes `standby`/`active`).
- **Sessions**: an Editor/Owner opening the stage in Connect starts one; share-link people
  cannot. The stage creator holds it — closing the stage ends it about a minute later even with
  programs connected (they wait and rejoin). A key created with **Let adapters start a session**
  turned on (Share → Adapter keys — a key keeps the setting it was issued with) can start it, and then the creator is just another participant — the session stays
  open while anyone (programs included) is connected and closes a few minutes (default 5) after the
  last one leaves. Revoking the last such key — or its expiry, noticed within 15 s — while no Editor or
  Owner is connected ends the session about a minute later; without such a key the session ends about a
  minute after its creator leaves (Editors inside do not hold it; a session an Editor opened without the
  creator ends a minute after the last Editor leaves), programs or not. `autoRejoin` (default on) waits through
  a closed stage.
- **Prototype → program messages**: a message a prototype, a Figma or Unity layer or the Console sends
  arrives with its name as the channel (servers from 2026-10-19); older servers deliver it on channel
  `STUDIO` with the name in `messageId`, where `on(name)` never fires.

## Step 5 — only if the program must show a screen

Putting a picture on the stage (Unity, a simulator, a canvas) or letting an editor operate it:
read `references/screens-and-remote-input.md` — `providesView`, `startScreenBroadcast`,
`ensureStreamLayer()`, ICE servers, remote input, the Unity checklist. Skip it for data-only.

## Step 6 — test before the hardware or the server exists

`MemoryStage` (all six) is an in-process stage with the wire rules: `stage.join()` opens a
client, `stage.inject()` plays the prototype, `disconnect()`/`reconnect()` reproduce a drop,
`end()` ends the session, `rejectPublish` injects failures; it skips the 60-messages-a-second
pacing (`bypassRateLimit`). `replay()` re-publishes a recording (`omni-adapter record` / `watch`
NDJSON) with its timing; `simulate()` emits sine/ramp/random/step; both wait for the queue to
drain before returning (giving up after 15 s without progress).
If the prototype reacts to a simulated signal, the wiring is right.

## Step 7 — verify like an operator

Run the **doctor** before changing code — token → URL shape → server → live session → a real
join, naming the broken step and its fix. Only ✖ lines need fixing. It checks the
**connection, not delivery**.

```bash
npx --no-install omni-adapter doctor "$CONNECT_TOKEN"     # Node project with the package installed
python -m omni_adapter doctor "$CONNECT_TOKEN"            # Python environment with the package
```

C#, Kotlin, Swift and Dart have no CLI — print the library report when `connect` fails:
C# `Doctor.FormatReport(await Doctor.RunAsync(token))` · Kotlin
`formatDoctorReport(runDoctor(token))` · Swift `formatDoctorReport(await runDoctor(credential: token))`
· Dart `formatDoctorReport(await runDoctor(token))` (TS `runDoctor`/`formatDoctorReport`, Python
`run_doctor`/`format_doctor_report` exist too).

```bash
omni-adapter watch "$CONNECT_TOKEN" 'line.*' --count 1 --timeout 5 | wc -l   # liveness: count lines
omni-adapter send  "$CONNECT_TOKEN" line.temp 23.5                          # exit 0 = server acked
```

`watch` takes the credential first and exits 0 even with zero messages after `--timeout` — count
its output lines. `buildReport(credential)` packs doctor + the server's public info (no credential, only a
fingerprint); its diagnostics list is empty unless you pass the client's `diagnostics()`.

## Common failures → causes

| Symptom | Cause / fix |
| --- | --- |
| `RoomEnded` / 404 "no live session" | Nobody has the stage open. An Editor/Owner opens it in Connect, or use a key created with **Let adapters start a session** turned on. `doctor` also flags `…/connect` vs `…/api/connect`. |
| `AuthRevoked` | Key revoked/expired, or for another stage — create a new one. With `stage-locked`: the stage is closed to guests (adapters join as guests); the owner must reopen it. |
| `ConfigError … not self-describing` | A bare `ck_` without `url`/`stageId` — use the `cst_` token (create a new key if Copy is gone). |
| `ConfigError … requires a credential` | The env var is not set in *this* process (IDE run configs don't see the terminal's). |
| `StageFull` | Each connection is a seat — reuse one client. |
| `ValidationError` on send | A non-primitive value (use the JSON helper) or a bad channel. On connect with `(400)`: version skew between server and adapter. |
| `JoinTimeout` | A proxy blocking WebSocket — switch to long-polling (`transports: ['polling']`, `UsePolling`, `usePolling`). |
| Name shows `name (2)` | Another program with a different key has it, or (screen adapters) a layer with that name is bound to another key that is still valid — revoke the old key after rotating. Same key never renamed. |
| **"Connected, I publish, nothing reacts"** | In order: (1) `state` + `queuedCount` — connected with a backlog = sending held back (more than 60 messages a second, or a `TooManyRequests` backoff, which requeues silently with no `send-dropped`); (2) `client.standby`; (3) send once with `ack: true` — a fire-and-forget send (direct or queued) is never acknowledged, so a server rejection of it leaves no trace; (4) `diagnostics()` — `send-dropped` (acked sends only), `queue-overflow`, `error`, `no-listener` (recorded only by a `listeners()` call); (5) `listeners(channel)`; (6) `omni-adapter watch` the channel from a second connection. Never `latest()`. |
| Subscribed, nothing arrives | `strictRouting` on and the channel not in `receives`, or a wildcard under strictRouting. |
| `doctor` green, peer still idle | Doctor never tests delivery — channel name, routing, or the receiver. |

## Honest limits — tell the user, don't design around them silently

Soft real-time, not a control bus · sessions belong to the stage creator (or a key created with "Let adapters
start a session") · one seat per connection · no binary payloads · the replay ring is not
storage · screen broadcast is one encode per viewer and you inject the WebRTC stack · the Engine
profile (inside a Connect Omni Web View layer, no credential, no seat) is experimental, and only TS picks
it automatically from a credential-less `connect()` — Dart/C# assemble an `EngineTransport`
themselves; Python, Kotlin and Swift have no browser runtime for it.

## Reference

- `references/llms.txt` — the full guide (types, every language's quickstart and pitfalls,
  diagnostics codes, CLI, MemoryStage/replay/simulate snippets, errors table).
- `references/screens-and-remote-input.md` — screens, broadcast, remote input, Unity, AAOS.
- In the npm and NuGet packages (from the package root): `README.md`, `docs/getting-started.md`,
  `docs/COMPATIBILITY.md` (feature × language × server), and the three common runnable examples
  (quickstart · bidirectional + roster · resilience) under `examples/` — start from those instead
  of writing one from scratch.
