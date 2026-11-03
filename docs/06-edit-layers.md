# 06 · `EDIT` layers

An `EDIT` layer changes **one file**, in place, as it is at that point of the fold: a line in a config file, a value
in a JSON file, a byte in an executable. The file is never modified at its source; the edit is applied to the
runtime's copy each launch.

```jsonc
{ "EDIT": [ { "MODE": "ConfigWrite", "KEY": "Resolution=", "VALUE": "%ScreenWidth%x%ScreenHeight%" },
            { "MODE": "AppendLine",  "VALUE": "GameFile1=Tribunal.esm" } ],
  "TARGET": "FILES/%GameDir%/Morrowind.ini" }

{ "EDIT": [ { "MODE": "Replace", "OFFSET": "0x44a45c", "EXPECT": "01", "REPLACE": "00",
              "COMMENT": "skip the CD presence check" } ],
  "TARGET": "FILES/%GameDir%/GAME.EXE" }
```

| Field | Meaning |
|-------|---------|
| `EDIT` | a non-empty list of ops, applied in order |
| `TARGET` | the file, as a `FILES` address (required) |

Every op has a `MODE`. Text and binary ops share the one form; a layer may mix them, though in practice a layer edits
either a text file or a binary. Every string field is `%TOKEN%`-substituted before the op runs (chapter 08). An op
MAY carry `COMMENT`, and MAY carry its own `WHEN`, which is evaluated against the fully resolved variables (unlike a
layer's `WHEN`, chapter 08 §8.6).

## 6.1 Where an edit applies

**An edit applies to the value beneath it**: the file as composed by every content layer below it in the fold, after
the runtime is mounted. Edits run in fold order, so an edit in a mod sees the file as the base game and every mod
before it left it.

- **Covered edits.** An edit beneath a later content layer that replaces its file (a `ZIP` providing the file, a
  `DELTA` rebuilding its folder, a `FILE` landing on it) has no effect: a validator MUST refuse it. Edits belong
  above the content they change. "Replaces" is exact: a zip replaces the files it holds, a delta those of the archive
  it reconstructs, a `FILE` its `TARGET/<name>`. A file an edit creates that no later layer contains is not covered.
- **Owned files.** When the user owns the edit's address (chapter 13), the edit is the **package's default**: it
  applies while the user has no saved copy of the file, and the user's copy wins after that. An option chosen in the
  launch window that writes into a user-owned file therefore needs its key taken back with `KEEP false`, or it
  applies on the first launch only (chapter 13 §13.4).

## 6.2 Text ops

Text ops read CRLF and LF alike and write LF. `Delete` is listed with them: it changes the file system, not bytes.

### `ConfigWrite`: replace a line by its prefix

```json
{ "MODE": "ConfigWrite", "KEY": "Resolution=", "VALUE": "%ScreenWidth%x%ScreenHeight%" }
```

Every line that **starts with** `KEY` becomes `KEY + VALUE`; other lines are kept. If no line matches, the file is
unchanged (the key is not appended; use `AppendLine`). The op fails if the file does not exist. It writes the keyed
address `FILES/<file>#<KEY>`.

### `JsonSet`: set one value in a JSON file

```json
{ "MODE": "JsonSet", "KEY": "/video/targetfps", "VALUE": "%fps%" }
```

`KEY` is a JSON Pointer (RFC 6901). `VALUE` is read as JSON text (`1920`, `true`, `"x"`, `{"w":1920}`); a value that
does not parse is written as a string. Missing objects on the way are created, and so is the file (absent or empty
reads as `{}`). Every other key, and key order, is kept; comments are accepted on read and not written back. A file
that is not JSON, or a pointer through a value that is neither object nor array, fails the op and leaves the file
untouched. A bool option is written with the `bool` format: `"%fullscreen:bool%"`. It writes `FILES/<file>#<KEY>`.

### `XmlSet`: set one value in an XML file

```json
{ "MODE": "XmlSet", "KEY": "/Attributes/Attr[@name='g_language']/@value", "VALUE": "%language%" }
```

`KEY` is a path of elements from the root, each matched by its name and, optionally, attribute predicates
(`[@name='value']`, several allowed, either quote). It ends in an attribute (`/@value`), or names an element whose text
is the value. The first element matching each step is the one meant. `VALUE` is written as text (escaped as XML needs).

- **Only the value changes.** Every other byte of the file stays as it was: its declaration, comments, quoting and
  indentation.
- **Missing parts are created.** A missing element is created with its predicates' attributes and placed after its
  parent's last child, indented like that child. A missing attribute is added to its element, and an absent or empty
  file becomes the path's document. A game that leaves a setting out of its file while it is at its default is set
  this way.
- **`!` steps into XML stored inside an attribute (or text).** That value is a document of its own, and the path after
  the `!` is read in it: `…/Attr[@name='keybinds_settings']/@value!/keybinds/superaction[@name='jump']/control[@slot='0']/@input`.
- **Refusals leave the file untouched.** The op fails on a file that is not well-formed XML, a root other than the
  path's, text set on an element that holds elements, or a value holding an entity that only a DTD defines.

It writes `FILES/<file>#<KEY>`.

### `Overwrite`: the whole file

```json
{ "MODE": "Overwrite", "VALUE": "%CDKEY%" }
```

`VALUE` becomes the file's entire content; the file and its folders are created if absent. For files whose whole
content is a value (a CD-key file, a version stamp).

### `AppendLine`: an idempotent append

```json
{ "MODE": "AppendLine", "VALUE": "GameFile1=Tribunal.esm" }
```

Appends `VALUE` as a line unless a line equal to it already exists. Creates the file if absent, and adds a newline
first if the file did not end in one. This is the load-order primitive: each mod appends its own line, so the file
lists exactly the ticked mods, in graft order.

### `Delete`: remove a file or a folder

```json
{ "EDIT": [ { "MODE": "Delete", "COMMENT": "PCGW: skip the intro movie" } ],
  "TARGET": "FILES/%GameDir%/videos/intro.bik", "WHEN": "%skip_intro% == 1" }
```

Removes the `TARGET` from the runtime as composed beneath it: a file, or a folder with everything in it. The content
layers that provided it are untouched; the removal lives in the session's writable layer (a deletion marker over the
files beneath) and is gone after the session like every other write. A link is removed as a link, never followed. The
op takes no other field than `COMMENT` and `WHEN`. Nothing at the `TARGET` is a failure (§6.4): the name is a typo, or
the content beneath changed. This is how a fix that says "delete or rename the file" is expressed, when the file is
named somewhere the package cannot patch (inside a compressed archive the game reads).

**A `Delete` never reaches the user's state.** One whose `TARGET` is an address the user owns, lies inside a kept folder
(taken back or not: a kept folder is served live, so the removal would be stored with the user's files), contains a kept
address, or names a file a kept pattern matches, is refused by the validator and at launch (chapter 13). The launch
judges the place the `TARGET` **resolves** to, not how it is written: `videos/../Saves`, another anchor naming the same
folder or a `%variable%` segment are the same place. A package removes only what is the package's.

## 6.3 Binary ops

Binary ops patch an executable over its **pristine** bytes. The pristine file is stored once and content-addressed,
so fixed offsets cannot drift; an `EXPECT` guard makes a wrong or foreign binary fail loudly. A seventy-five-patch
enhancement is one layer with seventy-five ops.

| Field | Meaning |
|-------|---------|
| `MODE` | `Replace` \| `ReplaceAll` \| `Poke` \| `Or` \| `Cave` |
| `ANCHOR` | a hex signature locating the site, `??` a wildcard byte (`"8b ?? 24 08"`); it MUST match exactly one place. Or give `OFFSET`. |
| `OFFSET` | a fixed address: a virtual address when at or above the PE image base, else a raw file offset. Hex (`"0x44be08"`) or decimal. A file that is not a PE image (an ELF, a data file) takes `ANCHOR`s and raw offsets; `Cave` needs a PE image. |
| `EXPECT` | hex of the original bytes at the site. A mismatch is an error (§6.4); bytes already equal to the patched result are skipped (idempotent). Required for `Replace`, `ReplaceAll` and `Cave`, recommended for `Poke`, refused for `Or`. |
| `APPLY` | `"prefix"` (default) or `"memory"` (below) |
| `COMMENT` | what the op does. Seventy-five offsets without prose are unmaintainable. |

### `Replace`: overwrite bytes

Writes `REPLACE` (hex) at the site. `REPLACE` may be shorter than `EXPECT`; it is padded with `0x90` (NOP) to the
`EXPECT` length. The pad is x86 code, so only a PE image takes it: in any other file `REPLACE` covers all of `EXPECT`. The No-CD primitive.

### `ReplaceAll`: every match of a pattern

Takes no site (`ANCHOR` and `OFFSET` are refused): `EXPECT` is the pattern, and every place it occurs becomes
`REPLACE`, of the same length. For a fix that rewrites one header wherever it appears, such as the sample rate in
each of a sound archive's thousands of WAV headers. Ops run in their order, so a later op may match what an earlier
one wrote, as the fix's own tool would. A file where `EXPECT` occurs nowhere but `REPLACE` does is already patched
(skipped); one where neither occurs is the wrong file (§6.4). Not for `APPLY: "memory"`.

### `Poke`: write a value

Writes `VALUE` (hex) at the site, usually a variable rendered with a byte format, so a setting is baked into the
binary: `"VALUE": "%ResWidth:u16le%"` (chapter 08 §8.4).

### `Or`: set bits in a shared flag byte

ORs `VALUE` (hex) into the bytes at the site. For a byte that several independent options share (UserPatch keeps
installer checkboxes as single bits of one byte): each option ORs its own bits, so any subset composes in any order.
Bits already set are the idempotent skip. `EXPECT` is refused, because the byte legitimately holds other options'
bits.

### `Cave`: a jmp trampoline

Moves `EXPECT` (at least 5 bytes, room for a `jmp rel32`) into a **code cave** and runs code around it. The cave is
`PAYLOAD` + the displaced original bytes + a `jmp` back past the site; the site becomes `jmp <cave>`, NOP-padded.

| Field | Meaning |
|-------|---------|
| `PAYLOAD` | hex of the code to run before the displaced bytes |
| `CAVE` | `"auto"` (default: the first run of zero bytes long enough in an executable section) or a fixed virtual address |

### `APPLY: "memory"`

`"prefix"` (the default) patches the runtime's copy of the file on disk: it works for every runner and every launch
path, because it is what the loader reads. `"memory"` patches the **running process** instead: the process is created
suspended, patched, then resumed before its first instruction. Use it only when the file on disk must stay pristine
(anti-tamper that checksums the executable).

It needs a handle to the game's process, so it applies only when the entry starts the game through a helper that
creates the process itself (VidyaGod's `vglobby`, which also hosts DirectPlay sessions), only for `Replace` and
`Poke`, and only with an `OFFSET` that is a virtual address. This mode is optional for an implementation. An op it
cannot apply MUST be reported, never dropped silently.

*(VidyaGod: `BinaryPatch::ApplyOne` in `binarypatch.cpp`; `APPLY: "memory"` becomes `vglobby --mem-patch` arguments.)*

## 6.4 Failures

- A binary op whose `EXPECT` does not match (and whose result is not already present) is an error naming the file,
  the offset and the bytes found; the file stays unpatched.
- A text op that fails (a `ConfigWrite` on a missing file, a `JsonSet` on a non-JSON file) is an error naming its
  mode, key and file.
- An implementation MUST NOT report a launch with failed edits as clean. It MAY still start the game (VidyaGod does,
  with the errors in the launch log and its verdict), since a failed setting is usually not a reason to refuse play.
- An edit whose file resolves outside the runtime is refused.

## 6.5 Applicability

| Op | Native or emulator runner | Wine-family runner |
|----|---------------------------|--------------------|
| text ops | applied | applied |
| binary, `APPLY: "prefix"` | applied | applied |
| binary, `APPLY: "memory"` | through a process-creating helper | through a process-creating helper |

*(VidyaGod: `FileEdits` in `fileedits.cpp` (text), `BinaryPatch` in `binarypatch.cpp`; `NodeLower::LowerPlan`
orders them.)*

Next: [`REG`, `DLL`, `ENV`](07-registry-dll-env.md).
