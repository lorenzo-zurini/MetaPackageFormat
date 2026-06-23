# 06 · Edit layers

Edit layers *mutate* the runtime rather than contributing whole files: a registry key, a DLL load policy, a line in a
config file. There are three: `RegEdit`, `DllOverride`, `FileEdit`. They share one cross-cutting concept — the
**OVERRIDE pass** — defined first.

## 6.1 The two-pass model: base vs. OVERRIDE

Every edit is applied in one of two passes, selected by the layer's boolean `OVERRIDE` field (default `false`):

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

Edit layers are applied **after variable substitution** — every field below may contain `%TOKEN%`s, expanded before the
edit is performed (chapter 8).

## 6.2 `FileEdit` — patch a text file

Edits a single file in three modes. Common fields:

| Field | Meaning |
|-------|---------|
| `TYPE` | `"FileEdit"` |
| `MODE` | `"ConfigWrite"` \| `"Overwrite"` \| `"AppendLine"` |
| `FILE` | path of the target file, relative to the pass's base (the default-data root for base edits; the runtime root for OVERRIDE edits) |
| `VALUE` | the value to write (see per-mode meaning) |
| `KEY` | (ConfigWrite only) the line-prefix to match |
| `OVERRIDE` | base vs. override pass (§6.1) |

### `MODE: "ConfigWrite"` — line-prefix replace

Rewrites the file in place: **any line whose text starts with `KEY` is replaced by `KEY + VALUE`**; all other lines are
preserved verbatim. This patches INI/config files where the key is a line prefix.

```json
{ "TYPE": "FileEdit", "MODE": "ConfigWrite",
  "FILE": "drive_c/Game/config.ini", "KEY": "Resolution=", "VALUE": "%ScreenWidth%x%ScreenHeight%" }
```

A line `Resolution=800x600` becomes `Resolution=1920x1080`. If no line matches, the file is rewritten unchanged (the key
is *not* appended — use `AppendLine` for that).

### `MODE: "Overwrite"` — replace the whole file

Writes `VALUE` as the file's *entire* content, creating parent directories as needed. For files whose whole content *is*
the value (e.g. a CD-key file).

```json
{ "TYPE": "FileEdit", "MODE": "Overwrite",
  "FILE": "drive_c/Game/cdkey.txt", "VALUE": "%CDKEY%" }
```

### `MODE: "AppendLine"` — idempotent append (load-order primitive)

Appends `VALUE` as a line, **idempotently**: if a line equal to `VALUE` already exists (CRLF-tolerant), nothing is
written. Creates the file if absent and inserts a separating newline if the file didn't end in one. This is the building
block for *accumulating* config — e.g. registering each enabled mod in a load-order file, where every mod node
contributes one `AppendLine` and the resulting file lists exactly the enabled mods, in closure order.

```json
{ "TYPE": "FileEdit", "MODE": "AppendLine",
  "FILE": "drive_c/Morrowind/Morrowind.ini", "VALUE": "GameFile1=Tribunal.esm" }
```

> **Why this is powerful (non-normative).** Combine `AppendLine` with `OPTIONAL` content nodes and closure ordering and
> you get *automatic, ordered mod load lists* with no bespoke "mod manager" construct: enable a mod → its node enters the
> closure → its `AppendLine` runs in order → the load-order file is correct. Disable it → its line never appears.

(VidyaGod: `FileEdits::ConfigWrite` / `FileOverwrite` / `AppendLine` in `fileedits.cpp`.)

## 6.3 `RegEdit` — write Windows-registry values (Wine-family only)

Writes keys/values into the runner's Wine/Proton registry hives. **Only meaningful for a runner that generates a prefix**
(chapter 9); for native/emulator runners there are no hives and `RegEdit` is a no-op.

| Field | Meaning |
|-------|---------|
| `TYPE` | `"RegEdit"` |
| `REGPATH` | the key path in *natural* form, e.g. `HKCU\\Software\\Vendor\\App`. **Do not** include `Wow6432Node` — the runtime re-inserts WoW64 redirection itself based on `ARCHITECTURE`. |
| `ARCHITECTURE` | `"32"` or `"64"` — the bitness of the app reading this key. For `"32"` on a 64-bit prefix, the runtime re-inserts `Wow6432Node` so the key lands where a 32-bit app looks. |
| `KEYVALUES` | object of `name → value`. Values use Wine `.reg` encodings: a bare string (`"en"`), `"dword:0000000a"`, `"hex(b):..,.."`, etc. Use a `CustomVar` of the matching `VARTYPE` to *generate* these encodings from human input (chapter 8). |
| `OVERRIDE` | base vs. override pass (§6.1). |

A `RegEdit` with no `KEYVALUES` creates the key itself (key-only). 

```json
{ "TYPE": "RegEdit",
  "REGPATH": "HKCU\\Software\\nipkow\\SH2EEsetup",
  "ARCHITECTURE": "64",
  "KEYVALUES": { "lang": "en", "fullscreen": "dword:00000001" } }
```

Base `RegEdit`s are baked into the default-data hives (copied from the pristine prefix, never mutating it); OVERRIDE
`RegEdit`s are written into the mounted runtime hives post-mount. (VidyaGod: `RegistryWrapper::ApplyRegEdits`;
`BuildDefaultData`/`ApplyOverrideRegEdits` in `registrylayer.cpp`.)

## 6.4 `DllOverride` — Wine DLL load policy (Wine-family only)

Declares a Wine DLL override string. All `DllOverride` values in the closure are collected and joined into the
`WINEDLLOVERRIDES` environment variable for the launch. Only meaningful for prefix-generating runners.

| Field | Meaning |
|-------|---------|
| `TYPE` | `"DllOverride"` |
| `DLLOVERRIDE` | a Wine override token, e.g. `"d3d8=n,b"` (native then builtin), `"dinput8=n"`. |

```json
{ "TYPE": "DllOverride", "DLLOVERRIDE": "d3d8=n,b" }
{ "TYPE": "DllOverride", "DLLOVERRIDE": "dinput8=n,b" }
```

(VidyaGod: `ProcessDLLOverrides` collects them; `Execute` joins into `WINEDLLOVERRIDES`.)

## 6.5 Applicability summary

| Layer | Native / emulator runner | Wine-family runner |
|-------|--------------------------|--------------------|
| `FileEdit` (all modes) | ✅ applied (files exist for any runtime) | ✅ applied |
| `RegEdit` | ⛔ no-op (no hives) | ✅ applied |
| `DllOverride` | ⛔ no-op (no Wine) | ✅ applied |

An author MAY include Wine-only edits in a cross-platform package; they simply do nothing under a non-Wine chain. A
validator MAY note (not error) Wine-only edits on a launchable whose only resolvable runner is non-Wine.

Next: [Persistence layers](07-persistence.md).
