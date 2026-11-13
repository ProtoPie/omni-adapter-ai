# Screens: broadcasting and remote input

Read this only when the adapter must show a **picture** on the stage — a simulator, a
3D scene, a canvas, a Unity view. An adapter that carries values needs none of it.

The protocol lives in the adapter library (announce, answer, ICE relay, cleanup; holder gate,
direction whitelist, `[0,1]` clamp); **you inject the WebRTC session**, so the package
never gains a media dependency.

## When the adapter shows a screen

Declare it and start a broadcast:

```ts
const client = await connect({ secretKey, displayName: 'my-simulator',
                               providesView: 'stream' });   // 'none' | 'web' | 'stream'
const broadcast = startScreenBroadcast({ client, stream: canvas.captureStream(30) });
// broadcast.viewerCount · broadcast.stop()
```

**All six ship the helper**, and none of them gained a dependency — you inject one
session per viewer built on the WebRTC stack you already use. Take the factory form that
receives the server's ICE servers and put them in your peer configuration (without them the
picture reaches viewers on the same network only):

| Language | Start | Stop |
| --- | --- | --- |
| TS | `startScreenBroadcast({ client, stream })` — applies ICE servers itself | `broadcast.stop()` |
| Python | `await omni_adapter.start_screen_broadcast(client, create_session=open_session)` — `open_session(viewer, send_ice, ice_servers)` (media via the `[stream]` extra, aiortc) | `await broadcast.stop()` |
| C# | `ScreenBroadcast.Start(client, (viewer, sendIce, iceServers) => …)` — use the **3-argument** factory; the 2-argument overload drops the ICE list | `broadcast.Dispose()` |
| Kotlin | `startScreenBroadcast(client, createSession = { viewer, sendIce, iceServers, wireInput -> … })` — name the argument: a trailing lambda would go to the last parameter (`scope`) and not compile | `broadcast.stop()` |
| Swift | `await startScreenBroadcast(client: client) { viewer, sendIce, iceServers, wireInput in … }` | `await broadcast.stop()` |
| Dart | `startScreenBroadcast(client: client, createSession: (viewer, sendIce, iceServers, wireInput) async => …)` | `await broadcast.stop()` |

The helper also stops itself when the client ends.

Connect with `providesView: 'stream'` (Python `provides_view="stream"`): on a server with
adapter stream layers (unreleased) the server then creates and binds the **Adapter Stream**
layer itself when the adapter joins — in the first group placed on the stage, matched by the
name the server gave the adapter (`client.me.displayName`; the participant id changes every
run, the name does not), and a deleted layer comes back on the next join. Without the
declaration no layer is created; the helper records `stream-undeclared` once in
`diagnostics()`. A designer can also add the layer by hand: **Add → Adapters → Adapter
Stream**, then type that name.

`await client.ensureStreamLayer()` (Python `ensure_stream_layer()`, C#
`EnsureStreamLayerAsync()`) is optional — an idempotent call that returns the layer. Read
`renamed` on the result: `true` — the server gave this adapter another name than it asked for (`name (2)`): another program on a different key uses this name, or you rotated keys without revoking the old one (the layer stays bound to the old key) — rename one, or revoke the old key. The same key is never renamed. When the stream source starts or
changes after start — not per frame — call `sourceChanged()` (Python `source_changed()`, C#
`SourceChanged()`) so viewers that got no answer offer again; calls are coalesced to one
announcement per second.

Sessions are per viewer **receiver**: the web keeps one receiver per (viewer page × adapter
participant) — not per layer instance — and tags offers and ICE with its `receiverId`, so two
tabs of the same PIN guest are two sessions instead of closing each other's. The session
factory runs once per receiver — the same participant id twice — and `viewerCount` counts
receivers. Early viewer ICE (before the offer, or before the connection is set up) is queued and applied once the offer is set — candidates waiting for an offer expire after 10 s, with at most 4 waiting receivers per viewer (64 in all); the ICE-server lookup gives up after 5 s and uses the last known servers, and a failed lookup is never cached. A participant keeps at most 4 receiver sessions (a fifth evicts first a session the helper knows is not live, then the least recently negotiated — so report closures where you inject sessions). When the adapter comes back with a new participant id, every session from before
is closed. The TS helper closes a session whose connection goes `failed`/`closed`; where the
app injects the session it reports that by passing it back — `broadcast.session_closed(session)`
(Python), `SessionClosed(session)` (C#), `sessionClosed(session)` (Kotlin, Swift, Dart).

Three things to tell the user up front:

- The screen is **view-only by default** — control it with values, or accept **remote
  input** (below) on a cloud deployment.
- Cost grows with the audience (one encode per viewer). Good for 5–10 people; past that
  use a streaming URL.
- `'web'` is the other option: the adapter serves a URL and, in Connect Omni, a **Web View**
  layer (Add → Web View: the URL) embeds it, giving each viewer their own interactive copy
  with no credential in the page and no participant seat. Every Web View hands the page a
  MessagePort; what passes is the layer's **Stage messages** rules — **Receive** (channels the
  page may receive) and **Send** (channels it may send on), exact names, up to 32 each, empty =
  nothing passes. Have the page declare every channel in `receives` / `sends`: on hosts from
  2026-11-05 an editor's screen displaying the page turns the declared channels on in the rules
  (a channel an editor turned off stays off). Anything undeclared — a refused send, a wildcard,
  names past 32 — shows up on that layer as "This page asked for" (only in the editor
  displaying the page) for the designer to **Add**; older hosts list every declared channel
  there. In
  TypeScript, `connect()` with no credential inside that layer takes the Engine profile
  automatically (call it during start-up; the port is handed over once, at the iframe's load
  event). Dart (Flutter web) and C# (Blazor WASM) assemble an `EngineTransport` themselves,
  and engine control lives on that `EngineTransport`, not on the client.

## Remote input — let an editor drive the screen

**Remote input needs no code**: the stage arms it for an Editor as soon as the screen is streaming, so the adapter only has to *receive and inject*. Connect Omni servers send lease-holder notices (key adapters are registered automatically on join and told the holder): input is accepted only from the Editor holding the lease — the Connect editor claims it automatically on the first pointer-down — and PIN or share-link viewers never control. Older servers that predate the notices never announce a holder; then the helper's gate stays permissive and accepts every viewer's input. The registration API (`submitRegistrationCode` / `resumeRegistration` /
`onRemoteInputHolder`, per-language case) is for registration-code deployments.

```ts
const broadcast = startScreenBroadcast({
  client, stream,
  onRemoteInput: (msg, viewer) => inject(msg),   // injecting is the app's job — x/y are in [0,1]
});
```

The callback gives `t` (`down|move|up|cancel`), `p`, and `x`/`y` in `[0,1]`; directions are
whitelisted and coordinates clamped for you, and a holder change or channel close
synthesizes `cancel` for open pointers. Python: `on_remote_input=` plus a 4th factory
parameter `wire_input` (hand it the aiortc data channel). C#: the `Start` overload whose
factory takes 4 arguments (`…, wireInput`) plus `onRemoteInput:`; wrap your channel as
`IRemoteInputChannel`. Kotlin/Swift/Dart: pass `onRemoteInput` to `startScreenBroadcast`
and hand the viewer's data channel to `wireInput`.

**Injecting is yours** — two paths (both in `dart/example_aaos`): into your own app view
(you correct the capture↔view offset), or **whole-device** via Android
`AccessibilityService.dispatchGesture` (screen-absolute, 1:1 with the capture, drives
other apps too; re-query it on foreground resume). Only pointers today — no physical
keyboard. If the target is a vehicle, block `down`/`move` while speed > 0 but always pass
`up`/`cancel` so a held pointer is released.

**Unity checklist (each one fails silently):**
`Application.runInBackground = true` (focus loss stalls the WebRTC pump → ICE dies —
and the operator is by definition in another window) · UI for the stage goes on a
`Canvas` in `ScreenSpaceCamera` bound to the broadcast camera (`OnGUI` composites into
the back buffer, never the offscreen `targetTexture`; `Camera.main` may be a leftover
tagged camera, not the broadcast one) · queue data-channel sends until `OnOpen`
(`RTCDataChannel.Send` throws while Connecting; the helper catches it, notes
`hello … send failed` in `client.Diagnostics()` and tries again only on its 10-second
keepalive, so operating stays dead until then — and nothing reaches the `Error` event) ·
marshal adapter callbacks to the main thread (the example ships a dispatcher). Unity WebGL is not supported, IL2CPP is
unverified, and there is no UPM package yet — copy the plugin DLLs as
`csharp/examples/Unity` shows.

## When operation "does nothing" — read the trace before the code

`client.diagnostics()` answers the three-way fork directly: `input-channel` (did the
viewer's channel arrive) · `hello` (did the announce go out) · `input-dropped` (is the
holder gate discarding frames). No entries at all → the viewer never opened a channel —
check the stage side (Preview mode, editor role, and `omni-adapter doctor`'s **remote
input** line, which says whether this deployment has the operate surface at all). On
local/desktop/embed deployments there is no remote-input surface, and that is expected.

For plain data channels the same ring holds `no-listener`, but **only after you call
`listeners(channel)`** — it is never recorded on its own. It is `warn` when nobody would
receive the channel at all (a typo, or the peer is not up) and `info` when nobody
*declared* it but undeclared receivers exist — the normal case when the receiver is a
prototype, because prototypes never declare channels.

⚠️ **If the user says "only some controls respond", check the accessibility service before
looking at the code.** Without it, injection is confined to the app's own view — on a
gauge-heavy cluster UI only the handful of tappable widgets react, which reads exactly like
a broken feature. On **AAOS** the Settings app does not list third-party accessibility
services at all (only *Captions*), so the in-app "open settings" hint is a dead end, and an
emulator reboot resets the enabled list. Enable it over adb — **append** to the existing
list (`RotaryService` and `CarSafetyAccessibilityService` live there; overwriting kills the
head-unit controls), toggle `accessibility_enabled` 0 → 1 to force the bind, and pass
**`--user 10`** for any `pm grant` because AAOS runs apps as the driver profile.
`dart/example_aaos` shows the dev-only self-enable path (`WRITE_SECURE_SETTINGS`, granted
by adb).
