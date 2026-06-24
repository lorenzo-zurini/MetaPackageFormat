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

A `runner` node declares:

```json
"PLATFORM": { "HOST": "<the platform the runner itself runs on>",
              "GUEST": ["<platform it can run>", "<…>"] }
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

## 10.4 The runner build comes from PARENTS, not LAYERS

A runner needs *binaries* to do its job — the Proton tree, the emulator executable. Those bytes are the runner's
**build**, and they are supplied by the runner node's **`PARENTS`** (content nodes carrying VFS layers), **not** by the
runner node's own `LAYERS`.

```json
{ "NODE_ID": "ge-proton10-30", "ROLE": "runner",
  "PLATFORM": { "HOST": "linux64", "GUEST": ["win32","win64"] },
  "EXEC": { "EXECUTABLE": "%RunnerMount%/proton", … },
  "PARENTS": ["geproton_build"] }            // ← the build lives here

{ "NODE_ID": "geproton_build", "ROLE": "content",
  "LAYERS": [ { "TYPE": "VFSZipLayer", "PATH": "GE-Proton10-30.zip",
                "SOURCE": { "TYPE": "ipfs", "CID": "Qm…" } } ] }
```

The runner's build is the runner node's content closure (its `PARENTS`, resolved like any closure — chapter 12), **minus
the runner node itself**. This keeps runner nodes thin and puts the heavy, shareable, content-addressed bytes on ordinary
content nodes (so a Proton build is fetched/seeded/deduplicated exactly like a game).

## 10.5 Runner LAYERS are ignored (a deliberate trap)

Because the build comes from `PARENTS`, **VFS layers placed directly on a runner node are ignored** — never mounted. An
author who ships the runtime as the *runner node's own* `LAYERS` gets a runner with no build and a silent failure.
Validators MUST warn on a runner node that carries its own VFS layer and point at the fix: move the layer to a content
node and add it to the runner's `PARENTS`. (VidyaGod: this warning is in `ValidateNodeGraph`.)

This is the one place the "everything is a node" symmetry is intentionally broken, and it's worth the asymmetry: it means
a runner and the game it runs compose their builds the same way (content nodes pulled via `PARENTS`), and a runner's
description stays separable from its (large) bytes.

## 10.6 Runner availability & the installed model

A runner is **usable on this machine** when it can actually execute. Two kinds:

- **PATH runner** — its `EXECUTABLE` is a bare system command (`wine`, `umu-run`). Usable iff that command resolves on the
  host's executable search path.
- **Build-shipping runner** — its `EXECUTABLE` resolves from its mounted build (`%RunnerMount%/proton`), or is a
  build-relative path (e.g. `vortexemu.exe`). Usable iff its build is **hydrated** (every build VFS layer present locally)
  and, if it generates a prefix, its prefix artifact exists.

Crucially, **a runner that ships its own build is "available" even if its `EXECUTABLE` is not a system command** — the exe
lives in the build, not on `PATH`. An implementation MUST treat "ships a build" as a form of availability; otherwise a
nested Windows-only emulator (`EXECUTABLE: "vortexemu.exe"`) would be wrongly judged missing. (VidyaGod: `RunnerWrapper::
ExecutableAvailable` for the PATH case, OR a build-presence check; `RunnerInstalled`/`RunnerAvailable`.)

(VidyaGod resolves `%var%`-bearing or empty executables as "available" — they come from a mount or are pass-throughs —
and checks bare commands against the host `PATH`, falling back to the build-presence check.)

## 10.7 Build placement: separate mount vs unified

A build-shipping runner mounts its build read-only:

- **Separate mount (default):** the build mounts at its own mount point, reached via `%RunnerMount%` (so
  `EXECUTABLE: "%RunnerMount%/proton"`). The game's content mounts separately under `CONTENT_ROOT`.
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
