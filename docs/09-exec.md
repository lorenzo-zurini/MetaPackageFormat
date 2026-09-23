# 09 · Invocation: `ENTRYPOINTS`

How to run is a fact about the node, and like every fact it **folds along the chain**: a node's **effective
entrypoints** are its own `ENTRYPOINTS` if it declares any, else those of the nearest node beneath it that does
(nearest by `OVER` distance; ties by `OVER` order, and a validator warns when tied candidates differ). Own
*replaces* — the whole list, never a merge. A version chain declares its command once and every delta above
inherits it; an expansion with a new executable declares its own.

An entry that lists `GUEST` platforms makes the node a **runner**. There are no exec nodes, no runner types and
no hard-coded launchers: an executor's entire behaviour is its entry's fields.

| `GUEST` | The entry is | It contributes |
|---------|--------------|----------------|
| absent or empty | a **way to run** the node — an entry point | a target to run, and the `HOST` platform needed to run it |
| non-empty | a **runner** — a `GUEST → HOST` executor edge | a launcher definition: executable, args, env, content root, prefix policy |

They are one shape because *a game is a runner that provides nothing*: the terminal link of the chain.

**Runnable is not listed.** Any node with effective entries can be started — the CLI, an author testing a chain at
the no-CD patch level. Only a node that also declares `VARIANT` is on the shelf ([ch. 3 §3.4](03-roles.md)).

**What runs is the selected node's effective entry**, chosen by label. Nothing beneath a variant is *offered* as a
way to run it: playing 1.16.5 runs 1.16.5's effective entry (declared or inherited) and never lists 1.16.4's.
Inheritance decides what the selected node's entry *is*; it never adds a choice ([ch. 12](12-resolution.md)).

## 9.1 Entry fields

| Field | Type | Applies to | Meaning |
|-------|------|-----------|---------|
| `LABEL` | string | both | The entry's name in the picker. Entries of one list MUST have distinct labels; a nameless entry reads as its index. The **first** entry is the default. |
| `HOST` | string | both | The platform this entry **needs** (`win32`/`snes`/`linux64`/`java8`/…). For a game it is the platform of its content; for a runner it is the platform the runner program itself is. The runtime finds a chain that reaches it ([ch. 11](11-runner-chaining.md)). **Required.** |
| `GUEST` | array of string | runner | The platforms this entry **provides**. Non-empty ⇒ runner. |
| `PATH` | string | both | The **one universal target**: for a game, the path (VFS-root-anchored, like a layer `TARGET`) of whatever is to be run or loaded — an executable, a ROM, a data root, or empty for content a runner finds on its own (a JRE runner given a classpath in `ARGS`). For a runner, the program to execute (`%RunnerMount%/proton`, `umu-run`, or empty for a native pass-through). Exposed to runners as `%ContentPath%` (relative) and `%Content%` (absolute). `%TOKEN%`-expanded. |
| `ARGS` | array of string | both | The argument vector, **one element per argv entry**, kept verbatim (spaces safe). `%TOKEN%`-expanded. |
| `WORKDIR` | string | game | Working directory, relative to `ProgramPath`. Defaults to the *directory of `PATH`*, or `ProgramPath` if `PATH` is empty. |
| `RUNNER` | string | game | A soft, package-side runner preference — a runner node's handle. Never a pin; the user overrides in the picker. |
| `ENV` | object | both | Environment variables to set, `name → value`, `%TOKEN%`-expanded. A game's `ENV` is merged **over** its runner's on a shared key, and a game's `ENV_REMOVE` beats a runner's `ENV` — the game states what THIS program needs. (Environment is mutation and is slated to fold like the registry does, from every node of the chain; until then it rides the entry.) |
| `ENV_REMOVE` | array of string | both | Environment variable names to remove before launch (applied before `ENV`). |
| `CONTENT_ROOT` | string | runner | Where the game's content mounts under the runtime root. `""` = the root; `"pfx/drive_c/%PackageUID%"` places content inside a Wine prefix's `C:` drive. Non-empty ⇒ a **namespace boundary** ([ch. 11](11-runner-chaining.md)). |
| `PREFIX_GENERATE` | bool | runner | The runner needs a one-time generated Wine/Proton prefix ([ch. 13](13-runtime-model.md)). |
| `UNIFIED_RUNTIME` | bool | runner | Mount the runner's build *into* the game runtime root rather than at a separate mount. |
| `GUEST_PATH` | string | runner | A template mapping a content-root-relative path to this runner's **guest** path, for cross-namespace nesting ([ch. 11](11-runner-chaining.md)). Derived from `CONTENT_ROOT` for Wine-family runners when absent. |

`RECOMMENDED` is **not** an entry field: it is a node facet ([ch. 3 §3.4](03-roles.md)). An entry MUST NOT carry
`WHEN`: an entrypoint is never conditional. Use two entries, or two variants.

## 9.2 Games

The pristine declares the face and, when every version runs the same way, the entry; variants above it inherit:

```jsonc
{ "LABEL": "wipeout_xl", "TILE": { "UID": "17260", "TITLE": "Wipeout XL", "COVER": "WOXL_Cover.png" },
  "LAYERS": [ { "FORM": "zip", "PATH": "Wipeout XL.zip", "TARGET": "%PrefixRoot%/drive_c/%PackageUID%" } ] }

{ "LABEL": "Single Player", "VARIANT": "Single Player", "RECOMMENDED": true, "OVER": ["wipeout_xl"],
  "PATCHES": [ … ],
  "ENTRYPOINTS": [ { "LABEL": "Play", "HOST": "win32", "PATH": "%PrefixRoot%/drive_c/%PackageUID%/Wipeout2.exe" } ] }

{ "LABEL": "Multiplayer (IPX)", "VARIANT": "Multiplayer (IPX)", "OVER": ["wipeout_xl", "network_patch"],
  "ENTRYPOINTS": [ { "LABEL": "Play", "HOST": "win32", "PATH": "%PrefixRoot%/drive_c/%PackageUID%/WOLOBBY.EXE" } ] }
```

A node with several entries is several ways to run one mount:

```json
"ENTRYPOINTS": [
  { "LABEL": "Play",   "HOST": "win32", "PATH": "%PrefixRoot%/drive_c/%PackageUID%/game.exe" },
  { "LABEL": "Editor", "HOST": "win32", "PATH": "%PrefixRoot%/drive_c/%PackageUID%/editor.exe" } ]
```

**A mod loader is a graft that carries an entry**, not a variant. SKSE is `OVER` either version; you tick it on
the version you picked, that version stays selected (so its mods stay offered), and the picker gains *SKSE* as a
way to run:

```json
{ "LABEL": "SKSE 2.2.6", "OVER": [["…sk-640", "…sk-659"]], "LAYERS": [ … ],
  "ENTRYPOINTS": [ { "LABEL": "SKSE", "HOST": "win64", "PATH": "%PrefixRoot%/drive_c/%PackageUID%/skse64_loader.exe" } ] }
```

> **`ARGS` is an array of argv entries, never a command line.** Each element survives substitution verbatim, so an
> argument containing a space is expressible and nothing is re-split.

`PATH` is **case-sensitive** against the real content and is validated as such ([ch. 15](15-validation.md)).

## 9.3 Runners

An entry with a non-empty `GUEST` *is* a launcher definition. A runner is a node like any other: its build is its
closure — its own `LAYERS` and what it is `OVER` — and the runtime mounts that build separately at `%RunnerMount%`
unless `UNIFIED_RUNTIME`. Runner versions inherit like games: a build delta `OVER` the previous build keeps the
previous entry unless it declares one. A runner needs no tile; it is pickable by its `GUEST` entry, and a node
`OVER` a runner that is not a variant is a graft offered when that runner is in the chain ([ch. 12](12-resolution.md)).

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
          + [ each game ARGS element, substituted ]              # e.g. xres=1920  yres=1080
env       = host env, minus runner ENV_REMOVE, plus runner ENV,
            minus game ENV_REMOVE, plus game ENV (substituted)
          + WINEDLLOVERRIDES (joined DLLOVERRIDES, if a prefix runner)
workdir   = WORKDIR (or fallback)
```

"The game" here is the node whose entry was chosen: the selected variant, or a ticked graft that carries an
entry. Exactly one process is started with that program/args/env. The nested case (runner chains) is
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

**Minecraft — 903 variants, one tile, a handful of entries.** The pristine carries the face; the first version of
each java era declares the entry; every other version is a delta, a `VARIANT`, and nothing else:
```jsonc
{ "LABEL": "minecraft", "TILE": { "UID": "320", "TITLE": "Minecraft", "COVER": "cover.png" },
  "LAYERS": [ { "FORM": "file", "PATH": "rd-132211.jar", "TARGET": "client" } ],
  "ENTRYPOINTS": [ { "LABEL": "Play", "HOST": "java8", "PATH": "", "ARGS": ["-cp", "%MC_CP%", "net.minecraft.client.main.Main", "…"] } ] }

{ "LABEL": "1.16.5", "VARIANT": "1.16.5", "OVER": ["…v1.16.4"],
  "LAYERS": [ { "FORM": "delta", "PATH": "1.16.5.vgdelta", "TARGET": "client" } ] }

{ "LABEL": "1.17", "VARIANT": "1.17", "OVER": ["…v1.16.5"],                       // a new java era: declares
  "LAYERS": [ { "FORM": "delta", "PATH": "1.17.vgdelta", "TARGET": "client" } ],
  "ENTRYPOINTS": [ { "LABEL": "Play", "HOST": "java16", "PATH": "", "ARGS": ["-cp", "%MC_CP%", "net.minecraft.client.main.Main", "…"] } ] }
```
Adding 1.21 is one node. Nothing under it is offered as a way to run it, nothing under it is offered as a mod
for it; its entry, its face and its identity all come from beneath.

Next: [Platforms & runners](10-platforms-and-runners.md).
