# AGENTS.md

## Mission

This repository is a fork of `mhwlng/streamdeck-elite`. It currently contains a
Windows-only Elgato Stream Deck plugin for Elite Dangerous. The active project
goal is to add a native Linux plugin for
[StreamController](https://streamcontroller.github.io/docs/latest/) while
keeping the existing Windows plugin usable.

The Linux work is a port of behaviour and domain rules, not an attempt to run
the existing .NET Framework executable inside StreamController. StreamController
plugins are Python packages with a different lifecycle, settings model, input
API, and distribution format.

At the time this file was added, the Linux plugin had not yet been implemented.
The documents in `docs/` describe the intended architecture and delivery plan;
do not report that Linux support exists until runnable plugin code and the
relevant acceptance tests are present.

## Read before changing code

Read these files in order:

1. `docs/README.md`
2. `docs/architecture.md`
3. `docs/linux-runtime.md`
4. `docs/porting-plan.md`
5. The files involved in the feature being ported under `Elite/Buttons/`

For StreamController APIs, check the current official documentation and current
`StreamController/PluginTemplate` before coding. StreamController is in beta and
its API can change. Pin the tested application version in both code and the
compatibility documentation. New actions must derive from `ActionCore`; do not
start new code on deprecated `ActionBase` examples.

## Repository map

- `Elite/` — Windows Elgato plugin, including startup, actions, property
  inspectors, images, sounds, and the Elgato `manifest.json`.
- `Elite/Program.cs` — Windows path discovery, bindings loading, telemetry
  watcher startup, and Elgato plugin bootstrap.
- `Elite/EliteData.cs` — shared mutable game-state projection used by the
  Windows actions.
- `Elite/Buttons/` — the behavioural reference for the twelve action types.
- `EliteJournalReader/` — .NET journal and companion-file parsers/watchers.
- `InputSimulatorPlus/` — vendored Windows input simulation; it is not a Linux
  input abstraction and must not be imported into the Linux plugin.
- `docs/` — Linux design, runtime assumptions, and delivery status.
- Repository root — planned home of StreamController's `main.py`,
  `manifest.json`, `attribution.json`, `actions/`, `backend/`, and `assets/`.
  StreamController requires its manifest in the plugin repository root. Do not
  confuse the future root manifest with `Elite/manifest.json`.

## Architecture rules

1. Keep the Windows plugin and Linux plugin as separate adapters. Avoid broad
   rewrites of the legacy C# while building Linux support.
2. Put shared Linux runtime work in one plugin-wide backend: path discovery,
   `.binds` parsing, telemetry watching, state projection, serialized input,
   and health reporting. Actions should remain thin StreamController adapters.
3. Keep four domain boundaries explicit:
   - telemetry input (`Status.json`, journals, `Cargo.json`, `NavRoute.json`);
   - Elite binding lookup (`StartPreset*.start` and `*.binds` XML);
   - Linux input emission;
   - action rendering and configuration.
4. Do not let an action create its own file watchers or its own virtual input
   device. Those resources are plugin-wide and must have one owner and a clean
   shutdown path.
5. Keep action identifiers and Elite binding names in central registries. Do
   not duplicate the long function lists from the legacy HTML inspectors across
   Python actions.
6. Prefer immutable state snapshots or typed value objects at the backend/UI
   boundary. Do not expose partially updated global dictionaries to actions.
7. Preserve user-visible semantics rather than line-for-line implementation.
   Record deliberate parity changes in `docs/porting-plan.md`.

## StreamController conventions

- Register actions through `ActionHolder` with explicit support for key, dial,
  and touchscreen inputs.
- React to input with event assigners. A held action must handle both down and
  up/cancel paths.
- Use `on_ready`/`on_update` for rendering. `on_tick` may read an already-cached
  snapshot, but must not parse files, wait, sleep, or perform input injection.
- Use StreamController's settings APIs and Generative UI for ordinary options.
  Use custom GTK rows only where the standard rows cannot express the setting.
- Respect stacked actions: check image/label ownership before drawing, and test
  multi-action behaviour explicitly.
- Keep plugin-wide settings (journal path, bindings path, input driver) separate
  from per-action settings (function, images, colours, sounds).
- Keep frontend dependencies small. Put dependencies that need their own
  environment in the backend and install them using the documented backend
  virtual-environment mechanism.
- Do not call undocumented StreamController internals when a documented
  `ActionCore`, `PluginBase`, or backend API exists.

## Linux runtime and path rules

- Manual journal and bindings paths are the authoritative configuration. Auto
  discovery is best effort and must never overwrite a valid manual choice.
- Do not assume Steam is installed in the user's home directory, that there is
  only one Steam library, or that the Wine user is named `steamuser`.
- Treat native Steam, Flatpak Steam, custom Steam libraries, and other Wine
  managers as different discovery candidates. Non-Steam Wine managers can be
  supported initially through manual paths.
- Validate a candidate by its expected files, not just by a matching directory
  name. Surface which paths were selected and why a candidate was rejected.
- A Flatpak may not be allowed to read a Proton prefix or access `/dev/uinput`.
  Detect and report this as a permissions problem; do not silently treat it as
  "Elite is not installed".
- Never require StreamController or the plugin to run as root.

## Input safety

Elite bindings use DirectInput-style key names. Linux virtual input uses Linux
event codes. Keep the conversion in a reviewed, unit-tested mapping layer.

- Prefer a keyboard binding from `Primary`; fall back to a keyboard binding in
  `Secondary`. Report an unsupported action when neither side is a keyboard.
- Preserve all modifiers, press modifiers before the main key, and release in
  reverse order.
- A tap, key-down, and key-up are different operations. Repeating Static and
  dial actions rely on correct key-up behaviour.
- Serialize macros so concurrent actions cannot interleave key sequences.
- Release every key in `finally` blocks and during backend shutdown. Add a
  watchdog or equivalent recovery for an action removed while held.
- Use a Linux input interface that behaves under both X11 and Wayland. The
  expected direction is a uinput-based driver with a capability/health check.
- Synthetic input goes to the focused application. Do not add focus stealing
  or launch the game unless a separately approved feature requires it.
- Unit and CI tests must use a fake input sink. Tests that write real input must
  be opt-in and clearly marked.

The official StreamController OS plugin is GPL-3.0 and is useful as a runtime
behaviour reference for uinput permissions. Do not copy its implementation into
this MIT-licensed repository without an explicit licensing decision.

## Telemetry correctness

- Build initial state from existing companion files before publishing the first
  healthy snapshot, then continue with live updates.
- Tail journals by byte offset and handle journal rotation. Do not re-run old
  control actions while replaying telemetry.
- Elite rewrites companion JSON files. Read with bounded retry for transient
  empty, partial, locked, or replaced files.
- Order updates by the timestamp in the Elite payload where one exists. Ignore
  stale updates and expose source health/staleness.
- Missing or unreadable data must produce a visible degraded state, not a
  plausible default that looks live.
- Keep state parsing independent of rendering so fixtures can cover all state
  transitions without StreamController or a physical deck.

## Code quality and tests

For new Python code:

- use type hints, `pathlib.Path`, dataclasses/enums where they clarify domain
  values, and the standard `logging` API;
- inject clocks, file sources, and input sinks at test boundaries;
- avoid unbounded threads and sleeps; use stoppable workers with bounded joins;
- never log complete user paths at info level if a shorter diagnostic is
  sufficient, and never log journal contents by default;
- do not add runtime network calls; this plugin should operate on local Elite
  files and local input only.

Minimum tests for a ported feature:

1. fixture tests for its relevant status/journal transitions;
2. `.binds` parsing tests, including primary/secondary and modifiers;
3. fake-input tests for tap/hold/release ordering;
4. action rendering tests for healthy, disabled, and degraded state;
5. a StreamController FakeDeck smoke test for the action lifecycle.

Keep fixtures synthetic and small. Do not commit a commander's real journals,
bindings, account identifiers, or absolute home paths.

The legacy solution targets .NET Framework 4.8 and is a Windows build. Do not
claim that it was validated on Linux merely because source inspection or Python
tests passed. On Windows, restore NuGet packages before building `Elite.sln`.

## Model Routing

- Delegate bounded, multi-step read-heavy exploration, documentation comparison,
  warning triage, and evidence gathering to the project `explorer` agent.
- Delegate independent tests, lint, syntax checks, documentation checks, and
  diff verification after edits to the project `verifier` agent.
- Keep trivial lookups, single status commands, edits, integration,
  architectural and safety decisions, final verification, and commits in the
  root agent.
- Spawn support agents with no inherited history or only the minimum recent
  turns needed for their bounded task.
- Use one support agent per evidence stream. Parallelize only genuinely
  independent work when the latency reduction justifies the additional context.
- The Sol root agent must review conclusions affecting public contracts,
  security, production data, external side effects, or project safety gates.

## Change discipline

- Preserve unrelated user changes and do not reformat the legacy tree as part
  of a Linux feature.
- Update the feature matrix and milestone status in `docs/porting-plan.md` with
  every material Linux feature change.
- Update `docs/linux-runtime.md` when paths, permissions, supported install
  methods, or troubleshooting behaviour change.
- Keep attribution for the original plugin and every reused asset. Audit asset
  licences before publishing the StreamController plugin.
- When implementation and documentation disagree, fix both in the same change.
- Clearly distinguish implemented, tested behaviour from proposed design in
  issues, documentation, and handoffs.
