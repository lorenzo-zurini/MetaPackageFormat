# 06 · Edit nodes

Edit nodes *mutate* the runtime rather than contributing whole files: a registry key, a DLL load policy, a line in a
config file, a byte in an executable. There are four types: `RegEdit`, `FileEdit`, `BinaryPatch`, `DllOverride`. They
share one cross-cutting concept — the **OVERRIDE pass** — defined first. Any node MAY carry a **`WHEN`** condition
([chapter 8 §8.8](08-variables.md#88-when--conditional-layers)) — a false `WHEN` makes the whole node inert.

**Plural payloads batch within a node.** One edit type means one node, and a node of that type holds *all* of that
kind of edit that belongs together: a `RegEdit` node holds a whole hive tree, a `BinaryPatch` node holds every patch to
one executable. Seventy-five hand-written patch layers are not seventy-five nodes — they are one node whose `EDITS`
array has seventy-five entries. Granularity is a graph question, not a payload question: split a node when something
needs to depend on *part* of it.

## 6.1 The two-pass model: base vs. OVERRIDE

Every edit is applied in one of two passes, selected by a boolean `OVERRIDE` field (default `false`):

- **Base edit (`OVERRIDE: false`)** — written into a dedicated **default-data layer** that sits in the overlay *between*
  the package content and the user's writable layer. Effect: the edit overrides the package's *own* shipped content, but
  the **user's persisted writes shadow it**. This is "factory defaults under live user data": the game starts with the
  edit applied, but if the user (or the game) later changes that file/key, their version wins on subsequent launches.
- **Override edit (`OVERRIDE: true`)** — applied *after* the overlay is mounted, writing straight through to the user's
  writable layer. Effect: it **wins unconditionally** — over the package content *and* over any persisted user state.
  Re-applied every launch. Use it for values that must be forced each run (a fixed compatibility tweak, a kill-switch).

The overlay priority, lowest → highest:

```
runner prefix  <  package content layers  <  DEFAULT-DATA (base edits)  <  user writable layer  <  OVERRIDE edits
```

(VidyaGod: base edits build the `DEFAULTDATA` layer in `BuildDefaultData`; OVERRIDE edits run post-mount via
`ProcessFileEdits(override=true)` / `ApplyOverrideRegEdits`. Full stack in [chapter 13](13-runtime-model.md).)

Edit nodes are applied **after variable substitution** — every field below may contain `%TOKEN%`s, expanded before the
edit is performed (chapter 8).

> **`FILE` must be RELATIVE.** It is resolved against the pass's base, and a `%RuntimePath%/`-prefixed value is an
> *absolute* path that silently escapes that base — the edit then lands somewhere nothing reads, with no diagnostic.

## 6.2 `FileEdit` — patch a text file

The node names the target file and the pass; `EDITS` holds the edits to make to it.

| Field | Where | Meaning |
|-------|-------|---------|
| `FILE` | node | path of the target file, relative to the pass's base (the default-data root for base edits; the runtime root for OVERRIDE edits) |
| `OVERRIDE` | node | base vs. override pass (§6.1) |
| `EDITS` | node | array of edit objects, applied in order |
| `MODE` | entry | `"ConfigWrite"` \| `"Overwrite"` \| `"AppendLine"` |
| `KEY` | entry | (ConfigWrite only) the line-prefix to match |
| `VALUE` | entry | the value to write (see per-mode meaning) |

```json
{ "LABEL": "morrowind_ini", "TYPE": "FileEdit", "PARENTS": ["morrowind_content"],
  "FILE": "drive_c/Morrowind/Morrowind.ini", "OVERRIDE": true,
  "EDITS": [ { "MODE": "ConfigWrite", "KEY": "Resolution=", "VALUE": "%ScreenWidth%x%ScreenHeight%" },
             { "MODE": "AppendLine",  "VALUE": "GameFile1=Tribunal.esm" } ] }
```

> **A `ConfigWrite` on content this package itself ships must be `OVERRIDE: true`.** The base pass runs *before* the
> content is mounted, against `DEFAULTDATA` — so it has no file to rewrite and does nothing.

### `MODE: "ConfigWrite"` — line-prefix replace

Rewrites the file in place: **any line whose text starts with `KEY` is replaced by `KEY + VALUE`**; all other lines are
preserved verbatim. This patches INI/config files where the key is a line prefix.

```json
{ "MODE": "ConfigWrite", "KEY": "Resolution=", "VALUE": "%ScreenWidth%x%ScreenHeight%" }
```

A line `Resolution=800x600` becomes `Resolution=1920x1080`. If no line matches, the file is rewritten unchanged (the key
is *not* appended — use `AppendLine` for that).

### `MODE: "Overwrite"` — replace the whole file

Writes `VALUE` as the file's *entire* content, creating parent directories as needed. For files whose whole content *is*
the value (e.g. a CD-key file).

```json
{ "MODE": "Overwrite", "VALUE": "%CDKEY%" }
```

### `MODE: "AppendLine"` — idempotent append (load-order primitive)

Appends `VALUE` as a line, **idempotently**: if a line equal to `VALUE` already exists (CRLF-tolerant), nothing is
written. Creates the file if absent and inserts a separating newline if the file didn't end in one. This is the building
block for *accumulating* config — e.g. registering each enabled mod in a load-order file, where every mod node
contributes one `AppendLine` and the resulting file lists exactly the enabled mods, in closure order.

```json
{ "MODE": "AppendLine", "VALUE": "GameFile1=Tribunal.esm" }
```

> **Why this is powerful (non-normative).** Combine `AppendLine` with `TOGGLE: "off"` content nodes and closure ordering and
> you get *automatic, ordered mod load lists* with no bespoke "mod manager" construct: enable a mod → its node enters the
> closure → its `AppendLine` runs in order → the load-order file is correct. Disable it → its line never appears.

(VidyaGod: `FileEdits::ConfigWrite` / `FileOverwrite` / `AppendLine` in `fileedits.cpp`.)

## 6.3 `RegEdit` — write Windows-registry values (Wine-family only)

Writes keys/values into the runner's Wine/Proton registry hives. **Only meaningful for a runner that generates a prefix**
(chapter 9); for native/emulator runners there are no hives and `RegEdit` is a no-op.

The registry **is a tree**, so the payload is a tree. A `RegEdit` node holds an `EDITS` array; each entry is one group
of keys sharing an architecture and a pass, written as nested JSON hanging off a hive name:

| Field | Where | Meaning |
|-------|-------|---------|
| `EDITS` | node | array of entries, each a hive tree plus its selectors |
| `ARCHITECTURE` | entry | **array** of `"32"` / `"64"` — the bitness(es) of the app reading these keys. The same tree is written once per listed architecture. For `"32"` on a 64-bit prefix the runtime re-inserts `Wow6432Node` so the key lands where a 32-bit app looks. An empty/absent array means "no redirection decision", written once. |
| `OVERRIDE` | entry | base vs. override pass (§6.1) |
| `WHEN` | entry | condition for this entry alone (the node's own `WHEN` still gates everything) |
| *any other key* | entry | a **hive name** (`HKLM`, `HKCU`, …) whose value is the nested key tree |

Inside a tree, an **object** value is a subkey and a **non-object** value is a registry value. A key may hold both at
once. An **empty object** means "create this key, no values".

Values use Wine `.reg` encodings: a bare string (`"en"`), `"dword:0000000a"`, `"hex(b):..,.."`. The name `"@"` is the
key's *default* value. Drive them from a `CustomVar` rendered at the use site — `"%FULLSCREEN:dword%"` /
`"%MASK:qword%"` (chapter 8 §8.4).

**Do not write `Wow6432Node` yourself** — the runtime re-inserts WoW64 redirection based on `ARCHITECTURE`.

```json
{ "LABEL": "sh2ee_registry", "TYPE": "RegEdit", "PARENTS": ["sh2ee_content"],
  "EDITS": [
    { "ARCHITECTURE": ["64"],
      "HKCU": { "Software": { "nipkow": { "SH2EEsetup": {
          "lang": "en",
          "fullscreen": "dword:00000001",
          "Paths": { "install": "C:\\10972" } } } } } },
    { "ARCHITECTURE": ["32", "64"], "OVERRIDE": true,
      "HKLM": { "Software": { "Vendor": { "App": { "Forced": "1" } } } } }
  ] }
```

> **A value name and a subkey name cannot collide.** `{"Thing": "a-value", "Thing": {…}}` is not expressible, and a
> tool that flattens the tree for editing MUST refuse to write one over the other rather than silently dropping
> whichever it processes second.

Base `RegEdit`s are baked into the default-data hives (copied from the pristine prefix, never mutating it); OVERRIDE
`RegEdit`s are written into the mounted runtime hives post-mount. (VidyaGod: `RegistryWrapper::ApplyRegEdits`;
`BuildDefaultData`/`ApplyOverrideRegEdits` in `registrylayer.cpp`.)

## 6.4 `DllOverride` — Wine DLL load policy (Wine-family only)

Declares Wine DLL overrides as a **map of `dll → resolution order`**. All overrides in the closure are collected and
joined into the `WINEDLLOVERRIDES` environment variable for the launch. Only meaningful for prefix-generating runners.

| Field | Meaning |
|-------|---------|
| `OVERRIDES` | object of `dll name → order string`: `"n,b"` (native then builtin), `"b,n"`, `"n"`, `"b"`, `"d"` (disabled) |

```json
{ "LABEL": "asiloader_overrides", "TYPE": "DllOverride",
  "OVERRIDES": { "d3d8": "n,b", "dinput8": "n,b" } }
```

> **An EMPTY order is meaningful.** `"winegstreamer": ""` means *disabled* in Wine — it is not an omission to be
> defaulted to `"n,b"`. Treating an empty string as "unset" silently inverts what the package asked for.

A value MAY also carry a multi-DLL spec (`"n,b;dinput8=n,b"`) where a package needs one; an editor offering the common
orders as a menu MUST still let such a value through unchanged.

(VidyaGod: `ProcessDLLOverrides` collects them; `Execute` joins into `WINEDLLOVERRIDES`.)

## 6.5 `BinaryPatch` — patch an executable

Mutates a binary in place, declaratively, over the **pristine** file. The shipped/content-addressed executable is
never modified; the patch is applied to a copy-on-write copy in the runtime prefix at launch. This replaces the
"ship a whole patched copy of the exe" convention (a No-CD crack, a crash fix): the pristine binary is stored once
and the modification travels as a few hundred bytes of JSON. Because the base is content-addressed (a frozen hash),
fixed offsets can never drift, and an `EXPECT` guard makes a wrong or already-patched target fail loud.

The node names the binary; `EDITS` holds every patch to it, applied in order. A seventy-five-patch enhancement is one
node, not seventy-five.

| Field | Where | Meaning |
|-------|-------|---------|
| `FILE` | node | target binary, relative to the mount root (same convention as a `DeclareExec` `PATH`) |
| `EDITS` | node | array of patch objects |
| `MODE` | entry | `"Replace"` \| `"Poke"` \| `"Cave"` |
| `APPLY` | entry | `"prefix"` (default — on-disk into the writelayer) \| `"memory"` (patch the live process; see below) |
| `ANCHOR` | entry | a hex signature locating the site, `??` = wildcard byte (`"8b ?? 24 08"`); **must match exactly one place** — *or* give `OFFSET` |
| `OFFSET` | entry | a fixed address: a **VA** if `>=` the PE image base, else a raw **file offset**. Hex (`"0x44be08"`) or decimal. |
| `EXPECT` | entry | hex of the original bytes at the site. A mismatch is an error (wrong/foreign binary); bytes already equal to the patched result are skipped (idempotent). Required for `Replace`/`Cave`; recommended for `Poke`. |
| `COMMENT` | entry | what this patch does. Not optional in practice: seventy-five offsets with no prose is a binary nobody can maintain. |

Every field is `%TOKEN%`-substituted (chapter 8), so offsets, bytes and values can come from `CustomVar`s.

### `MODE: "Replace"` — overwrite bytes

Writes `REPLACE` (hex) at the site. `len(REPLACE) ≤ len(EXPECT)`; a shorter `REPLACE` is padded with `0x90` (NOP)
to the `EXPECT` length. The No-CD primitive.

```json
{ "LABEL": "game_nocd", "TYPE": "BinaryPatch", "PARENTS": ["game_content"],
  "FILE": "%PrefixRoot%/drive_c/%PackageUID%/GAME.EXE",
  "EDITS": [ { "MODE": "Replace", "OFFSET": "0x44a45c", "EXPECT": "01", "REPLACE": "00",
               "COMMENT": "skip the CD presence check" } ] }
```

### `MODE: "Poke"` — write a scalar

Writes `VALUE` (hex) at the site — usually a `CustomVar` rendered with a numeric format so a setting is baked
straight into the binary: `"VALUE": "%ResWidth:u16le%"` (see chapter 8 §8.4 for `u8`/`u16le`/`u16be`/`u32le`/`u32be`).

### `MODE: "Cave"` — jmp trampoline

Displaces `EXPECT` (≥5 bytes, room for a `jmp rel32`) into a **code cave** and runs custom code around it. The
engine writes the cave as `PAYLOAD` (your code) + the displaced original bytes + a `jmp` back to just past the
site, then overwrites the site with `jmp <cave>` (NOP-padded to the `EXPECT` length).

| Field | Meaning |
|-------|---------|
| `PAYLOAD` | hex of the cave body to run *before* the displaced bytes |
| `CAVE` | `"auto"` (default — the first zero run of sufficient length in an executable section) or a fixed VA |

```json
{ "MODE": "Cave", "OFFSET": "0x44be08", "EXPECT": "c705d07d4d0001000000",
  "PAYLOAD": "5031c0a390d09000...58", "CAVE": "0x4cf360" }
```

### `APPLY: "memory"` (opt-in)

`"prefix"` (default) patches the file on disk (in the writelayer) and works for **every** runner and launch path —
it is what the loader reads. Use `"memory"` only when the on-disk file must stay pristine at load (anti-tamper that
checksums the file): the patch is applied to the live process image after `CREATE_SUSPENDED` and before the first
instruction runs. This is **single-player only** (it requires a direct process handle, which DirectPlay-spawned and
several-hops-into-wine launches do not provide) and is a VidyaGod-specific capability, not portable.

(VidyaGod: `BinaryPatch::ApplyOne` / `ProcessBinaryPatches` in `binarypatch.cpp`; `APPLY:"memory"` via `vglobby`.)

## 6.6 Applicability summary

| `TYPE` | Native / emulator runner | Wine-family runner |
|-------|--------------------------|--------------------|
| `FileEdit` (all modes) | ✅ applied (files exist for any runtime) | ✅ applied |
| `BinaryPatch` (`APPLY:"prefix"`) | ✅ applied (patches the on-disk exe) | ✅ applied |
| `BinaryPatch` (`APPLY:"memory"`) | ✅ single-player direct-launch only | ✅ single-player direct-launch only |
| `RegEdit` | ⛔ no-op (no hives) | ✅ applied |
| `DllOverride` | ⛔ no-op (no Wine) | ✅ applied |

An author MAY include Wine-only edits in a cross-platform package; they simply do nothing under a non-Wine chain. A
validator MAY note (not error) Wine-only edit nodes on a launchable whose only resolvable runner is non-Wine.

Next: [Persistence](07-persistence.md).
