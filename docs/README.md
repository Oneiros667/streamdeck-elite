# Linux support documentation

This directory defines the planned Linux port of the Elite Dangerous Stream
Deck plugin to [StreamController](https://streamcontroller.github.io/docs/latest/).

> **Status:** design and implementation plan only. The repository does not yet
> contain a runnable StreamController plugin.

The existing plugin remains the Windows/.NET Framework implementation under
`Elite/`. Linux support will be a native Python StreamController plugin in the
same repository. It will reuse the behaviour, assets (after a licence audit),
Elite binding names, and telemetry semantics of the Windows plugin, but not its
Elgato SDK or Win32 input layer.

## Documents

- [Architecture](architecture.md) — current Windows design, target Linux design,
  ownership boundaries, and proposed source layout.
- [Linux runtime](linux-runtime.md) — Proton paths, configuration precedence,
  Flatpak considerations, uinput, and troubleshooting expectations.
- [Porting plan](porting-plan.md) — milestones, feature-parity matrix,
  acceptance criteria, risks, and decisions that must be made before release.
- [Icon design system](icon-design.md) — proposed original visual language,
  coverage, state semantics, and artwork provenance constraints.

Repository-wide instructions for coding agents are in [`AGENTS.md`](../AGENTS.md).

## Scope

The port is intended to provide:

- StreamController actions for normal keys, Stream Deck + dials, and touch
  events where the original action supports them;
- images and labels driven by Elite's journal and companion JSON files;
- automatic use of the commander's keyboard bindings, including modifiers;
- safe tap and hold input on X11 and Wayland;
- configurable paths with best-effort Steam/Proton discovery;
- visible diagnostics when telemetry, bindings, or input permissions are not
  available.

The first release does not need to discover every Wine manager automatically.
Manual paths are the compatibility fallback. Automatic Elgato profile switching
also does not map directly to StreamController and is tracked as a later page
switching investigation.

Reproducing the Elgato `.streamDeckPlugin` archive-generation step is outside
the Linux port's scope. The existing Windows project should remain buildable,
but the Linux release will use StreamController's native distribution model.

## StreamController baseline

The design is based on StreamController's current Python plugin model:

- a plugin registers one or more actions;
- actions use `ActionCore`, event assigners, and declared key/dial/touch support;
- settings use StreamController's settings and configuration UI APIs;
- shared or dependency-heavy work can run in one plugin-wide backend process;
- a FakeDeck can exercise the plugin without physical hardware.

StreamController remains a beta project. At the start of implementation, record
the exact app version and PluginTemplate commit used, and update the compatibility
notes whenever that baseline changes.

## Primary references

- [StreamController plugin introduction](https://streamcontroller.github.io/docs/latest/plugin_dev/intro/)
- [StreamController key concepts](https://streamcontroller.github.io/docs/latest/plugin_dev/key_concepts/)
- [Current PluginTemplate](https://github.com/StreamController/PluginTemplate)
- [ActionCore reference](https://streamcontroller.github.io/docs/latest/plugin_dev/bases/ActionCore_py/)
- [Plugin-wide backend guide](https://streamcontroller.github.io/docs/latest/plugin_dev/modify_template/add_a_plugin_backend/)
- [Plugin manifest guide](https://streamcontroller.github.io/docs/latest/plugin_dev/submitting/manifest/)
- [Original streamdeck-elite repository](https://github.com/mhwlng/streamdeck-elite)
- [MagicMau EliteJournalReader](https://github.com/MagicMau/EliteJournalReader)
- [EDAssets catalogue](https://edassets.org/) and
  [source repository](https://github.com/Venefilyn/EDAssets)
