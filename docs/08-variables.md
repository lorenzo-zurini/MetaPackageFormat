# 08 · Variables & `WHEN`

Almost every string in a node may contain `%TOKEN%` placeholders, expanded at launch: content and edit targets, edit
values, registry values, environment values, entry fields. This chapter defines the substitution engine, the
built-in tokens, the `VARS` layer that declares a package's own variables (including the options shown in the launch
window), the use-site render formats and the `WHEN` conditions.

## 8.1 Substitution

`%NAME%` is a placeholder; `%NAME:format%` renders the value for one consumer (§8.4).

```
function substitute(text, vars):
    out = ""; i = 0
    while i < len(text):
        start = indexOf(text, '%', i)
        if start == -1: out += text[i:]; break
        end = indexOf(text, '%', start + 1)
        if end == -1: out += text[i:]; break              # an unmatched '%': keep the rest verbatim
        out += text[i:start]
        name, fmt = split_first(text[start+1 : end], ':')
        if name in vars: out += (fmt == "" ? vars[name] : render(vars[name], fmt))
        else:            out += text[start : end+1]       # unknown: the whole token stays, visible
        i = end + 1
    return out
```

- Tokens are paired `%`s, scanned left to right, never overlapping. A name MUST NOT contain `:`.
- An **unknown** name leaves the whole token in place, never an empty string: a missing variable stays visible, and a
  validator reports it (§8.7).
- One substitution is a single pass; it does not re-expand what it inserted. (Variables are resolved against each
  other first, §8.5.)

## 8.2 Built-in tokens

| Token | Value |
|-------|-------|
| `%UID%` | the launched tile's `UID` |
| `%PackageUID%` | the family's root `UID` (the install folder, the owner of the instance) |
| `%PackageName%`, `%GameName%` | the launched tile's title |
| `%ScreenWidth%`, `%ScreenHeight%` | the primary display's size (`"0"` when unknown) |
| `%RuntimePath%` | the runtime mount, as a host path |
| `%RunnerMount%` | where the boundary runner's build is mounted, as a host path (inside an inner runner's own fields: that runner's build, chapter 11 §11.6) |
| `%TempPath%` | the session's scratch directory |
| `%PrefixRoot%` | where the prefix sits in the runtime (`pfx` for Proton, `""` for Wine at the root) |
| `%ContentPath%` | the entry's `EXE`, relative to the content root, folders joined with `/` (a Windows runner spells it `%ContentPath:winpath%`) |
| `%Content%` | the entry's `EXE`, as a host path |
| anchors | every `GUEST_ROOTS` anchor of the boundary runner (`%GameDir%`, `%Documents%`, …), as its guest path (chapter 10 §10.4) |
| `%HostUserProfile%`, `%HostAppData%`, `%HostLocalAppData%`, `%HostDocuments%`, `%HostProgramData%`, `%HostPublic%`, `%HostWindows%`, `%HostSysDir32%`, `%HostSysDir64%`, `%HostProgramFiles%`, `%HostProgramFiles32%` | on a Windows host, that machine's own folder (`%HostSysDir32%` is the 32-bit system folder, `SysWOW64` on a 64-bit Windows); unset elsewhere. A runner whose guest is the host's own Windows spells its anchors from them (chapter 10 §10.4) |

A runner that assembles a Wine prefix also gets its build's layout: `%DefaultPfxDir%`, `%WineFontsDir%`,
`%WineLibDir%`, `%WineSys32Dir%`, `%WineSysWow64Dir%`. Inside a runner's guest-path template, `%REL%` is the path
being translated. An implementation MAY add tokens for its own features (VidyaGod adds `%VIDYAGOD_…%` tokens for its
LAN overlay). A package's variables are added last and MAY shadow a built-in.

## 8.3 `VARS`: declared variables

```jsonc
{ "VARS": {
    "RENDERER": { "DEFAULT": "dxvk",
                  "UI": { "LABEL": "Renderer", "CONTROL": "enum", "SECTION": "Graphics",
                          "CHOICES": [ { "LABEL": "DXVK (Vulkan)", "VALUE": "dxvk" },
                                       { "LABEL": "WineD3D",       "VALUE": "wined3d" } ] } },
    "VRAM":     { "DEFAULT": "1024", "WHEN": "%RENDERER% == dxvk",
                  "UI": { "LABEL": "VRAM (MB)", "CONTROL": "int", "SECTION": "Graphics", "MIN": 64, "MAX": 8192 } },
    "dgv_dir":  { "DEFAULT": "bin" },
    "CDKEY":    { "UI": { "LABEL": "CD key", "CONTROL": "secret", "POOL": ["N9UBI4…", "1YREOT…"] } } } }
```

A declaration:

| Field | Meaning |
|-------|---------|
| `DEFAULT` | the value when the instance holds none. May contain `%tokens%` (a computed value). |
| `WHEN` | gates the **value**: when the condition does not hold, the variable is `""` and its control is hidden. |
| `EVAL` | `true`: once its tokens resolve, the value is an integer expression and the variable is its decimal result. Numbers in decimal or `0x` hex; `( )`, unary `- ~ !`, `* / %`, `+ -`, `<< >>`, `&`, `^`, `\|` with C precedence. Until every operand resolves the text stays; an expression that never parses keeps its text. For values built from independent options, such as a registry bit field: `"%dark_red%*2 \| %dark_purple%*32"`, used as `%minimap:dword%`. |
| `UI` | its presence makes the variable an **option** in the launch window; without it the variable is an internal binding. |
| `MAP` | makes the declaration a **table**: `{value: mapped value}`, any exact string to a string. A value is read through it as `%KEY:map:TABLE%` (§8.4). Entries may contain `%tokens%`, resolved like a `DEFAULT`; the table's `WHEN` switches it as a whole. A table has no `UI`. |
| `ELSE` | a table only: what a value missing from `MAP` becomes. Without it, a missing value passes through unchanged. |
| `COMMENT` | why it exists. |

The `UI` facet:

| Field | Meaning |
|-------|---------|
| `LABEL` | the control's label (default: the key) |
| `CONTROL` | the value's domain and widget: `bool`, `int`, `float`, `text`, `enum`, `secret`, `key`. No encoding here; encoding is the use site's (§8.4). A `key` option holds a key name (below), `""` when unbound; the launch window binds it by a key, mouse or gamepad press. |
| `SECTION` | where the option sits in the launch window's tree, a path whose `/` nests (`"UserPatch/Interface"`). Sections appear with their first option, collapsed. A runner's options appear under its own section. |
| `CHOICES` | `enum` only: `[{"LABEL": …, "VALUE": …}]` |
| `MIN`, `MAX` | `int` and `float` bounds |
| `PATTERN` | `text` and `secret`: a regular expression the input must match |
| `POOL` | `secret` only: values one of which is drawn for a new instance (a CD key). The draw is kept in the instance, so it is the same every launch until the user changes it. A `secret` without `POOL` is a masked text field. |
| `WHEN` | a condition, used when the declaration has no `WHEN` of its own: it gates the value and the control alike. |

**Key names.** A `key` option is bound by pressing the key (or button); it stores the key's name, the same on every platform. What the
game wants for it is the package's data, mapped like any other option value (a `WHEN`-gated layer or edit per key, a
variable whose `DEFAULT` builds on it); a `MAP` table spells each name the game's way, `%jump:map:unity_keys%`.
Names follow the key caps of the user's layout; modifiers have sides:

`a` … `z`, `0` … `9`, `f1` … `f12`, `space`, `enter`, `escape`, `tab`, `backspace`, `insert`, `delete`, `home`, `end`,
`page up`, `page down`, `up`, `down`, `left`, `right`, `left shift`, `right shift`, `left ctrl`, `right ctrl`, `left alt`,
`right alt`, `left meta`, `right meta`, `menu`, `caps lock`, `num lock`, `scroll lock`, `print screen`, `pause`, `minus`,
`equals`, `left bracket`, `right bracket`, `backslash`, `semicolon`, `quote`, `backquote`, `comma`, `period`, `slash`,
`keypad 0` … `keypad 9`, `keypad period`, `keypad divide`, `keypad multiply`, `keypad minus`, `keypad plus`,
`keypad enter`, `mouse left`, `mouse right`, `mouse middle`, `mouse 4`, `mouse 5`, `mouse wheel up`, `mouse wheel down`
(a turn of the wheel away from the user is up); and a game controller's, named by
position as on an Xbox pad: `pad a`, `pad b`, `pad x`, `pad y`, `pad back`, `pad guide`, `pad start`, `pad left stick`,
`pad right stick`, `pad left shoulder`, `pad right shoulder`, `pad left trigger`, `pad right trigger`,
`pad left stick up` / `down` / `left` / `right`, `pad right stick up` / `down` / `left` / `right` (a trigger or stick binds
when pushed past half; one already pushed when the capture starts must return to the centre first), `pad dpad up`,
`pad dpad down`, `pad dpad left`, `pad dpad right`, `pad misc`, `pad paddle 1` … `pad paddle 4`, `pad touchpad`.

A key option's `DEFAULT` is its binding until the user rebinds it; the launch window names the default beside a binding
that differs from it and offers to put it back. A primary and a secondary binding are two `key` options (`jump`,
`jump_alt`). The launch window marks two options of one `SECTION` bound to the same key.

**Tables.** One value often has to become a different spelling in each place it is written: a key name, a
language, a preset. A table says it once, and any variable reads through it where it is used:

```jsonc
{ "VARS": {
    "lang":     { "DEFAULT": "en", "UI": { "LABEL": "Language", "CONTROL": "enum", "CHOICES": ["en", "fr"] } },
    "lang_dir": { "MAP": { "en": "English", "fr": "French" } },
    "lang_id":  { "MAP": { "en": "1033",    "fr": "1036" } },
    "lang_pak": { "MAP": { "en": "%GameDir%/lang/eng.pak" }, "ELSE": "%GameDir%/lang/int.pak" },
    "lcid":     { "DEFAULT": "%lang:map:lang_id%" } } }        // a mapped value as a variable of its own
… "Folder": "%lang:map:lang_dir%"    "LCID": "%lcid:dword%"    "Pak": "%lang:map:lang_pak%"
```

One value in, one value out: a value maps through one table at a time, and several outputs are several tables. A table
folds, gates and travels like any declaration: a later node may replace it, and a library node may carry one that
many games contain (Unity's key names).

**Declarations fold per key.** A later declaration of the same key replaces the earlier one whole (its `DEFAULT`,
`UI` and `WHEN` together). That is how a game parameterises a library it contains: the library declares
`"dgv_dir": {"DEFAULT": ""}`, the game declares `"dgv_dir": {"DEFAULT": "bin"}` above it, and the library's layers
read `%dgv_dir%`. A `TAKE` decides which of a contained node's variables arrive at all (chapter 05 §5.2).

## 8.4 Render formats: `%KEY:format%`

A variable's value is stored raw (`"1"`). The consumer asks for an encoding inline:

```
"Fullscreen": "%FULLSCREEN:dword%"          → dword:00000001       (a registry value)
"fullscreen=%FULLSCREEN%"                  → 1                    (raw)
"fullscreen": "%FULLSCREEN:bool%"          → true                 (JSON, text)
```

| Format | Effect |
|--------|--------|
| `dword` | decimal → `dword:XXXXXXXX` |
| `qword` | decimal → `hex(b):XX,…,XX` (8 bytes, little-endian) |
| `bool` | `1` / `true` / `yes` → `true`, else `false` |
| `winpath` | `/` → `\` |
| `upper`, `lower` | ASCII case |
| `u8`, `u16le`, `u16be`, `u32le`, `u32be` | decimal → hex bytes of that width and byte order (`10` → `u32le` → `0a000000`): a binary op's `VALUE`, or bytes inside a `REPLACE` or `PAYLOAD` |
| `ascii`, `asciiz` | the value's bytes as hex, `asciiz` with a NUL terminator: a C string written into a binary |
| `map:TABLE` | the value's entry in the table `TABLE` (§8.3), else the table's `ELSE`, else the value unchanged. A table whose `WHEN` is off passes every value through. The one format that reads another declaration. |
| empty or unknown | the value unchanged |

An implementation MAY add formats; these are the portable set.

## 8.5 Resolution

Each variable's raw value comes from, highest first:

1. a per-launch override (the CLI's `--var KEY=VALUE`, the launch window's control);
2. the instance's saved value (including a drawn `POOL` value);
3. the folded declaration's `DEFAULT`.

All variables form **one namespace**, and every reference sees each key's final value regardless of where it was
declared: values are substituted against the built-ins and every other variable until nothing changes (VidyaGod stops
after 16 passes). A declaration's `WHEN` is evaluated inside that loop, so conditions may chain. A reference cycle
(`A` → `B` → `A`) is not an error: resolution stops with the token left literal, and a validator MAY warn.

Variables are resolved **before** the layers that use them are applied, so a `%KEY%` in a target, an edit or an
argument sees its value.

## 8.6 `WHEN`: conditions

`WHEN` holds a boolean condition over variables. It appears in three places:

- **On a layer** (any type): the layer is skipped when the condition does not hold. A `NODE` layer does not expand, a
  content layer does not mount, a fact is not written, an entry does not exist.
- **On a declaration** (or its `UI`): the variable's value is `""` when the condition does not hold (§8.3).
- **On an `EDIT` op**: that op is skipped.

**Layer `WHEN`s are read before expansion** (phase 1 of resolution, chapter 12 §12.1). They see the instance's values,
the per-launch overrides, the built-ins (`%UID%`, `%PackageUID%`, the screen size), and the `DEFAULT`s of `VARS`
layers that are themselves ungated and reached only through ungated `NODE` layers. A layer `WHEN` cannot read a
variable that is declared only inside a gated layer (it would be circular); a validator refuses it. Declaration and
op `WHEN`s are read after the fold, against every variable.

**Grammar.**

```
condition := or
or        := and ( '||' and )*
and       := unary ( '&&' unary )*
unary     := '!' unary | primary
primary   := '(' or ')' | operand ( ('==' | '!=') operand )?
operand   := %KEY%  |  "quoted"  |  bare-word
```

- An operand alone is true when its value is non-empty and not `0`, `false` or `no`.
- `==` and `!=` compare strings. `!` binds tighter than `&&`, which binds tighter than `||`.
- Operands are resolved while evaluating, so a value containing `&&` or `==` is compared as data, never re-parsed.
- An empty condition is true. A malformed condition is **true at launch** (fail-open) and a **validator error**, so a
  typo can never silently disable a layer.

Choices are variables: an option with `enum` or `bool` control, and the layers that implement each choice gated on
it. A package's own optional features are variables; a change that someone else may publish is a graft (chapter 12
§12.5).

```jsonc
{ "LABEL": "Netplay", "LAYERS": [
    { "VARS": { "NETMODE": { "DEFAULT": "host",
                  "UI": { "LABEL": "Network", "CONTROL": "enum",
                          "CHOICES": [ {"LABEL": "Host", "VALUE": "host"}, {"LABEL": "Join", "VALUE": "join"} ] } },
                "JOIN_ADDR": { "DEFAULT": "", "WHEN": "%NETMODE% == join",
                               "UI": { "LABEL": "Host address", "CONTROL": "text" } } } },
    { "REG": { "HKCU": { "Software": { "Game": { "Hosting": "dword:00000001" } } } },
      "ARCH": ["32"], "WHEN": "%NETMODE% == host" } ] }
```

## 8.7 Validation

- **Undefined reference.** A `%KEY%` that is neither built in, nor an anchor some runner declares, nor declared by
  any `VARS` layer is almost certainly a typo: an error.
- **Orphan option.** A declaration with `UI` whose key nothing references is a dead control: a warning.
- **Missing table.** A `%KEY:map:TABLE%` naming no declared `MAP` would pass every value through: an error. A `MAP`
  that is not strings to strings, a table with a `UI`, an `ELSE` on a non-table: errors.
- **Not a key.** A `key` option whose literal `DEFAULT` is not a key name (§8.3): an error. Two `key` options of one
  node and `SECTION` with the same `DEFAULT`: a warning.
- **Malformed condition.** An error (§8.6).
- **Phase violation.** A layer `WHEN` reading a variable declared only in a gated layer: an error.

*(VidyaGod: `VarSubst` for substitution, formats and conditions; `Fold::ResolveVars` for the fixpoint;
`PreLaunchWindow` for the controls.)*

Next: [`EXEC`, tiles & variants](09-exec-tiles-variants.md).
