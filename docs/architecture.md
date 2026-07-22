# Architecture

## Status

This is the target design for the Linux port. Names and module boundaries may be
refined during scaffolding, but changes should preserve the ownership rules in
this document.

## Current Windows plugin

The existing program is a single .NET Framework 4.8 executable loaded by the
Elgato Stream Deck application.

```text
Elite Dangerous files
  StartPreset + *.binds XML
  Status.json / Cargo.json / NavRoute.json / Journal.*.log
               |
               v
Elite/Program.cs
  path discovery + watcher lifecycle + binding deserialization
               |
               v
EliteData (shared mutable state) <---- EliteJournalReader
               |
               v
Elite/Buttons/*  <---->  BarRaider / Elgato SDK
       |
       v
WindowsInput + user32.dll ----> focused Elite Dangerous window
```

The useful behaviour and the platform dependencies are interleaved:

<!-- markdownlint-disable MD013 -->

| Area | Behaviour to preserve | Windows-specific mechanism to replace |
| --- | --- | --- |
| Bindings | Read active General, Ship, SRV, and On Foot presets; prefer keyboard bindings; preserve modifiers | `%LocalAppData%`, registry discovery, DirectInput-to-Win32 injection |
| Telemetry | Status flags, pips, GUI focus, fire group, route, cargo, alarms, and journal transitions | Known-folder Win32 call and .NET `FileSystemWatcher` |
| Actions | Static, held, toggle, alarms, FSD/FSS, route, power, limpets, fire groups, and dials | BarRaider base classes, Elgato connection and property inspectors |
| Rendering | State images, text, colours, pips, alarms, GIFs | `System.Drawing` and Elgato `SetImageAsync` |
| Audio | Per-action click and error sounds | NAudio |
| Profile switching | Choose an Elgato profile from game state | Elgato profile files and `SwitchProfileAsync` |

<!-- markdownlint-enable MD013 -->

The legacy source is the behavioural reference, not a portable library. In
particular, `InputSimulatorPlus` is Windows-only and `EliteData` is not a safe
cross-process state boundary.

## Target Linux plugin

StreamController actions live in the application process, while one plugin-wide
backend owns long-lived Elite and Linux resources.

```text
                         StreamController process
                    +-------------------------------+
deck input -------->| ActionCore actions             |
                    | - event assigners              |
                    | - settings UI                  |
                    | - image/label rendering        |
                    | - cached state view            |
                    +---------------+---------------+
                                    | local backend API
                                    v
                    +-------------------------------+
                    | Elite plugin backend           |
                    |                               |
Proton/Wine files --> path + bindings + telemetry   |
                    |             |                 |
                    |             v                 |
                    |       immutable snapshot      |
                    |             |                 |
                    |       command planner         |
                    |             |                 |
                    |       serialized input ------>| /dev/uinput
                    +-------------------------------+
```

### Frontend responsibilities

- register every action and its supported input types;
- translate StreamController input events into domain commands;
- expose plugin and action settings;
- render only from a cached, complete state snapshot;
- show degraded/error state supplied by backend health;
- subscribe/unsubscribe cleanly as pages are cached or actions are removed.

The frontend must not tail journals, parse bindings, create uinput devices, or
sleep in callbacks. All actions on all decks share the same backend state.

### Backend responsibilities

- resolve configured and discovered journal/bindings paths;
- watch/retry/reload `StartPreset*.start` and active `*.binds` files;
- build initial state from companion files and relevant journal history;
- watch `Status.json`, `Cargo.json`, `NavRoute.json`, and journal rotation;
- project raw events into a typed state snapshot plus per-source health;
- resolve action names to keyboard chords;
- serialize tap/down/up commands and guarantee release on failure/shutdown;
- expose diagnostics without leaking journal content.

The backend should expose coarse operations such as `snapshot()`,
`binding_health()`, `tap(function)`, `key_down(function)`, and
`key_up(function)`. Do not expose file watchers or uinput objects over the
process boundary.

## Proposed source layout

StreamController store plugins require `manifest.json` in the repository root,
so the Linux entry point should be rooted here rather than under `Elite/`.

```text
main.py                     StreamController PluginBase entry point
manifest.json               StreamController/store manifest
about.json                  maintainer and support metadata
attribution.json            code and asset provenance
requirements.txt            small frontend-only dependencies, if any
__install__.py              backend environment creation, if needed
actions/
  base.py                   common state subscription/render helpers
  static.py
  repeating_static.py
  toggle.py
  power.py
  ...
backend/
  backend.py                BackendBase entry point and lifecycle
  requirements.txt
  elite/
    bindings.py             StartPreset and .binds parsing
    binding_catalog.py      action ID -> Elite binding selector
    paths.py                manual configuration and discovery
    telemetry.py            watchers and journal tailing
    state.py                typed snapshot and state reducer
    commands.py             action domain logic and serialization
    input.py                abstract sink plus Linux uinput implementation
    health.py               structured diagnostics
assets/                     audited Linux plugin assets
locales/                    StreamController localization files
store/                      store thumbnail and related metadata
tests/
  fixtures/                 synthetic Elite files
  unit/
  integration/
```

The exact module names are not a contract. The separation of path discovery,
parsing, state reduction, input, and UI is.

The future root `manifest.json` is for StreamController. The existing
`Elite/manifest.json` remains the Elgato manifest and has a different schema.

## State model

A snapshot should contain both game values and provenance:

```text
EliteSnapshot
  sequence
  produced_at
  status_timestamp
  journal_timestamp
  flags / flags2 / gui_focus / pips / fire_group
  route / cargo / alarm data
  sources:
    status: healthy | stale | missing | unreadable
    journal: replaying | live | stale | missing | unreadable
    cargo: healthy | stale | missing | unreadable
    nav_route: healthy | stale | missing | unreadable
```

Reducers accept parsed records and return a complete new snapshot. They must be
testable without GTK, StreamController, threads, or filesystem watchers.

Actions should render a degraded indicator when required sources are unhealthy.
For example, a Static action may still be able to send an input when the journal
is unavailable, while a Toggle action cannot claim that its displayed state is
current. Health is therefore source-specific, not one global online boolean.

## Binding model

Keep the original Elite function name as the stable domain identifier. Resolve
it at execution time to a typed chord:

```text
BindingChord
  source: primary | secondary
  key: Linux event code
  modifiers: ordered Linux event codes
  keyboard_layout: optional Elite layout hint
```

Resolution rules:

1. choose `Primary` when `Device="Keyboard"`;
2. otherwise choose `Secondary` when it is a keyboard;
3. otherwise return an explicit unsupported/unbound result;
4. map each Elite `Key_*`/DIK-style name through one reviewed table;
5. never reinterpret joystick or mouse bindings as keyboard input.

## Action adapter pattern

Each action should have three small pieces:

1. a settings schema (function, images, colours, sounds, and action-specific
   options);
2. event assigners that call domain commands;
3. a pure rendering decision that maps snapshot + settings to media and labels.

Common rendering and backend error handling belong in an action base/helper.
Action classes must not reproduce path, binding, or input-driver logic.

## Lifecycle

1. `PluginBase` registers action holders and launches one backend.
2. The backend validates configuration and publishes a starting health state.
3. It loads bindings and reconstructs the current game snapshot.
4. It transitions the journal source from replaying to live.
5. Visible actions render the latest complete snapshot.
6. Input callbacks enqueue commands; they do not execute blocking input inline.
7. On page/action removal, subscriptions are removed. Held inputs are released.
8. On plugin unload, watchers stop, workers join within a bound, all keys are
   released, and the virtual input device closes.

## Packaging consequence

The StreamController store consumes a plugin repository and expects its manifest
at repository root. Keeping both implementations in one repository is possible,
but a store install will also clone the legacy source. Before publishing, verify
that the store accepts the combined repository size and layout. If it does not,
publish the Linux subtree through a dedicated release repository while keeping
this repository as the development source. Do not silently move the Windows
projects merely to satisfy packaging.
