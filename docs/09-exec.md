# 09 · Invocation: `DeclareExec` & `DeclareRunner`

Invocation is declared by two identity layers ([ch. 3](03-roles.md)): **`DeclareExec`** on a launchable (*what* to run)
and **`DeclareRunner`** on a runner (*how* to run things). Both are pure data — there are no special-cased "runner
types" or hard-coded launchers; an executor's entire behavior is its `DeclareRunner` fields.

## 9.1 `DeclareExec` — the launch target

A launchable's `DeclareExec` describes the target *within its mounted content* (plus the content's `PLATFORM`, and the
variant's `LABEL`/`RECOMMENDED`). The runner (chapter 11) decides how to actually execute it; the launchable only points
at it.

| Field | Type | Meaning |
|-------|------|---------|
| `PLATFORM` | string | The platform of *its content* (`win32`/`snes`/`linux64`/…) — what the runtime must find a runner chain for. |
| `CONTENTPATH` | string | The **one universal target**: the path, relative to the content/program mount, of whatever is to be run or loaded — an executable, a ROM, a data root, or empty for self-contained content. Exposed as `%ContentPath%` (relative) and `%Content%` (absolute). `%TOKEN%`-expanded. |
| `WORKDIR` | string | Working directory, relative to `ProgramPath`. If absent, defaults to the *directory of `CONTENTPATH`*, or `ProgramPath` if `CONTENTPATH` is empty. `%TOKEN%`-expanded. |
| `EXEARGS` | string | Arguments belonging to the content itself (not the runner), space-separated, appended after the runner's own args. `%TOKEN%`-expanded. |
| `LABEL` / `RECOMMENDED` | string / bool | The variant's name + default flag in its game's picker ([ch. 3 §3.4](03-roles.md)). |

```json
{ "TYPE": "DeclareExec", "PLATFORM": "win32",
  "CONTENTPATH": "aom.exe", "WORKDIR": "",
  "EXEARGS": "xres=%ScreenWidth% yres=%ScreenHeight%" }
```

> **Composition.** `DeclareExec` **composes along the launch closure** (field-level last-wins, the launch node
> highest-priority — like CustomVar/Persist): a base/parent node can supply `CONTENTPATH`/`WORKDIR` and a variant or mod
> override `EXEARGS`. The effective EXEC is the merge across the closure.

For a ROM, `CONTENTPATH` is the ROM file; for a native binary, the binary; for a self-contained engine that finds its own
data (e.g. some emulators/launchers), `CONTENTPATH` MAY be empty and the runner runs without a target.

> **Why one path, not "EXE vs ROM vs data."** A launchable doesn't know whether it'll be run by Wine, an emulator, or
> natively. It exposes a single relative path; each runner composes the right invocation from `%Content%` /`%ContentPath%`
> in *its* `ARGS`. The launchable describes *what its content is*, the runner describes *how to run that kind of content*.

`CONTENTPATH` is **case-sensitive** against the real content and is validated as such (chapter 5 §5.5, chapter 15).

## 9.2 `EXEARGS` splitting

`EXEARGS` is a single string split on spaces into argument tokens, after substitution. (This is simple whitespace
splitting; arguments containing spaces are not currently expressible via `EXEARGS` — put such a target in the runner's
`ARGS` array, which preserves elements verbatim.) Each token is appended after the runner's composed command.

## 9.3 Runner invocation fields

A runner's `DeclareRunner` layer (besides `HOST`/`GUEST`) *is* its launcher definition. The full invocation field set:

| Field | Type | Default | Meaning |
|-------|------|---------|---------|
| `EXECUTABLE` | string | `""` | The program to execute. May reference a mounted build (`%RunnerMount%/proton`), a system command (`umu-run`, `wine`), a build-relative path (`vortexemu.exe`, for a nested inner runner), or be **empty** / `%Content%` for a **native pass-through** (run the inner command / the content directly). `%TOKEN%`-expanded. |
| `ARGS` | array of string | `[]` | The runner's argument vector. The author composes the launch target into it using `%Content%`/`%ContentPath%` (e.g. Proton's `["waitforexitandrun", "C:\\%PackageUID%\\%ContentPath%"]`, an emulator's `["-fullscreen", "%Content%"]`). Elements are kept verbatim (spaces safe). `%TOKEN%`-expanded. |
| `ENV` | object | `{}` | Environment variables to set, `name → value`. Values `%TOKEN%`-expanded (e.g. `"STEAM_COMPAT_DATA_PATH": "%RuntimePath%"`). |
| `REMOVE_ENV` | array of string | `[]` | Environment variable names to *remove* before launch (applied before `ENV`), e.g. `["LD_LIBRARY_PATH"]` to strip a bundling launcher's library path so the runner loads host libraries. |
| `CONTENT_ROOT` | string | `""` | Where the game's content mounts under the runtime root. `""` = the root (native/emulator); `"drive_c/%PackageUID%"` (Wine) or `"pfx/drive_c/%PackageUID%"` (Proton) places content inside the prefix's `C:` drive. A non-empty `CONTENT_ROOT` makes the runner a **namespace boundary** (chapter 11). `%TOKEN%`-expanded. |
| `PREFIX_GENERATE` | bool | `false` | The runner needs a one-time generated Wine/Proton prefix (chapter 13). When true, `CONTENT_ROOT` SHOULD route through `drive_c`. |
| `UNIFIED_RUNTIME` | bool | `false` | Mount the runner's build *into* the game runtime root rather than at a separate mount. Rare; for runtimes that must share the game's filesystem view. When false, the build mounts separately and is reached via `%RunnerMount%`. |
| `GUEST_PATH` | string | derived | A template mapping a content-root-relative path to this runner's **guest** path, used for cross-namespace nesting (chapter 11). Uses `%REL%` for the relative path plus other tokens (e.g. `"C:\\%PackageUID%\\%REL%"`). If absent, it is **derived from `CONTENT_ROOT`** for Wine-family runners. |

### Native pass-through

A runner with an **empty** `EXECUTABLE` (or `EXECUTABLE: "%Content%"`) is a *pass-through*: as the innermost runner it
runs the content's own executable directly; as an outer wrapper it forwards the inner command unchanged. This is the
mechanism behind the **native terminal** (chapter 11) and behind native Linux games. Such a runner declares no
`CONTENT_ROOT` and no prefix.

```json
{ "NODE_ID": "native-passthrough",
  "LAYERS": [ { "TYPE": "DeclareRunner", "HOST": "linux64", "GUEST": ["linux64"],
                "EXECUTABLE": "%Content%", "ARGS": [], "ENV": {}, "REMOVE_ENV": [] } ] }
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
program   = EXECUTABLE (substituted)                         # e.g. /…/RUNNER/proton
args      = [ each ARGS element, substituted ]               # e.g. waitforexitandrun  C:\7804\aom.exe
          + [ each EXEARGS token from the launchable ]       # e.g. xres=1920 yres=1080
env       = host env, minus REMOVE_ENV, plus ENV (substituted)
          + WINEDLLOVERRIDES (joined DllOverride values, if a prefix runner)
workdir   = WORKDIR (or fallback)
```

and exactly one process is started with that program/args/env. The general, *nested* case (runner chains) generalizes
this and is specified in [chapter 11](11-runner-chaining.md).

## 9.5 Worked examples

**Proton (Wine-family, prefix):**
```json
"EXEC": {
  "EXECUTABLE": "%RunnerMount%/proton",
  "ARGS": ["waitforexitandrun", "C:\\%PackageUID%\\%ContentPath%"],
  "ENV": { "STEAM_COMPAT_DATA_PATH": "%RuntimePath%", "SteamGameId": "%PackageUID%" },
  "REMOVE_ENV": ["LD_LIBRARY_PATH"],
  "CONTENT_ROOT": "pfx/drive_c/%PackageUID%",
  "PREFIX_GENERATE": true
}
```

**umu (Wine-family, slightly different layout):**
```json
"EXEC": {
  "EXECUTABLE": "umu-run",
  "ARGS": ["C:\\%PackageUID%\\%ContentPath%"],
  "ENV": { "WINEPREFIX": "%RuntimePath%", "GAMEID": "%UMUID%", "PROTON_VERB": "waitforexitandrun" },
  "REMOVE_ENV": ["LD_LIBRARY_PATH"],
  "CONTENT_ROOT": "drive_c/%PackageUID%",
  "PREFIX_GENERATE": true
}
```

**A native-Linux emulator (no prefix, content at root):**
```json
"EXEC": { "EXECUTABLE": "snes9x", "ARGS": ["-fullscreen", "%Content%"], "ENV": {}, "REMOVE_ENV": [] }
```

**A win32-only emulator meant to be nested under Wine (build-relative exe, no prefix):**
```json
"EXEC": { "EXECUTABLE": "vortexemu.exe", "ARGS": ["%Content%"], "ENV": {}, "REMOVE_ENV": [] }
```

The first emulator (`snes9x`) is a native-Linux build — content for its platform runs in a single bridge hop. The second
(`vortexemu.exe`) is a Windows-only emulator for a platform with *no* native runner, so it must be chained under a Wine
runtime (chapter 11). The chain resolver routes each platform through whatever runners exist — a native one-hop where one
is available, a cross-platform chain only where it isn't.

Next: [Platforms & runners](10-platforms-and-runners.md).
