# Linux runtime and development

## Status

This document defines runtime requirements for the planned StreamController
plugin. Commands or UI described as future behaviour are not available in this
repository yet.

## Supported runtime model

The primary target is Elite Dangerous running through Steam Proton, controlled
by StreamController on the same Linux desktop session. Manual path configuration
is part of the supported design and is the fallback for custom Steam libraries,
Flatpak boundaries, Lutris, Bottles, and other Wine layouts.

The plugin reads local game files and emits local keyboard input. It should not
require a Frontier login, a network service, or the Elgato Stream Deck software.

## StreamController development baseline

Follow the current
[StreamController development setup](https://streamcontroller.github.io/docs/latest/plugin_dev/setup/):

1. clone StreamController;
2. create and activate a virtual environment;
3. install StreamController's requirements;
4. launch it with a disposable development data directory;
5. place or link the plugin in that data directory's `plugins` directory.

The official guide recommends `--data data` so development does not modify a
Flatpak user's normal data. It also requires the plugin directory name to use
reverse-domain notation with underscores.

Once this repository contains the planned root `main.py` and StreamController
manifest, a developer can link the entire repository into the development
plugin directory under the selected plugin ID. Do not invent that ID locally:
it becomes a public, persistent identifier and must match `manifest.json`.

Use StreamController's FakeDeck for action lifecycle and rendering tests. A
FakeDeck removes the hardware dependency; it does not remove the need for
fixtures, a fake input sink, or explicit Linux permission tests.

## Configuration precedence

The planned plugin-wide settings are:

- **Journal directory** — contains `Status.json` and `Journal.*.log`;
- **Bindings directory** — contains `StartPreset.4.start` or
  `StartPreset.start` and the selected `*.binds` files;
- **Input driver** — initially the supported uinput driver, plus a disabled/fake
  choice for diagnostics and tests;
- **Auto discovery** — whether best-effort Steam/Proton discovery is allowed.

Path selection must use this precedence:

1. a valid explicit path selected by the user;
2. a previously selected discovery result that is still valid;
3. a newly discovered Steam/Proton candidate;
4. no path, with a clear diagnostic.

Auto discovery must not replace an explicit path. Clearing the explicit value is
the user's opt-in to discovery again.

The settings screen should show the effective path, its origin (manual or
discovered), validation result, last successful read time, and a rescan action.

## Steam and Proton paths

Valve documents Proton prefixes under the same Steam library as the game:

```text
<steam-library>/steamapps/compatdata/<app-id>/pfx/
```

Elite Dangerous uses Steam app ID `359320`. A common prefix is therefore:

```text
<steam-library>/steamapps/compatdata/359320/pfx/
```

Within that prefix, the Windows paths used by the existing plugin commonly map
to:

```text
# journals and companion JSON
drive_c/users/<wine-user>/Saved Games/Frontier Developments/Elite Dangerous/

# active control presets and bindings
drive_c/users/<wine-user>/AppData/Local/Frontier Developments/
  Elite Dangerous/Options/Bindings/
```

These are candidates, not hard-coded contracts. The Wine user may not be
`steamuser`, Steam may have several libraries, and the game may be installed on
another filesystem.

Typical Steam roots worth discovering include native Steam data directories and
the Flatpak Steam data directory. Discovery should parse Steam library metadata
and enumerate library roots rather than only checking `~/.steam`.

The discovery algorithm should:

1. enumerate native and Flatpak Steam roots that are readable;
2. parse `steamapps/libraryfolders.vdf` when present;
3. add every library containing `steamapps/compatdata/359320/pfx`;
4. enumerate `drive_c/users/*` below each prefix;
5. validate journal candidates by `Status.json` or `Journal.*.log`;
6. validate binding candidates by `StartPreset*.start` and/or `*.binds`;
7. rank complete pairs first and report ambiguous matches to the user;
8. canonicalize paths, but retain a useful display path for diagnostics.

Never create, edit, or delete anything in the Proton prefix during discovery.

References:

- [Valve Proton FAQ: prefix locations](https://github.com/ValveSoftware/Proton/wiki/Proton-FAQ#how-does-proton-manage-wine-prefixes)
- [Elite Dangerous on Steam](https://store.steampowered.com/app/359320/Elite_Dangerous/)

## Flatpak boundaries

There are two independent Flatpak cases:

- Steam may be a Flatpak, placing the Proton prefix below its app data
  directory;
- StreamController may be a Flatpak, limiting which host files and devices its
  plugin can access.

An existing path on the host can therefore be invisible to the plugin. Runtime
diagnostics must distinguish:

- path does not exist;
- path exists but is not readable;
- parent is outside the sandbox's visible filesystem;
- file is present but temporarily incomplete or locked.

Document the minimum filesystem/device permission only after it has been tested
against the pinned StreamController Flatpak version. Do not tell users to expose
their entire home directory when a narrower grant is sufficient, and never
recommend running StreamController as root.

## Keyboard input

The legacy plugin emits Windows scan codes. The Linux port needs a virtual
keyboard input driver. A uinput-based driver is the preferred direction because
it is independent of X11/Wayland application automation APIs.

The driver needs permission to open `/dev/uinput`. StreamController's official
[OS plugin](https://github.com/StreamController/OSPlugin) uses a udev rule and
input-group access for its hotkey action. The Elite plugin should provide a
non-destructive self-check and link to tested setup instructions. It must not
attempt to change groups, install udev rules, elevate privileges, or run itself
as root.

Input readiness should report at least:

- device node present;
- read/write access available;
- virtual device created successfully;
- required Linux key code exists in the device capability set;
- backend shutdown released all held keys.

The initial driver contract is:

```text
tap(chord)
key_down(chord, owner_token)
key_up(owner_token)
release_all()
health()
```

An owner token prevents one held action from releasing another action's chord.
Modifiers are pressed before the main key and released after it in reverse
order. Every error and cancellation path must release what it pressed.

Synthetic keyboard input is sent to the focused application. The plugin should
warn about this in its settings and must not steal focus.

## Binding loading

The bindings service watches both the active preset marker and the resolved
binding files. It should support the filename versions the Windows plugin tries:

```text
.4.2.binds
.4.1.binds
.4.0.binds
.3.0.binds
.binds
```

For Odyssey, the preset marker can select separate General, Ship, SRV, and On
Foot files. Horizons can use one preset for all categories. Whitespace and line
endings in the marker must be normalized before resolving filenames.

A binding is usable only when its primary or secondary entry has
`Device="Keyboard"`. Joystick, controller, and mouse entries remain visible as
"no keyboard binding" diagnostics; they are not converted.

When a preset changes, parse all new files into a complete replacement map, then
publish it atomically. Do not expose a mixture of old and new categories.

## Telemetry files

The backend consumes:

<!-- markdownlint-disable MD013 -->

| Source | Purpose | Update shape |
| --- | --- | --- |
| `Status.json` | flags, GUI focus, pips, fire group, position, Odyssey flags | rewritten frequently while the game runs |
| `Journal.*.log` | discrete game transitions, alarms, route targets, startup context | append-only files with rotation |
| `Cargo.json` | limpet inventory | rewritten snapshot |
| `NavRoute.json` | route and remaining jumps | rewritten snapshot |

<!-- markdownlint-enable MD013 -->

Companion JSON files can be observed between truncate and rewrite. Use bounded
retry with a short backoff for empty, partial, or transiently inaccessible
files. Preserve the last valid value while marking the source stale/unhealthy;
never replace it with a false but apparently live default.

Journal replay reconstructs state only. It must never emit keyboard input,
sounds, or other user-facing command side effects.

## Expected diagnostics

<!-- markdownlint-disable MD013 -->

| Symptom | Diagnostic category | User action |
| --- | --- | --- |
| No game state, candidate path absent | discovery/configuration | select the journal directory or rescan Steam libraries |
| Host path exists but plugin cannot see it | Flatpak filesystem permission | grant the narrow tested path and restart/reload |
| Buttons display state but presses do nothing | input permission | check `/dev/uinput` access and input-driver health |
| One action cannot send input | Elite binding | add a keyboard primary/secondary binding for that function |
| Bindings stop after changing preset | binding watcher/parser | inspect selected marker/file and trigger reload |
| A held key remains down | input safety defect | backend must call `release_all`; report as a bug with logs |
| State is old after game restart | journal rotation/health | verify the selected journal directory and source timestamps |

<!-- markdownlint-enable MD013 -->

Logs should include a stable diagnostic code, source status, and shortened path.
Raw journal lines and full home paths should be available only in an explicit
debug mode.

## Test modes

Use three separate modes:

1. **Unit tests** — synthetic files, deterministic reducer, fake clock, and fake
   input sink. Safe for CI.
2. **FakeDeck smoke tests** — real StreamController lifecycle and rendering,
   but fixture telemetry and fake/disabled input.
3. **Host integration tests** — real Proton paths and uinput. Opt-in only; never
   part of an unguarded default test command.

No automated test should launch Elite Dangerous, send input to an unrelated
focused window, or modify a commander's real binding files.
