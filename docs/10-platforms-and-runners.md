# 10 · Platforms & runners

The platform model is deliberately small: platforms are opaque strings, runners are edges between them, and the host
is one more platform.

## 10.1 Platform tokens

A **platform token** names a software platform: `linux64`, `win32`, `win64`, `snes`, `dos`, `java_17`, … Tokens are
compared **only for equality**. There is no inheritance and no architecture arithmetic: if two platforms should be
interchangeable, a runner says so by listing both in its `GUEST`. Adding a platform is adding a string.

## 10.2 The machine platform

The **machine platform** is the token of the host the implementation runs on (`linux64`, `win64`). Every runner chain
must reach it (I5). An implementation MUST know its machine platform and MUST refuse to run content it cannot route
to it.

## 10.3 Runners are edges

A runner is a node whose default entry has a non-empty `GUEST`:

```json
{ "EXEC": [ { "LABEL": "snes9x", "HOST": "linux64", "GUEST": ["snes"], "EXE": "%RunnerMount%/snes9x",
              "ARGS": ["-fullscreen", "%Content%"] } ] }
```

For each guest platform it is a directed edge **guest → host**: it consumes content of the guest platform and is
itself a program of the host platform.

- Proton: `win32 → linux64`, `win64 → linux64`.
- A native emulator: `snes → linux64`, one hop.
- A Windows-only emulator: `vortex → win32`, which must itself be chained on to the machine.
- The native terminal: `linux64 → linux64`.

The edges of every available runner form the **platform graph**; running content is finding a path through it
(chapter 11). A package never names its runner (I9).

## 10.4 Anchors and drives: where a runner puts things

Packages name places by anchor (chapter 03 §3.4). A runner entry says where each anchor is in its guest, and where
each guest drive is in its runtime layout:

```jsonc
"GUEST_ROOTS": { "%GameDir%":      "C:\\%PackageUID%",
                 "%UserProfile%":  "C:\\users\\steamuser",
                 "%Documents%":    "C:\\users\\steamuser\\Documents",
                 "%AppData%":      "%UserProfile%\\AppData\\Roaming",
                 "%SysDir32%":     "C:\\windows\\syswow64" },
"DRIVES":      { "C:": "%PrefixRoot%/drive_c" }
```

- **`GUEST_ROOTS`** maps each anchor (a `%Name%` key) to its **guest path**: what a value naming it must say (a
  registry `InstallPath`, an argument, a config line). An anchor may be spelled from another anchor, in any order;
  anchors spelled from each other in a circle are refused.
- **`DRIVES`** maps each guest drive to a path in the runtime layout. A guest path the launch *places* (a content
  `TARGET`, an edit's file, a kept file) is first substituted, then moved to where its drive lives: the longest
  matching drive wins, drives compare without case, a path under no mapped drive stays as written. Every anchor's
  guest path MUST be on a drive `DRIVES` lays out (a validator refuses `E:` with no `E:` drive).
- A value the run **wrote** (a captured registry value `C:\Program Files (x86)\X\x.ax`) is re-spelled from the most
  specific anchor before it is stored, so the user's state stays valid under another runner.

The anchors in effect are the **boundary** runner's (chapter 11 §11.5). Validators check packages against the anchors
the runners in the library declare; runners that disagree about an anchor's guest path are warned about.

A native engine uses the same mechanism to give a game a folder: doukutsu-rs declares
`"GUEST_ROOTS": {"%GameDir%": "C:\\cavestory"}` and `"DRIVES": {"C:": "game"}`, so a package's `%GameDir%/data`
lands at `game/cavestory/data` in its runtime.

A runner's own layers are written in its own layout, with relative targets (`FILES/files/lib64`, `FILES/pfx`).

## 10.5 The runner's build

A runner needs binaries: the Proton tree, the emulator. Those bytes are the runner's **build**: the content layers of
the runner's own resolution (its node and everything it contains, folded like any node).

```jsonc
{ "LABEL": "geproton_7_55", "LAYERS": [
    { "NODE": "bafkrei…geproton_7_55_files" },                                  // the Proton tree
    { "NODE": "bafkrei…dxvk_2_3", "TARGET": "FILES/files/lib64/wine/dxvk" },     // a shared library, placed
    { "DIR": "%DefaultPfxDir%", "TARGET": "FILES/pfx" },                         // the prefix, assembled at launch
    { "ENV": { "STEAM_COMPAT_DATA_PATH": "%RuntimePath%", "LD_LIBRARY_PATH": null } },
    { "EXEC": [ { "LABEL": "geproton_runner", "HOST": "linux64", "GUEST": ["win32", "win64"], … } ] } ] }
```

Two kinds of content end up in a runner's resolution:

- **Build content**: real bytes (zips, deltas, files). This is the runner, mounted read-only at `%RunnerMount%`.
- **Runtime-sourced content**: a `DIR` (or other layer) whose path is a variable resolved only at launch
  (`%DefaultPfxDir%`, `%WineSys32Dir%`). It assembles the **game's** runtime, typically the Wine prefix, and is placed
  **beneath** the game's content, so a package's own DLLs win over the runner's builtins at the same path.

The boundary runner's other facts reach the launch too: its `ENV` (chapter 07 §7.3), its `KEEP` (chapter 13), and its
edits, registry and DLL policy, which apply after the game's. A package and its runner SHOULD NOT write the same
address.

## 10.6 `custom`: a game's own engine

One token is reserved. **`custom`** names content that only its own package can run: a game whose engine ships with
it, such as an open-source reimplementation (GemRB for Baldur's Gate, doukutsu-rs for Cave Story, VCMI for Heroes
III), a source port, or a static recompilation of a console game. The engine is a runner **in the game's package**
with `GUEST ["custom"]`, and the game's entry says `HOST "custom"`. A package does not invent a platform token for
its engine.

A `custom` edge is **package-scoped**: a runner serving `custom` serves only launches from a package folder that holds
a copy of it, everywhere a runner is matched: the chain search, a pinned chain, the per-step choices in a launch
window, the play gate and validation. Another package's engine never runs this one, even when it would be a shorter
route. The same engine node may be shipped in several packages (GemRB for every Infinity Engine game); it serves each
of them.

An engine for a real, general platform (a console emulator) is an ordinary shared runner instead.

*(VidyaGod: `ManifestModel::ServesPlatform`; a node's copies are `Node::Copies`.)*

## 10.7 Availability

A runner is **available** when it can execute here:

- a runner whose `EXE` is a bare system command (`wine`) is available when the command resolves on the host's
  search path;
- a runner that ships a build is available once its build is hydrated (every real content file of its resolution
  present; a runtime-sourced layer is not build content and is not counted). Its `EXE` is a build path
  (`%RunnerMount%/proton`, `vortexemu.exe`) and is never looked up on the search path;
- an empty or `%Content%` `EXE` (a pass-through) is always available.

The chain search prefers **installed** runners and falls back to any runner that could be downloaded only when the
installed ones cannot carry the launch (chapter 11 §11.4).

## 10.8 Build placement

- **Separate mount (default).** The build mounts read-only at its own mount point, `%RunnerMount%`; the game's
  content mounts under the runner's `CONTENT_ROOT` in the runtime.
- **Unified (`UNIFIED_RUNTIME: true`).** The build is mounted into the runtime itself, beneath everything, and
  `%RunnerMount%` is the runtime root. For runners that must see their own files and the game's in one tree.

A runner nested inside another runner's guest mounts its build inside the boundary's content root (chapter 11
§11.6).

## 10.9 Prefix-generating runners

A runner with `PREFIX_GENERATE: true` (Wine, Proton) runs content inside a Windows prefix: a `drive_c` tree and
registry hives. The prefix is **declared in the runner's own layers** and assembled at launch with no special case:
runtime-sourced `DIR` layers mount the default prefix and the system DLL folders its build ships (`%DefaultPfxDir%`
at `FILES/pfx`, `%WineSys32Dir%` at `FILES/pfx/drive_c/windows/system32`), and `EDIT` layers write the few marker
files the runner expects. Nothing is copied, no `wineboot` runs, and the prefix's sources are never modified: each
session's changes go to the runtime's writable layer, and the registry hives are composed per launch (chapter 14
§14.4). Its `CONTENT_ROOT` puts content inside the `C:` drive (`pfx/drive_c/%PackageUID%`); the part before
`drive_c` is `%PrefixRoot%`.

Next: [Runner chaining](11-runner-chaining.md).
