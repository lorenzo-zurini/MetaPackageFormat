# 09 · Invocation: `ENTRYPOINTS`

What to run is a **facet of the node that is run**: `ENTRYPOINTS`, a list of entries. Each entry is a **variant**
of the node — *Play*, *Multiplayer*, *Editor*, *Dedicated server* — and the launcher offers a node's entries as
its variants. A node with `ENTRYPOINTS` is launchable; an entry that also lists `GUEST` platforms makes the node a
**runner**. There are no exec nodes, no runner types and no hard-coded launchers: an executor's entire behaviour
is its entry's fields.

One field decides which reading an entry has:

| `GUEST` | The entry is | It contributes |
|---------|--------------|----------------|
| absent or empty | a **launchable** variant — an entry point | a target to run, and the `HOST` platform needed to run it |
| non-empty | a **runner** — a `GUEST → HOST` executor edge | a launcher definition: executable, args, env, content root, prefix policy |

They are one shape because *a launchable is a runner that provides nothing*: the terminal link of the chain.

**Execution is not transitive.** The exec of a launch is the **selected entry of the selected node** and nothing
else. A node under it (`1.16.5 OVER [1.16.4]`; a base under a variant) contributes no path, no args, no env:
what travels down `OVER` is bytes, not execution. Different versions of a game have different entrypoints for a
reason — each version carries its own.

## 9.1 Entry fields

| Field | Type | Applies to | Meaning |
|-------|------|-----------|---------|
| `LABEL` | string | both | The variant's name in the picker. Entries of one node MUST have distinct labels; a nameless entry reads as its index. |
| `HOST` | string | both | The platform this entry **needs** (`win32`/`snes`/`linux64`/`java8`/…). For a launchable it is the platform of its content; for a runner it is the platform the runner program itself is. The runtime finds a chain that reaches it ([ch. 11](11-runner-chaining.md)). **Required.** |
| `GUEST` | array of string | runner | The platforms this entry **provides**. Non-empty ⇒ runner. |
| `PATH` | string | both | The **one universal target**: for a launchable, the path (VFS-root-anchored, like a layer `TARGET`) of whatever is to be run or loaded — an executable, a ROM, a data root, or empty for content a runner finds on its own (a JRE runner given a classpath in `ARGS`). For a runner, the program to execute (`%RunnerMount%/proton`, `umu-run`, or empty for a native pass-through). Exposed to runners as `%ContentPath%` (relative) and `%Content%` (absolute). `%TOKEN%`-expanded. |
| `ARGS` | array of string | both | The argument vector, **one element per argv entry**, kept verbatim (spaces safe). `%TOKEN%`-expanded. |
| `WORKDIR` | string | launchable | Working directory, relative to `ProgramPath`. Defaults to the *directory of `PATH`*, or `ProgramPath` if `PATH` is empty. |
| `RECOMMENDED` | bool | launchable | The node's **default entry** (the first `RECOMMENDED`, else the first entry), and the preferred variant of its card. |
| `RUNNER` | string | launchable | A soft, package-side runner preference — a runner node's handle. Never a pin; the user overrides in the picker. |
| `ENV` | object | both | Environment variables to set, `name → value`, `%TOKEN%`-expanded. A launchable's `ENV` is merged **over** its runner's on a shared key, and a launchable's `ENV_REMOVE` beats a runner's `ENV` — the game states what THIS program needs. |
| `ENV_REMOVE` | array of string | both | Environment variable names to remove before launch (applied before `ENV`). |
| `CONTENT_ROOT` | string | runner | Where the game's content mounts under the runtime root. `""` = the root; `"pfx/drive_c/%PackageUID%"` places content inside a Wine prefix's `C:` drive. Non-empty ⇒ a **namespace boundary** ([ch. 11](11-runner-chaining.md)). |
| `PREFIX_GENERATE` | bool | runner | The runner needs a one-time generated Wine/Proton prefix ([ch. 13](13-runtime-model.md)). |
| `UNIFIED_RUNTIME` | bool | runner | Mount the runner's build *into* the game runtime root rather than at a separate mount. |
| `GUEST_PATH` | string | runner | A template mapping a content-root-relative path to this runner's **guest** path, for cross-namespace nesting ([ch. 11](11-runner-chaining.md)). Derived from `CONTENT_ROOT` for Wine-family runners when absent. |

An entry MUST NOT carry `WHEN`: an entrypoint is never conditional (it becomes the node's identity at index time
and is never re-evaluated). Use `TOGGLE` on the node, or two entries.

## 9.2 Launchables

```json
{ "LABEL": "Single Player", "OVER": ["…woxl-pristine"],
  "TILE": { "UID": "17260", "TITLE": "Wipeout XL" },
  "LAYERS": [ … ],
  "ENTRYPOINTS": [ { "LABEL": "Play", "HOST": "win32", "PATH": "%PrefixRoot%/drive_c/%PackageUID%/Wipeout2.exe",
                     "ARGS": [], "RECOMMENDED": true } ] }
```

A node with several entries is several variants of one mount:

```json
"ENTRYPOINTS": [
  { "LABEL": "Play",   "HOST": "win32", "PATH": "%PrefixRoot%/drive_c/%PackageUID%/game.exe" },
  { "LABEL": "Editor", "HOST": "win32", "PATH": "%PrefixRoot%/drive_c/%PackageUID%/editor.exe" } ]
```

A **launchable graft** — a mod loader such as SKSE or Forge — is a node with `ENTRYPOINTS` that is `OVER` the
version(s) it works on. It is picked from the card like any launchable; its any-of group is the version choice:

```json
{ "LABEL": "SKSE 2.2.6", "OVER": [["…sk-640", "…sk-659"]], "LAYERS": [ … ],
  "ENTRYPOINTS": [ { "LABEL": "SKSE", "HOST": "win64", "PATH": "%PrefixRoot%/drive_c/%PackageUID%/skse64_loader.exe" } ] }
```

Picking it selects it **and** the version it inherits its identity through (the chosen member), so the version's
own mods are offered alongside ([ch. 12](12-resolution.md)).

> **`ARGS` is an array of argv entries, never a command line.** Each element survives substitution verbatim, so an
> argument containing a space is expressible and nothing is re-split.

`PATH` is **case-sensitive** against the real content and is validated as such ([ch. 15](15-validation.md)).

## 9.3 Runners

An entry with a non-empty `GUEST` *is* a launcher definition. A runner is a node like any other: its build content
is its `LAYERS` (or the `LAYERS` of what it is `OVER`), and the runtime mounts that build separately at
`%RunnerMount%` unless `UNIFIED_RUNTIME`.

### Native pass-through

A runner with an **empty** `PATH` (or `PATH: "%Content%"`) is a *pass-through*: as the innermost runner it runs
the content's own executable directly; as an outer wrapper it forwards the inner command unchanged. This is the
mechanism behind the **native terminal** ([ch. 11](11-runner-chaining.md)) and behind native Linux games.

```json
{ "LABEL": "native-passthrough",
  "ENTRYPOINTS": [ { "HOST": "linux64", "GUEST": ["linux64"], "PATH": "%Content%", "ARGS": [] } ] }
```

### Override invocation (tooling)

An implementation MAY run a runner with an *override target* instead of the content (a setup step such as
generating a prefix: run `wineboot` instead of the game). The content-bearing `ARGS` are dropped and the override
appended after the launcher verb. A runtime mechanism, not a package field.

## 9.4 How a launch command is built (single runner)

```
program   = the runner entry's PATH (substituted)                # e.g. /…/RUNNER/proton
args      = [ each runner ARGS element, substituted ]            # e.g. waitforexitandrun  C:\7804\aom.exe
          + [ each launchable ARGS element, substituted ]        # e.g. xres=1920  yres=1080
env       = host env, minus runner ENV_REMOVE, plus runner ENV,
            minus launchable ENV_REMOVE, plus launchable ENV (substituted)
          + WINEDLLOVERRIDES (joined DLLOVERRIDES, if a prefix runner)
workdir   = WORKDIR (or fallback)
```

Exactly one process is started with that program/args/env. The nested case (runner chains) is
[chapter 11](11-runner-chaining.md).

## 9.5 Worked examples

**Proton (Wine-family, prefix):**
```json
{ "LABEL": "GE-Proton 10-30", "OVER": ["…geproton_build"],
  "ENTRYPOINTS": [ {
    "HOST": "linux64", "GUEST": ["win32", "win64"],
    "PATH": "%RunnerMount%/proton",
    "ARGS": ["waitforexitandrun", "C:\\%PackageUID%\\%ContentPath%"],
    "ENV": { "STEAM_COMPAT_DATA_PATH": "%RuntimePath%", "SteamGameId": "%PackageUID%" },
    "ENV_REMOVE": ["LD_LIBRARY_PATH"],
    "CONTENT_ROOT": "pfx/drive_c/%PackageUID%",
    "PREFIX_GENERATE": true } ] }
```

**A native-Linux emulator (no prefix, content at root):**
```json
{ "LABEL": "snes9x", "OVER": ["…snes9x_build"],
  "ENTRYPOINTS": [ { "HOST": "linux64", "GUEST": ["snes"], "PATH": "snes9x", "ARGS": ["-fullscreen", "%Content%"] } ] }
```

**Minecraft — 903 versions, zero exec nodes.** Each version is one launchable carrying its own entry; a JRE runner
serves the `java*` platforms:
```json
{ "LABEL": "1.16.5", "OVER": ["…v1.16.4"], "TILE": { "UID": "320", "TITLE": "Minecraft" },
  "LAYERS": [ { "FORM": "delta", "PATH": "1.16.5.vgdelta", "TARGET": "…" } ],
  "ENTRYPOINTS": [ { "LABEL": "1.16.5", "HOST": "java8", "PATH": "",
                     "ARGS": ["-cp", "%MC_CP%", "net.minecraft.client.main.Main", "…"] } ] }
```
Adding 1.21 is one node. Nothing under 1.21 runs, nothing under it is offered as a mod for it.

Next: [Platforms & runners](10-platforms-and-runners.md).
