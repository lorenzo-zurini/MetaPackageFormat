# 08 · Variables & CustomVar

Almost every string field in a node may contain `%TOKEN%` placeholders that are expanded at resolve/launch time: runner
args and env, layer paths and targets, edit values, persistence paths, exec fields. This chapter defines the
substitution engine, the built-in token table, and the `CustomVar` layer that lets packages declare their own tokens.

## 8.1 The substitution engine

`%NAME%` is the placeholder syntax. The algorithm:

```
function substitute(text, vars):           # vars: NAME -> value
    out = ""
    i = 0
    while i < len(text):
        start = indexOf(text, '%', i)
        if start == -1: out += text[i:]; break            # no more tokens
        end = indexOf(text, '%', start+1)
        if end == -1:                                      # unmatched '%'
            out += text[i:]; break                         # stop, keep remainder verbatim
        out += text[i:start]
        name = text[start+1 : end]
        if name in vars: out += vars[name]                 # replace
        else:            out += "%" + name + "%"           # unknown ⇒ leave in place (diagnosable)
        i = end + 1
    return out
```

Rules a conforming engine MUST follow:

- Tokens are delimited by paired `%`. Scanning is left-to-right, non-overlapping.
- An **unmatched** `%` (no closing `%`) stops substitution; the remainder is preserved verbatim. (So a stray `%` doesn't
  corrupt the rest of the string.)
- An **unknown** token (name not in the map) is **left in place** as `%NAME%` — *not* replaced with empty. This makes a
  missing variable visible and diagnosable rather than silently dropping content.
- Substitution is a single pass and does **not** recurse into the substituted value. (Values that themselves contain
  tokens — e.g. a `CustomVar` `DEFAULT` of `%ScreenWidth%` — are resolved when *that* variable is resolved, against the
  map as it stands at that point. Built-in path tokens never contain other tokens, so order is not load-bearing for
  them.)

(VidyaGod: `VarSubst::StringVariableSubstitution`.)

## 8.2 Built-in token table

These tokens are provided by the runtime's variable map. Values are concrete strings (paths are absolute host paths
unless noted). Custom tokens (§8.3) are layered on top and MAY shadow a built-in.

| Token | Value |
|-------|-------|
| `%PackageUID%` | The launchable's `UID` (the package/save key; e.g. `"299"`). Used heavily in guest paths (`C:\%PackageUID%\…`). |
| `%PackageName%` | The launchable's human title (from `META.TITLE`). |
| `%PackagePath%` | The bundle directory of the launchable. |
| `%GameName%` | The selected game's title (usually = `%PackageName%`). |
| `%UMUID%` | The `META.UMUID` value (a Steam/UMU app id some Wine runtimes use), or `"0"`. |
| `%ScreenWidth%` / `%ScreenHeight%` | The host's primary display geometry in pixels (or `"0"` if unknown/headless). |
| `%RuntimePath%` | The single overlay mount root (what a runner's prefix/env points at). |
| `%ProgramPath%` | `RuntimePath` joined with the content root — where the game's content actually sits. |
| `%Content%` | The **absolute host path** of the launch target (`ProgramPath` + the launchable's `CONTENTPATH`). |
| `%ContentPath%` | The launch target **relative to `ProgramPath`** (the launchable's `CONTENTPATH`). Authors compose guest paths from this, e.g. `C:\%PackageUID%\%ContentPath%`. |
| `%WorkDirPathRelative%` / `%WorkDirPathComplete%` | The working directory (relative to `ProgramPath`, and absolute). |
| `%RunnerMount%` | Where the boundary runner's build is mounted (a separate mount, or the runtime root for a unified runner). Runner executables reference this: `%RunnerMount%/proton`. |
| `%RunnerRuntimePath%` | A runner's resolved external runtime locator, if any. |
| `%TempPath%` | The per-session scratch root (under which `RuntimePath` etc. are nested). |
| `%WriteLayerPath%` | The ephemeral writable overlay branch. |
| `%UserDataPath%` | The durable `USERDATA` store (chapter 7). |
| `%DefPrefixPath%` | The generated Wine prefix artifact directory (chapter 13). |
| `%DefaultData%` | The default-data layer directory (base edits; chapter 6/13). |
| `%REL%` | (Only inside a runner's `GUEST_PATH` template — chapter 11.) The content-root-relative path being translated into the runner's guest namespace. |

(VidyaGod: `ContainerParams::GetVariablesMap` in `launchparams.cpp`.) Implementations MAY expose additional tokens but
MUST provide at least the above for portability of existing packages.

## 8.3 `CustomVar` — package-declared knobs

A `CustomVar` layer declares a **user-tweakable variable** that becomes a `%KEY%` token usable anywhere substitution
runs. It is how a package exposes options (resolution, language, a CD-key field, a logging toggle) without bespoke
schema, and how a game can pass values to its runner.

| Field | Meaning |
|-------|---------|
| `TYPE` | `"CustomVar"` |
| `KEY` | the bare token name. Becomes `%KEY%`. (No namespacing — see §8.5.) |
| `LABEL` | human label for the UI control. Falls back to `KEY`. |
| `VARTYPE` | `"options"` \| `"string"` \| `"number"` \| `"bool"` \| `"dword"` \| `"qword"` \| `"random"`. Drives both the UI control and the value encoding (§8.4). |
| `DEFAULT` | the default value (a string; may itself contain tokens, expanded on resolve). |
| `OPTIONS` | for `"options"` / `"random"`: an array of `{ "LABEL": "...", "VALUE": "..." }` choices. |
| `DISPLAY` | bool (default `true`). `false` = a hidden knob: it resolves and is usable as `%KEY%`, but no UI control is shown (a packager-internal variable). |

Examples:

```json
{ "TYPE": "CustomVar", "KEY": "PROTON_LOG", "LABEL": "Proton logging",
  "VARTYPE": "options", "DEFAULT": "0", "DISPLAY": true,
  "OPTIONS": [ { "LABEL": "Off", "VALUE": "0" }, { "LABEL": "On", "VALUE": "1" } ] }

{ "TYPE": "CustomVar", "KEY": "CDKEY", "LABEL": "CD Key",
  "VARTYPE": "string", "DEFAULT": "" }

{ "TYPE": "CustomVar", "KEY": "FULLSCREEN", "LABEL": "Fullscreen",
  "VARTYPE": "bool", "DEFAULT": "1" }     // → "dword:00000001" when used in a RegEdit
```

### Resolution priority

A `CustomVar`'s value is resolved highest-priority-first:

1. **Explicit override** — a per-launch override (CLI `--var KEY=VALUE`, or the launch UI's control value).
2. **Saved user setting** — the value the user last chose, persisted per-package.
3. **`DEFAULT`** — the declared default.

The chosen raw value is then `%TOKEN%`-expanded and **encoded** per `VARTYPE` (§8.4), and bound to `%KEY%`.

(VidyaGod: `ResolveCustomVariables` in `launchresolver.cpp`; saved values live under the package's user settings.)

## 8.4 `VARTYPE` value encoding

After substitution, the value is translated to its storage form by `VARTYPE`. This is what lets a human type `10` and get
a correct Wine-registry `dword`:

| `VARTYPE` | UI control | Encoding of the resolved value |
|-----------|-----------|--------------------------------|
| `string` | text field | unchanged |
| `number` | spin box | unchanged (decimal string) |
| `options` | dropdown of `OPTIONS` | unchanged (the chosen `VALUE`) |
| `bool` | checkbox | `"1"/"true"/"yes"` → `dword:00000001`, else `dword:00000000` |
| `dword` | spin box | decimal → `dword:XXXXXXXX` (8 hex digits, 32-bit) |
| `qword` | spin box | decimal → `hex(b):XX,XX,…,XX` (8 bytes, little-endian) |
| `random` | (none) | a fresh choice from `OPTIONS` is picked **each launch** (an explicit override still wins) |

`dword`/`qword`/`bool` produce Wine `.reg` encodings, so a `CustomVar` of those types feeds directly into a `RegEdit`
`KEYVALUES` entry. (VidyaGod: `VarSubst::TranslateCustomVarValue`.)

`random` is special: it has no UI control and is re-rolled every launch from its `OPTIONS` — useful for things like a
random port or a rotating identifier. Because re-rolling is the point, an implementation MUST NOT surface a `random` var
as an editable control whose (possibly empty) value would override the roll.

## 8.5 Bare keys, sharing & `FORCEVARS`

`CustomVar` keys are **bare and global within a launch** — there is no per-node namespace. This is intentional: it lets
one logical knob be shared. Two important consequences:

- **A game can seed a runner's knob.** Runners expose tweakable variables (e.g. Proton's `%PROTON_LOG%`, `%UMUID%`) as
  `%KEY%` tokens in their `ENV`/`ARGS`. A game package can *set* those by declaring a `CustomVar`/value with the same
  `KEY` (or via a saved value), so it passes env/args to its runner with no bespoke field. This game-seeds-runner pattern
  is the `FORCEVARS` idea: the game forces values onto the runner through the shared token namespace.
- **Same-key collapse.** Because keys are bare, the same logical option declared by several nodes collapses to one knob.
  In a closure, later nodes (higher priority) win when they re-declare a key. Within a multi-version runner, scoping
  resolution to the *active* version's nodes prevents an inactive version's knob from leaking.

Runner-contributed knobs SHOULD be surfaced in the UI distinctly (e.g. grouped/labelled as runner options) so a user can
tell a game option from a runtime option, but they resolve through the same mechanism.

## 8.6 When variables are expanded

Substitution happens at the moment each consumer is built, against the variable map as it then stands:

- **Layer/subcomponent strings** (paths, targets, edit values, persistence paths) — expanded when the closure's layers
  are collected, after `CustomVar`s are resolved (so custom tokens are available).
- **Runner `EXECUTABLE`/`ARGS`/`ENV`** — expanded at launch composition (chapter 11).
- **`CONTENT_ROOT`, exec `CONTENTPATH`/`WORKDIR`/`EXEARGS`** — expanded during path derivation.

An implementation MUST resolve `CustomVar`s **before** expanding the layers that reference them, so a `%KEY%` in a layer
path/value sees its value. (VidyaGod orders `ResolveCustomVariables` before `BuildSubComponentsArray`.)

Next: [The EXEC block](09-exec.md).
