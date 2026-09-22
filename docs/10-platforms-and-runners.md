# 10 · Platforms & runners

The platform model is the engine behind cross-platform execution. It is deliberately tiny: platforms are opaque strings,
runners are edges between them, and the host is just another platform.

## 10.1 Platform tokens

A **platform token** is an opaque string naming a software platform. Examples: `"linux64"`, `"win32"`, `"win64"`,
`"snes"`, `"nes"`, `"macos"`, `"arm64"`. The format ascribes no structure to them — they are compared **only for
equality**. There is no inheritance, no "win32 is a subset of win64," no architecture math. If two things should be
interchangeable, a runner declares it by listing both in `GUEST`.

This opacity is a feature: adding a platform is adding a string, and the whole system (resolution, validation, UI) works
unchanged.

## 10.2 The machine platform

The **machine platform** is the platform token of the host actually running the implementation (e.g. `"linux64"`). It is
the target every runner chain must reach (invariant I5). A conforming implementation MUST expose its machine platform and
MUST refuse to "run" content it cannot route to that platform.

(VidyaGod: `MachinePlatform()` — currently the constant `"linux64"`, to be replaced by real OS/arch detection when other
hosts are supported. The format already supports any host token; only detection is implementation-specific.)

## 10.3 Runners as platform-graph edges

A runner is a node whose `ENTRYPOINTS` entry declares:

```json
{ "ENTRYPOINTS": [ { "HOST": "<the platform the runner itself runs on>",
                     "GUEST": ["<platform it can run>", "<…>"], … } ] }
```

This is a set of **directed edges** `guest → host`: for each `g` in `GUEST`, the runner is an edge from `g` to `HOST`.
Read it as "this runner *consumes* guest-platform content and *produces* a host-platform process."

- Proton: edges `win32 → linux64` and `win64 → linux64`.
- A native-Linux emulator (e.g. snes9x): edge `snes → linux64` — content runs in one hop.
- A Windows-only emulator (e.g. the hypothetical VortexEmu, chapter 11): edge `vortex → win32` — its host is win32, so it
  must itself be chained onward to the machine.
- The native terminal: edge `linux64 → linux64` (a self-loop; the universal executor).

The union of all available runners' edges is the **platform graph**. Running content is finding a path through it from
the content's platform to the machine platform (chapter 11).

## 10.4 The runner build is content

A runner needs *binaries* to do its job — the Proton tree, the emulator executable. Those bytes are the runner's
**build**: the `LAYERS` of the runner node itself, or of what it is `OVER` (a shared Wine tree, a DXVK build).

```json
{ "LABEL": "GE-Proton 10-30", "OVER": ["dxvk_2_4"],
  "LAYERS": [ { "FORM": "zip", "PATH": "GE-Proton10-30.zip", "SOURCE": { "TYPE": "ipfs", "CID": "Qm…" } } ],
  "ENTRYPOINTS": [ { "HOST": "linux64", "GUEST": ["win32", "win64"], "PATH": "%RunnerMount%/proton" } ] }
```

The runner's build is the runner's closure resolved like any closure (chapter 12). Heavy, shareable bytes stay
content-addressed and deduplicated exactly like a game's: a Proton build is fetched and seeded like anything else.

## 10.5 Which content builds the runner, and which assembles the prefix

Two kinds of `Content` end up in a runner's closure, and the distinction is a property of **the content**, not of which
node it sits on:

- **Build content** — real bytes on disk. This *is* the runner, and it mounts separately at `%RunnerMount%`.
- **Prefix-assembly content** — content whose `PATH` is **runtime-sourced** (a `%variable%` that only resolves to a real
  path at mount time, e.g. `%RunnerMount%/files/share/default_pfx`). It contributes to the **game's** runtime, laying
  down the wine prefix.

When "a runner" was one node with an ordered layer array, these were told apart by *which node they sat on*: assembly
layers happened to live on the runner node itself. One node per layer makes node membership meaningless, so the real
property is tested directly.

> **Prefix-assembly content is PREPENDED, not appended.** The wine prefix (the default prefix plus the
> `system32`/`syswow64` builtin DLLs) is the **base system**: it must sit *beneath* the game and library content so a
> package's native override DLLs win over wine's builtins at the same path. Appending it puts wine's builtins on top and
> silently masks every `syswow64`/`system32` override a package ships. (`FileEdit`/`RegEdit`/`DllOverride` are
> order-independent — separate passes — so those are appended.)

A runner's whole closure contributes its order-independent edits (`DllOverride`/`RegEdit`/`FileEdit`) to the game
runtime, exactly as a library pinned by the *game* does — a runner may legitimately pin a media stack that installs
native DirectShow filters and switches `winegstreamer` off. The runner chain is not less capable than the content chain.

## 10.6 Runner availability & the installed model

A runner is **usable on this machine** when it can actually execute. Two kinds:

- **PATH runner** — its `PATH` is a bare system command (`wine`, `umu-run`). Usable iff that command resolves on the
  host's executable search path.
- **Build-shipping runner** — its `PATH` resolves from its mounted build (`%RunnerMount%/proton`), or is a
  build-relative path (e.g. `vortexemu.exe`). Usable iff its build is **hydrated** (every real on-disk `Content` node in its closure present locally; the runtime-sourced prefix-assembly ones are not build content and must not be counted — see §10.5)
  and, if it generates a prefix, its prefix artifact exists.

Crucially, **a runner that ships its own build is "available" even if its `PATH` is not a system command** — the exe
lives in the build, not on `PATH`. An implementation MUST treat "ships a build" as a form of availability; otherwise a
nested Windows-only emulator (`PATH: "vortexemu.exe"`) would be wrongly judged missing. (VidyaGod: `RunnerWrapper::
ExecutableAvailable` for the PATH case, OR a build-presence check; `RunnerInstalled`/`RunnerAvailable`.)

(VidyaGod resolves `%var%`-bearing or empty executables as "available" — they come from a mount or are pass-throughs —
and checks bare commands against the host `PATH`, falling back to the build-presence check.)

## 10.7 Build placement: separate mount vs unified

A build-shipping runner mounts its build read-only:

- **Separate mount (default):** the build mounts at its own mount point, reached via `%RunnerMount%` (so
  `PATH: "%RunnerMount%/proton"`). The game's content mounts separately under `CONTENT_ROOT`.
- **Unified (`UNIFIED_RUNTIME: true`):** the build folds *into* the game runtime root (lowest priority), sharing one
  filesystem view with the content. `%RunnerMount%` then resolves to the runtime root. Use only when the runtime must see
  the game's files and its own in one tree.

For cross-namespace nesting (a runner running *inside* another runner's guest fs), inner runner builds mount as *content*
within the boundary runner's content root — see [chapter 11 §11.6](11-runner-chaining.md).

## 10.8 Prefix-generating runners

A runner with `PREFIX_GENERATE: true` (Wine/Proton) needs a one-time generated prefix: a `drive_c` + registry-hive tree
created by running the runner's own launcher with a `wineboot` target instead of the game. The prefix is generated once,
stored as a read-only artifact alongside the runner, and reused for every launch (it is never mutated; per-session
changes go to the overlay's writable layer). `CONTENT_ROOT` for such a runner SHOULD include `drive_c` so content lands
inside the `C:` drive; a validator warns if not. Prefix generation and the resulting layer stack are specified in
[chapter 13](13-runtime-model.md).

Next: [Runner daisy-chaining](11-runner-chaining.md).
