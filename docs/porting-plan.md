# StreamController porting plan

## Goal

Deliver a native Linux StreamController plugin that preserves the useful Elite
Dangerous behaviour of the Windows Elgato plugin, works with Steam Proton, and
fails visibly and safely when telemetry, bindings, or virtual input are not
available.

This file is also the implementation status record. Update it when a feature is
started, tested, deferred, or deliberately changed.

## Status vocabulary

- **Not started** — design only; no runnable Linux implementation.
- **In progress** — code exists but acceptance criteria are incomplete.
- **Fixture tested** — unit/integration fixtures pass with a fake input sink.
- **Host tested** — tested in StreamController on Linux with real Proton data.
- **Complete** — documented, fixture tested, host tested, and release-ready.
- **Deferred** — intentionally outside the current release, with a reason.

## Milestones

### M0 — Freeze identifiers and compatibility baseline

Status: **Not started**

- choose the public reverse-domain plugin ID and maintainer identity;
- choose whether this combined repository can be the store repository;
- pin the StreamController version and PluginTemplate commit;
- decide the minimum supported Python and distribution environments;
- record the input dependency and licence strategy;
- audit every image and sound intended for the Linux package.

Exit criteria: decisions are documented, root manifest metadata can be written,
and no public ID or licence field is a placeholder.

### M1 — Plugin shell and health UI

Status: **Not started**

- add the root StreamController `main.py`, `manifest.json`, `about.json`, and
  `attribution.json`;
- register one diagnostic action using `ActionCore` and event assigners;
- launch one plugin-wide backend;
- add plugin settings for manual paths and discovery;
- surface backend, path, binding, telemetry, and input health;
- add a safe development mode with fixture telemetry and disabled input.

Exit criteria: plugin loads on a FakeDeck, survives page changes/cache removal,
shows a useful no-configuration state, and shuts down without orphan processes.

### M2 — Telemetry vertical slice

Status: **Not started**

- implement path validation and Steam/Proton discovery;
- parse `Status.json` into a typed immutable snapshot;
- tail/rotate journals and replay state without side effects;
- add `Cargo.json` and `NavRoute.json` snapshot readers;
- add retry, stale-source health, and deterministic reducer fixtures;
- port Toggle as a display-only action before enabling input.

Exit criteria: a Toggle action follows recorded state transitions on a FakeDeck
and shows degraded state when its source is missing or stale.

### M3 — Bindings and safe input

Status: **Not started**

- parse `StartPreset.4.start` and `StartPreset.start`;
- atomically load General, Ship, SRV, and On Foot `.binds` files;
- implement and test Elite key-name to Linux event-code mapping;
- implement serialized uinput tap/down/up and release-all behaviour;
- add permission/capability diagnostics;
- port Static and Repeating Static;
- enable input on Toggle after binding and input health are proven.

Exit criteria: modifiers, primary/secondary fallback, tap, held release, backend
shutdown, and concurrent command ordering pass fake-input tests; an opt-in host
test confirms input under both an X11 and a Wayland session where available.

### M4 — Stateful key actions

Status: **Not started**

- Power, including long press and pip rendering;
- Alarm;
- FSS;
- Hyperspace/FSD;
- Route;
- Firegroup;
- Limpet.

Exit criteria: each action has fixture tests for enabled, disabled, active/alarm,
and degraded states, plus a FakeDeck lifecycle smoke test.

### M5 — Stream Deck + and presentation parity

Status: **Not started**

- Dial input: clockwise, counter-clockwise, press/release, short touch, and long
  touch;
- FireGroup Dial;
- custom images, GIF/video behaviour supported by StreamController, labels,
  colours, and alarm/pip overlays;
- click/error audio with a Linux-compatible backend;
- stacked action and multi-action behaviour.

Exit criteria: no input callback blocks, dial holds cannot leave keys pressed,
and media/settings survive page export, duplication, and cache removal.

### M6 — Release hardening

Status: **Not started**

- decide whether Elite-state page switching replaces legacy profile switching;
- test native and Flatpak StreamController, plus native and Flatpak Steam where
  feasible;
- document narrow filesystem and uinput permissions;
- validate backend dependency installation from a clean system;
- verify store manifest, attribution, thumbnail, version mapping, and release
  notes;
- test install, update, uninstall, and rollback without touching Elite files.

Exit criteria: all promised rows in the feature matrix are Complete, known gaps
are explicit, and store packaging installs from an immutable tested commit.

## Feature parity matrix

All Linux entries are **Not started** as of 2026-07-22.

<!-- markdownlint-disable MD013 -->

| Windows feature | Behavioural source | Linux release target | Status | Notes |
| --- | --- | --- | --- | --- |
| Static Button | `Elite/Buttons/Static.cs`, `EliteKeys.cs` | First usable release | Not started | Tap active Elite keyboard binding; no game-state image feedback |
| Repeating Static | `RepeatingStatic.cs` | First usable release | Not started | Key down until release; highest stuck-key risk |
| Toggle | `Toggle.cs` | First usable release | Not started | Ship/SRV/On Foot binding selection plus telemetry state |
| Power | `Power.cs` | First usable release | Not started | Normal press, long press to four pips, pip/alarm rendering |
| Alarm | `Alarm.cs` | First usable release | Not started | Under attack, heat, and shield conditions |
| FSS | `FSS.cs` | First usable release | Not started | Engaged/enabled/disabled state and optional combat-mode transition |
| Hyperspace/FSD | `Hyperspace.cs` | First usable release | Not started | Combined, supercruise, and hyperspace functions; route text |
| Route | `Route.cs` | First usable release | Not started | Remaining jumps and next-route-system binding |
| Firegroup | `Firegroup.cs`, `EliteKeys.cs` | First usable release | Not started | Cycle to selected group and preserve disabled-state rules |
| Limpet | `Limpet.cs` | First usable release | Not started | Cargo count, fire group selection, primary/secondary fire |
| Dial | `Dial.cs` | Follow-up or first release if stable | Not started | Directional hold/release and touch gestures |
| FireGroup Dial | `FireGroupDial.cs` | Follow-up or first release if stable | Not started | Fire-group rotation and configurable press/touch actions |
| Custom images | action settings and property inspectors | First usable release | Not started | Use StreamController media APIs and stacked-action ownership |
| Animated media | action settings | Follow-up | Not started | Preserve only formats/lifecycle StreamController supports reliably |
| Click/error sounds | `Elite/Audio`, action settings | Follow-up | Not started | Do not port NAudio; audit each bundled sound licence |
| Elgato multi-actions | manifest and Static action | Follow-up | Not started | Map to StreamController stacked/multi-action semantics |
| Automatic profile switching | `Profile.cs`, `StreamDeckCommon.cs` | Research/deferred | Not started | StreamController uses pages; no one-to-one profile API assumption |
| Binding hot reload | `Program.cs` | First usable release | Not started | Atomic reload of marker plus all active category files |
| Steam/Epic default bindings discovery | `SteamPath.cs`, `EpicPath.cs` | Proton/manual-path replacement | Not started | Prefer active user binds; manual paths cover non-Steam Wine initially |

<!-- markdownlint-enable MD013 -->

## Parity policy

Parity means the same user intent and safety outcome, not identical internal
timing or a copy of known legacy limitations.

Preserve:

- action names and Elite function identifiers where practical;
- the distinction between tap and hold actions;
- active/enabled/disabled/alarm state meanings;
- Ship, SRV, and On Foot binding selection;
- keyboard-only input and modifier support;
- long-press and dial semantics;
- user-selectable media, colours, and sounds when supported.

Allowed improvements, with tests and documentation:

- clearer missing-data and missing-binding states;
- atomic configuration reload instead of partially updated globals;
- state-driven redraws instead of incidental journal-event redraws;
- cancellation-safe input and bounded worker shutdown;
- configuration that supports arbitrary Wine prefixes.

Do not silently change action behaviour solely because it is easier to express
in StreamController. Add a compatibility note and update the matrix.

## Acceptance scenarios

The following scenarios define the minimum end-to-end coverage:

1. Fresh install with no paths and no uinput access gives actionable diagnostics
   and does not crash or spin.
2. Manual fixture paths load while Elite is not running and render the latest
   known state as stale/offline rather than live.
3. Starting a new journal transitions replaying to live without duplicating old
   events or triggering commands.
4. Changing `Status.json` changes relevant toggle and power images within the
   expected update cycle.
5. Changing the active preset swaps all four binding categories atomically.
6. A primary joystick plus secondary keyboard binding uses the keyboard; two
   non-keyboard bindings show "keyboard binding required".
7. A modified chord presses modifiers first, releases the main key first, then
   releases modifiers in reverse order.
8. Removing a held action, changing pages, backend failure, and application exit
   all release its keys.
9. Two actions pressed together do not interleave their chord sequences.
10. StreamController page caching does not duplicate watchers, backends, journal
    events, or subscriptions.
11. A source becoming unreadable retains the last valid value but visibly marks
    the action stale/degraded.
12. Flatpak denial is reported as a permission boundary, not an absent game.

## Open decisions

These decisions must be explicit before their related milestone begins:

<!-- markdownlint-disable MD013 -->

| Decision | Needed by | Constraint |
| --- | --- | --- |
| Permanent plugin ID and publisher name | M0 | Must use reverse-domain notation with underscores and remain stable |
| Combined repository vs dedicated store repository | M0/M6 | StreamController requires its manifest at plugin repository root |
| Pinned StreamController and Python versions | M0 | API is beta; `ActionCore` is required for new actions |
| uinput library and backend dependency installation | M0/M3 | Must work without root and under supported packaging |
| Flatpak filesystem/device permissions | M3/M6 | Use narrow, tested permissions and actionable diagnostics |
| Asset and sound set for Linux | M0/M5 | Existing provenance is incomplete; no unaudited redistribution |
| Page switching scope | M6 | Must use documented StreamController behaviour, not emulate Elgato profiles blindly |

<!-- markdownlint-enable MD013 -->

## Main risks

<!-- markdownlint-disable MD013 -->

| Risk | Impact | Mitigation |
| --- | --- | --- |
| StreamController API changes | Plugin fails after app update | Pin/test app baseline; use documented `ActionCore`; record compatibility |
| Flatpak cannot read Proton files or uinput | State or presses unavailable | Manual paths, self-checks, narrow permission documentation, test install matrix |
| DirectInput-to-Linux mapping errors | Wrong command reaches the game | Central explicit mapping, representative fixtures, opt-in host verification |
| Key remains held | User loses keyboard control | Owner tokens, reverse release, `finally`, shutdown `release_all`, watchdog tests |
| Journal/JSON partial writes | Flicker, crashes, false state | Bounded retry, last-valid snapshot, timestamps and source health |
| Store expects a focused plugin repository | Combined repo cannot publish cleanly | Validate early; publish a generated/dedicated Linux repository if necessary |
| Reused asset has unclear licence | Release cannot legally include it | Audit before packaging; replace or omit uncertain assets |
| Port duplicates large function lists | Drift between UI and execution | One central action/binding catalog with generated UI choices |

<!-- markdownlint-enable MD013 -->

## Definition of done

A feature is Complete only when:

- behaviour and known differences are documented;
- parsing/rendering/input tests cover happy, missing, and degraded paths;
- default tests do not emit real input;
- it passes a FakeDeck lifecycle test including page removal/cache behaviour;
- its required host integration has been tested on a declared Linux environment;
- settings and diagnostics are understandable without reading logs;
- no held keys, watcher threads, or backend processes remain after shutdown;
- attribution and dependencies are release-ready.
