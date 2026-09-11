# 09 · Invocation: `DeclareExec`

Invocation is declared by **one** type ([ch. 3](03-roles.md)): `DeclareExec`. It says *what to run* and, when it also
lists `GUEST` platforms, *how to run other things*. There are no special-cased "runner types" and no hard-coded
launchers; an executor's entire behaviour is its `DeclareExec` fields.

One field decides which reading applies:

| `GUEST` | The node is | It contributes |
|---------|-------------|----------------|
| absent or empty | a **launchable** — an entry point | a target to run, and the `HOST` platform needed to run it |
| non-empty | a **runner** — a `GUEST → HOST` executor edge | a launcher definition: executable, args, env, content root, prefix policy |

They are one type because *a launchable is a runner that provides nothing*: the terminal link of the chain. Splitting
them meant saying the same thing twice with two field vocabularies (`PLATFORM` vs `HOST`), and chaining then needed a
special case for the ends.

## 9.1 Fields

| Field | Type | Applies to | Meaning |
|-------|------|-----------|---------|
| `HOST` | string | both | The platform this node **needs** (`win32`/`snes`/`linux64`/…). For a launchable it is the platform of its content; for a runner it is the platform the runner program itself is. The runtime finds a chain that reaches it ([ch. 11](11-runner-chaining.md)). |
| `GUEST` | array of string | runner | The platforms this node **provides**. Non-empty ⇒ runner. |
| `PATH` | string | both | The **one universal target**: for a launchable, the path (relative to its content mount) of whatever is to be run or loaded — an executable, a ROM, a data root, or empty for self-contained content. For a runner, the program to execute (`%RunnerMount%/proton`, `umu-run`, or empty for a native pass-through). Exposed to runners as `%ContentPath%` (relative) and `%Content%` (absolute). `%TOKEN%`-expanded. |
| `ARGS` | array of string | both | The argument vector, **one element per argv entry**, kept verbatim (spaces safe). A launchable's args are appended after the runner's composed command; a runner composes the launch target into its own with `%Content%`/`%ContentPath%`. `%TOKEN%`-expanded. |
| `WORKDIR` | string | launchable | Working directory, relative to `ProgramPath`. If absent, defaults to the *directory of `PATH`*, or `ProgramPath` if `PATH` is empty. `%TOKEN%`-expanded. |
| `LABEL` / `RECOMMENDED` | string / bool | launchable | The variant's name + default flag in its tile's picker ([ch. 3 §3.4](03-roles.md)). |
| `ENV` | object | runner | Environment variables to set, `name → value`, `%TOKEN%`-expanded (e.g. `"STEAM_COMPAT_DATA_PATH": "%RuntimePath%"`). |
| `ENV_REMOVE` | array of string | runner | Environment variable names to *remove* before launch (applied before `ENV`), e.g. `["LD_LIBRARY_PATH"]` to strip a bundling launcher's library path so the runner loads host libraries. |
| `CONTENT_ROOT` | string | runner | Where the game's content mounts under the runtime root. `""` = the root (native/emulator); `"pfx/drive_c/%PackageUID%"` (Proton) places content inside the prefix's `C:` drive. A non-empty `CONTENT_ROOT` makes the runner a **namespace boundary** ([ch. 11](11-runner-chaining.md)). `%TOKEN%`-expanded. |
| `PREFIX_GENERATE` | bool | runner | The runner needs a one-time generated Wine/Proton prefix ([ch. 13](13-runtime-model.md)). When true, `CONTENT_ROOT` SHOULD route through `drive_c`. |
| `UNIFIED_RUNTIME` | bool | runner | Mount the runner's build *into* the game runtime root rather than at a separate mount. Rare; for runtimes that must share the game's filesystem view. When false, the build mounts separately and is reached via `%RunnerMount%`. |
| `GUEST_PATH` | string | runner | A template mapping a content-root-relative path to this runner's **guest** path, for cross-namespace nesting ([ch. 11](11-runner-chaining.md)). Uses `%REL%` plus other tokens (e.g. `"C:\\%PackageUID%\\%REL%"`). If absent it is **derived from `CONTENT_ROOT`** for Wine-family runners. |

## 9.2 A launchable

```json
{ "NODE_ID": "aom_game", "TYPE": "DeclareExec", "PARENTS": ["aom_registry", "aom"],
  "HOST": "win32", "PATH": "aom.exe",
  "ARGS": ["xres=%ScreenWidth%", "yres=%ScreenHeight%"] }
```

> **`ARGS` is an array of argv entries, never a command line.** Each element survives substitution verbatim, so an
> argument containing a space is expressible and nothing is re-split. (An earlier generation used a single
> space-separated `EXEARGS` *string* that the resolver split; a package that means "one argument with a space in it"
> could not say so.)

> **Composition.** `DeclareExec` **composes along the launch closure** (field-level last-wins, the launch node
> highest-priority — like `CustomVar`/`Persist`): a base/parent node can supply `PATH`/`WORKDIR` and a variant or mod
> override `ARGS`. The effective exec is the merge across the closure.

For a ROM, `PATH` is the ROM file; for a native binary, the binary; for a self-contained engine that finds its own data
(e.g. some emulators/launchers), `PATH` MAY be empty and the runner runs without a target.

> **Why one path, not "EXE vs ROM vs data."** A launchable doesn't know whether it'll be run by Wine, an emulator, or
> natively. It exposes a single relative path; each runner composes the right invocation from `%Content%`/`%ContentPath%`
> in *its* `ARGS`. The launchable describes *what its content is*, the runner describes *how to run that kind of content*.

`PATH` is **case-sensitive** against the real content and is validated as such (chapter 5 §5.5, chapter 15).

## 9.3 A runner

A `DeclareExec` with a non-empty `GUEST` *is* a launcher definition. The fields are in §9.1; what follows is how
they behave.

### Native pass-through

A runner with an **empty** `PATH` (or `PATH: "%Content%"`) is a *pass-through*: as the innermost runner it
runs the content's own executable directly; as an outer wrapper it forwards the inner command unchanged. This is the
mechanism behind the **native terminal** (chapter 11) and behind native Linux games. Such a runner declares no
`CONTENT_ROOT` and no prefix.

```json
{ "NODE_ID": "native-passthrough", "TYPE": "DeclareExec",
  "HOST": "linux64", "GUEST": ["linux64"], "PATH": "%Content%", "ARGS": [] }
```

### Override invocation (tooling)

An implementation MAY run a runner with an *override target* instead of the content (for setup steps like generating a
prefix: run `wineboot` instead of the game). When overriding, the content-bearing `ARGS` (those containing
`%Content%`/`%ContentPath%`) are dropped and the override is appended after the launcher verb, leaving the rest of the
runner command intact. This is how prefix generation reuses the exact runner definition (chapter 13 §13.4). It is a
runtime mechanism, not a package field.

## 9.4 How a launch command is built (single runner)

For the classic single-runner case (e.g. a Windows game under Proton), the launch process is:

```
program   = the runner's PATH (substituted)                  # e.g. /…/RUNNER/proton
args      = [ each runner ARGS element, substituted ]        # e.g. waitforexitandrun  C:\7804\aom.exe
          + [ each launchable ARGS element, substituted ]    # e.g. xres=1920  yres=1080
env       = host env, minus ENV_REMOVE, plus ENV (substituted)
          + WINEDLLOVERRIDES (joined DllOverride values, if a prefix runner)
workdir   = WORKDIR (or fallback)
```

and exactly one process is started with that program/args/env. The general, *nested* case (runner chains) generalizes
this and is specified in [chapter 11](11-runner-chaining.md).

## 9.5 Worked examples

**Proton (Wine-family, prefix):**
```json
{ "NODE_ID": "ge-proton10-30", "TYPE": "DeclareExec", "PARENTS": ["geproton_build"],
  "HOST": "linux64", "GUEST": ["win32", "win64"],
  "PATH": "%RunnerMount%/proton",
  "ARGS": ["waitforexitandrun", "C:\\%PackageUID%\\%ContentPath%"],
  "ENV": { "STEAM_COMPAT_DATA_PATH": "%RuntimePath%", "SteamGameId": "%PackageUID%" },
  "ENV_REMOVE": ["LD_LIBRARY_PATH"],
  "CONTENT_ROOT": "pfx/drive_c/%PackageUID%",
  "PREFIX_GENERATE": true }
```

**umu (Wine-family, slightly different layout):**
```json
{ "NODE_ID": "umu", "TYPE": "DeclareExec",
  "HOST": "linux64", "GUEST": ["win32", "win64"],
  "PATH": "umu-run",
  "ARGS": ["C:\\%PackageUID%\\%ContentPath%"],
  "ENV": { "WINEPREFIX": "%RuntimePath%", "GAMEID": "%UMUID%", "PROTON_VERB": "waitforexitandrun" },
  "ENV_REMOVE": ["LD_LIBRARY_PATH"],
  "CONTENT_ROOT": "drive_c/%PackageUID%",
  "PREFIX_GENERATE": true }
```

**A native-Linux emulator (no prefix, content at root):**
```json
{ "NODE_ID": "snes9x", "TYPE": "DeclareExec", "HOST": "linux64", "GUEST": ["snes"],
  "PATH": "snes9x", "ARGS": ["-fullscreen", "%Content%"] }
```

**A win32-only emulator meant to be nested under Wine (build-relative exe, no prefix):**
```json
{ "NODE_ID": "vortexemu", "TYPE": "DeclareExec", "HOST": "win32", "GUEST": ["vortex"],
  "PATH": "vortexemu.exe", "ARGS": ["%Content%"] }
```

The first emulator (`snes9x`) is a native-Linux build — content for its platform runs in a single bridge hop. The second
(`vortexemu.exe`) is a Windows-only emulator for a platform with *no* native runner, so it must be chained under a Wine
runtime (chapter 11). The chain resolver routes each platform through whatever runners exist — a native one-hop where one
is available, a cross-platform chain only where it isn't.

Next: [Platforms & runners](10-platforms-and-runners.md).
