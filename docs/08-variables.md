# 08 · Variables & CustomVar

Almost every string field in a node may contain `%TOKEN%` placeholders expanded at resolve/launch time: runner args and
env, layer paths and targets, edit values, persistence paths, exec fields. This chapter defines the substitution engine
(including **use-site render formats**), the built-in token table, and the `CustomVar` layer that lets packages declare
their own variables.

## 8.1 The substitution engine

`%NAME%` is the placeholder syntax. A token MAY carry a **render format**: `%NAME:format%` looks up `NAME` and then
renders the value for that consumer (§8.4). The algorithm:

```
function substitute(text, vars):                 # vars: NAME -> raw value
    out = ""; i = 0
    while i < len(text):
        start = indexOf(text, '%', i)
        if start == -1: out += text[i:]; break               # no more tokens
        end = indexOf(text, '%', start+1)
        if end == -1: out += text[i:]; break                  # unmatched '%' → stop, keep remainder verbatim
        out += text[i:start]
        token = text[start+1 : end]                           # e.g. "KEY" or "KEY:dword"
        name, fmt = split_first(token, ':')                   # fmt = "" if no ':'
        if name in vars: out += (fmt == "" ? vars[name] : render(vars[name], fmt))
        else:            out += "%" + token + "%"              # unknown ⇒ leave whole token in place (diagnosable)
        i = end + 1
    return out
```

Rules a conforming engine MUST follow:

- Tokens are delimited by paired `%`, scanned left-to-right, non-overlapping.
- A token MAY contain one `:` separating `NAME` from a render `format`. `NAME` MUST NOT itself contain `:`.
- An **unmatched** `%` stops substitution; the remainder is preserved verbatim (so a stray `%` can't corrupt the rest).
- An **unknown** `NAME` leaves the **whole token** (`%NAME%` or `%NAME:fmt%`) in place — *not* empty. A missing variable
  is then visible and diagnosable (and a validator flags it — §8.7).
- Substitution is a single pass; it does not recurse into substituted values.

(VidyaGod: `VarSubst::StringVariableSubstitution`.)

## 8.2 Built-in token table

Provided by the runtime's variable map. Values are concrete raw strings (paths are absolute host paths unless noted).
Custom variables (§8.3) layer on top and MAY shadow a built-in.

| Token | Value |
|-------|-------|
| `%PackageUID%` | The launchable's `UID` (package/save key; e.g. `"299"`). Used heavily in guest paths. |
| `%PackageName%` / `%GameName%` | The launchable's human title. |
| `%PackagePath%` | The bundle directory of the launchable. |
| `%UMUID%` | `META.UMUID`, or `"0"`. |
| `%ScreenWidth%` / `%ScreenHeight%` | Host primary display geometry (`"0"` if unknown/headless). |
| `%RuntimePath%` | The single overlay mount root. |
| `%ProgramPath%` | `RuntimePath` joined with the content root — where the game's content sits. |
| `%Content%` | Absolute host path of the launch target (`ProgramPath` + `CONTENTPATH`). |
| `%ContentPath%` | The launch target relative to `ProgramPath`. |
| `%WorkDirPathRelative%` / `%WorkDirPathComplete%` | The working directory (relative / absolute). |
| `%RunnerMount%` | Where the boundary runner's build is mounted (or the runtime root if unified). |
| `%RunnerRuntimePath%` | A runner's resolved external runtime locator, if any. |
| `%TempPath%` / `%WriteLayerPath%` / `%UserDataPath%` | Session scratch root / ephemeral writable branch / durable `USERDATA`. |
| `%DefPrefixPath%` / `%DefaultData%` | Generated Wine prefix dir / the default-data (base-edit) layer dir. |
| `%REL%` | (Only inside a runner's `GUEST_PATH` template — chapter 11.) The relative path being translated. |

(VidyaGod: `ContainerParams::GetVariablesMap`.)

## 8.3 `CustomVar` — package-declared variables

A `CustomVar` is a `LAYERS` entry that declares a variable, exposed as a `%KEY%` token. It lives in `LAYERS` so it
**resolves through the same dependency-chain hierarchy** as everything else (chapter 12): a node later in the closure
re-declaring the same `KEY` overrides an earlier one — that is the override mechanism (§8.5). It is **not** a content
payload; the runtime resolves it into the variable map and never mounts it.

A `CustomVar` is **one primitive** whose role is implied by its facets:

| Field | Meaning |
|-------|---------|
| `TYPE` | `"CustomVar"` |
| `KEY` | the bare token name → `%KEY%`. MUST NOT contain `:`. |
| `DEFAULT` | the value: the fixed value for a binding, or the user-overridable initial for an option. May itself contain `%tokens%` (a **computed** variable, expanded against the map as it stands). |
| `UI` | *optional.* Its **presence** makes the variable a **user-facing option** (a control is shown in the launch dialog, the choice persists per package). Its **absence** makes it an **internal binding** (no UI; e.g. a value one node passes to another). |

The `UI` facet (when present):

| `UI` field | Meaning |
|------------|---------|
| `LABEL` | human label for the control (falls back to `KEY`). |
| `CONTROL` | the value **domain / widget**: `bool` \| `int` \| `float` \| `text` \| `enum` \| `secret`. **No encoding here** — encoding is a use-site concern (§8.4). |
| `GROUP` | optional UI section heading (e.g. `"Graphics"`) — organizes a large option set into panels. |
| `WHEN` | optional condition (`"%RENDERER% == wined3d"`) — the control is shown only when it holds, re-evaluated live as other controls change (dependent options). Supports `==` / `!=`. |
| `CHOICES` | for `enum`: `[{ "LABEL": "...", "VALUE": "..." }]`. |
| `POOL` | for `secret`: an array of values; the engine picks one **per launch** (a masked, rotating value — e.g. a CD key). A `secret` with no `POOL` is a masked free-text field. |
| `MIN` / `MAX` | numeric bounds for `int`/`float`. |
| `PATTERN` | a regex the `text` input must match. |
| `REQUIRED` | the option must be set. |

Examples:

```jsonc
// a user option (enum), grouped, with a dependent option
{ "TYPE": "CustomVar", "KEY": "RENDERER", "DEFAULT": "dxvk",
  "UI": { "LABEL": "Renderer", "CONTROL": "enum", "GROUP": "Graphics",
          "CHOICES": [ {"LABEL":"DXVK (Vulkan)","VALUE":"dxvk"}, {"LABEL":"WineD3D","VALUE":"wined3d"} ] } }
{ "TYPE": "CustomVar", "KEY": "VRAM", "DEFAULT": "1024",
  "UI": { "LABEL": "VRAM (MB)", "CONTROL": "int", "GROUP": "Graphics", "WHEN": "%RENDERER% == dxvk", "MIN": 64, "MAX": 8192 } }

// an internal binding (no UI) — e.g. a value a game passes to a shared package
{ "TYPE": "CustomVar", "KEY": "dgvoodoo_dir", "DEFAULT": "" }

// a secret rotated each launch
{ "TYPE": "CustomVar", "KEY": "CDKEY",
  "UI": { "LABEL": "CD Key", "CONTROL": "secret", "POOL": ["N9UBI4…","1YREOT…","JAXDO1…"] } }
```

> **Design note.** Earlier MPF used a single `VARTYPE` that conflated the value's domain, its UI control, **and** a
> registry encoding (`dword`/`qword`/`bool`). That made one value unable to serve both a config file and a registry key,
> and merged user-options with internal-bindings. The current model separates those: `UI.CONTROL` is *domain only*,
> encoding moves to the **use site** (§8.4), and a binding is simply a `CustomVar` with no `UI`.

## 8.4 Use-site render formats: `%KEY:format%`

A variable's value is stored **raw** (e.g. `"1"`). Encoding is requested by the *consumer*, inline:

```
RegEdit KEYVALUES { "Fullscreen": "%FULLSCREEN:dword%" }   → dword:00000001    (Wine registry)
config  "fullscreen=%FULLSCREEN%"                          → 1                 (raw)
config  "fullscreen=%FULLSCREEN:bool%"                     → true              (human text)
```

So **one value feeds many consumers**, each formatted as it needs. Formats a conforming renderer MUST provide:

| Format | Effect |
|--------|--------|
| `dword` | decimal → `dword:XXXXXXXX` (8 hex, 32-bit) — Wine registry |
| `qword` | decimal → `hex(b):XX,…,XX` (8 bytes, little-endian) — Wine registry |
| `bool` | truthy (`1`/`true`/`yes`) → `true`, else `false` |
| `winpath` | `/` → `\` (wine/guest paths) |
| `upper` / `lower` | ASCII case fold |
| (empty / unknown) | the value unchanged |

(VidyaGod: `VarSubst::RenderValue`. Implementations MAY add formats; these are the portable set.)

## 8.5 Override via the dependency chain

Because `CustomVar`s live in `LAYERS`, they resolve in **closure order** (parents before children; the launchable last —
chapter 12). When the same `KEY` is declared more than once, the declaration **later in the chain wins**. That is the
override mechanism — and it falls out of the node graph, not a special field:

- A shared package (e.g. a dgVoodoo wrapper) declares `{ "KEY": "dgvoodoo_dir", "DEFAULT": "" }`.
- A game that depends on it declares `{ "KEY": "dgvoodoo_dir", "DEFAULT": "bin" }`. Because the game depends on the
  shared package, the game's node is *later* in the closure → its value wins. The shared package is thus parameterized
  by the game with no collision-prone magic.

This is the **intended, documented** behavior (older material called the pattern "FORCEVARS"). Duplicate keys across the
dependency chain are *the feature*, so a validator does not flag them — but it does flag genuinely undefined references
and dead options (§8.7).

## 8.6 Resolution priority & timing

Each `CustomVar` resolves to a **raw** value by priority, highest first:

1. **Explicit override** — a per-launch value (CLI `--var KEY=VALUE`, or the launch UI control).
2. **Saved user setting** — the value the user last chose (persisted per package).
3. **`DEFAULT`** — the declared default (token-expanded if computed).

A `secret` with a `POOL` ignores 2–3 and picks from the pool each launch (1 still wins). The resolved value is **not**
encoded — consumers render it at the use site (§8.4).

**Absolute scope.** All `CustomVar`s form **one global namespace**. Each key resolves to its hierarchy-final value
(§8.5), and that value is visible to **every** reference — including inside another var's `DEFAULT` — *regardless of
declaration order*. So a computed `DEFAULT` may reference a variable declared later, or one that a more-specific node
overrides; it sees the final value either way. (VidyaGod resolves this as a fixpoint: collect each key's winning source,
then substitute all sources against the built-ins + every var until stable.) A genuine **reference cycle** (`A`→`B`→`A`)
is not an error — resolution terminates, leaving the unresolvable `%token%` literal (and a validator MAY warn).

An implementation MUST resolve `CustomVar`s **before** expanding the layers/edits/args that reference them, so a `%KEY%`
in a layer path or edit value sees its value. (VidyaGod: `ResolveCustomVariables` runs before `BuildSubComponentsArray`.)

## 8.7 Validation (lint)

Because variables and their references are explicit, a validator (chapter 15) MUST catch two classes of error the old
silent-substitution model hid:

- **Undefined reference** — a `%KEY%` that is neither a built-in token nor declared by any `CustomVar` in the graph is
  almost certainly a typo (it would survive as a literal `%KEY%`) → **error**.
- **Orphan option** — a `CustomVar` with a `UI` facet whose `%KEY%` is referenced nowhere is a dead knob → **warning**.

(VidyaGod: `ManifestModel::ValidateNodeGraph`.)

Next: [The EXEC block](09-exec.md).
