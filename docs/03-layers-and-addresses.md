# 03 · Layers & addresses

A node's `LAYERS` list is ordered bottom → top. Each layer is one typed entry; each writes to **addresses**;
composition (chapter 05 §5.4, chapter 12) decides, per address, which writer wins.

## 3.1 Layer types

A layer has **exactly one type key**. A layer with none, or with two, is malformed and MUST be refused.

| Type key | Payload | Other fields | Writes | Chapter |
|----------|---------|--------------|--------|---------|
| `ZIP` `FILE` `DELTA` | a file name in the node's package | `SOURCE`, `SIZE`, `TARGET` | `FILES/…` at `TARGET` | [04](04-content-layers.md) |
| `CODED` | a record file in the node's package | `SOURCE`, `SIZE`, `TARGET`, `NAME`, `DESCRIPTION`, `BASE_SOURCES` | `FILES/…/<NAME>` at `TARGET` | [04](04-content-layers.md), [22](22-decoding-deltas.md) |
| `DIR` | a directory (a package folder, or a `%runtime%` directory) | `TARGET` | `FILES/…` at `TARGET` | [04](04-content-layers.md) |
| `NODE` | a node CID | `TAKE`, `TARGET` | whatever it takes | [05](05-node-layers.md) |
| `EDIT` | an ordered list of ops | `TARGET` (the file) | that file, or `FILES/<file>#<key>` | [06](06-edit-layers.md) |
| `REG` | `{key path: {value: data} \| {} \| null}` | `ARCH`, `KEY` | `REG/<key>`, `REG/<key>#<value>` | [07](07-registry-dll-env.md) |
| `DLL` | `{dll: order \| null}` | | `DLL/<dll>` | [07](07-registry-dll-env.md) |
| `ENV` | `{NAME: value \| null}` | | `ENV/<NAME>` | [07](07-registry-dll-env.md) |
| `VARS` | `{KEY: declaration}` | | `VARS/<KEY>` | [08](08-variables.md) |
| `EXEC` | `{label: entry}` | | `EXEC/<label>` (and every part of it) | [09](09-exec-tiles-variants.md) |
| `KEEP` | `{address: true \| false \| options}` | | ownership of those addresses | [13](13-ownership.md) |
| `INTO` | a non-empty list of node CIDs (alternatives) | `TAKE`, `RECOMMENDED` | nothing: what a part goes into, a hard link; only at the head of a list | [12](12-resolution.md), [23](23-mods.md) |

A field the layer's type does not list MUST be refused (a misspelt `TARGTE` is an error, not an ignored key).

## 3.2 Fields any layer may carry

| Field | Meaning |
|-------|---------|
| `WHEN` | A condition over variables (chapter 08 §8.6). When it does not hold, the layer is skipped as if absent: a `NODE` layer does not expand, a content layer does not mount, a fact is not written. |
| `COMMENT` | Why the layer is there. Prose for authors; part of the node's identity. |
| `LABEL` | What the layer is called in an editor. Presentation only: resolution never reads it. |
| `SECTION` | A folder in an editor's layer tree (`/` nests). Presentation only. |

An editor that shows a layer without a `LABEL` SHOULD name it from the launch-window label of the variable its
`WHEN` tests (`UI.LABEL`, `UI.SECTION`), else from its payload, so that a feature gated on an option is filed under
that option without repeating it.

## 3.3 Addresses

| Namespace | Address | Key | Case |
|-----------|---------|-----|------|
| `FILES` | `FILES/<path>` | the path | insensitive |
| `FILES` (inside a file) | `FILES/<file>#<key>` | the op's `KEY` | the key is bytewise |
| `REG` | `REG/<hive>\<key…>` (a key), `REG/<hive>\<key…>#<value>` (a value) | key path and value name | insensitive |
| `DLL` | `DLL/<name>` | DLL name | insensitive |
| `ENV` | `ENV/<NAME>` | variable name | bytewise |
| `VARS` | `VARS/<KEY>` | variable key | bytewise |
| `EXEC` | `EXEC/<label>`, `EXEC/<label>/<field>/…` | entry label, then the entry's own fields | bytewise |

- **`TARGET` names the address a layer writes, namespace included.** A content layer at `"TARGET":
  "FILES/%GameDir%"` writes the files it holds under `%GameDir%`; an `EDIT` with `"TARGET":
  "FILES/%GameDir%/game.ini"` writes that file. An absent `TARGET` on a content or `NODE` layer means `FILES` (the
  root of the current placement).
- **`#` ends the path.** Everything after it is the key, one opaque component that may contain `/` or spaces
  (`FILES/%GameDir%/Default.cfg#/W:`). `ConfigWrite`, `JsonSet`, `XmlSet` and `LineFlag` ops write such keyed addresses; every other edit
  writes the whole file.
- `FILES` and `REG` are trees: an address may name a folder or a key, and a `null` or a `KEEP` on it covers
  everything beneath it.
- **A registry key path is written as Windows writes it**: backslashes between key names, doubled in JSON
  (`"REG/HKCU\\Software\\X"`). A `/` inside a key name is part of the name. A value is the key's address, `#`, and
  the value's name (`REG/HKCU\\Software\\X#Volume`); a key name may hold `#`, a value name never, so the value starts
  after the **last** `#`.
- **`EXEC` is a tree too**: `EXEC/Play` is the entry, `EXEC/Play/TILE` its tile, `EXEC/Play/TILE/COVER` the cover.
  Every address is written whole: a later write of `EXEC/Play` replaces the entry, and a part of it changes only by a
  write or a `TAKE` of its deeper address (chapter 09 §9.2, invariant I10). Entries are ordered by where each label
  was **first** written.

## 3.4 Paths and anchors

A package never names a place by drive letter. A path either:

- **starts with an anchor**, a `%Name%` the runner places (§10.4): `%GameDir%/Data`, `%Documents%/My Games/X`,
  `%SysDir32%/d3d8.dll`; or
- is **relative**, and lands under the current placement: the root of the runtime for the node being launched, or
  the `TARGET` of the `NODE` layer that contained this node (chapter 05 §5.3).

The anchors a Windows-family runner defines (VidyaGod's Proton and Windows runners):

| Anchor | Typical guest path |
|--------|--------------------|
| `%GameDir%` | `C:\<PackageUID>` (the family's install folder) |
| `%UserProfile%` | `C:\users\<user>` |
| `%Documents%` | `C:\users\<user>\Documents` |
| `%AppData%` / `%LocalAppData%` | `…\AppData\Roaming` / `…\AppData\Local` |
| `%ProgramData%` | `C:\ProgramData` |
| `%ProgramFiles%` / `%ProgramFiles32%` | `C:\Program Files` / `C:\Program Files (x86)` |
| `%Windows%` | `C:\windows` |
| `%SysDir32%` / `%SysDir64%` | the 32-bit / 64-bit system directory (`syswow64` / `system32` on a 64-bit prefix) |

A runner for another platform defines its own (a native engine defines `%GameDir%` and nothing else). Rules, each a
validator error when broken:

- **No drive letters** in a package (`C:\x`, `E:/`, a value that is only `E:`). Only a runner's own `GUEST_ROOTS`
  and `DRIVES` (§10.4) map drives.
- **An anchor starts a path.** `mods/%GameDir%/x` is refused: it would substitute to `mods/C:\802/x` and mount
  literally. A value that is not a path (a registry string, an argument) may embed an anchor anywhere:
  `"RunDll32 %ProgramFiles32%\\x.dll"`.
- **One spelling per place: the most specific anchor.** `%Windows%/syswow64/x.dll` is refused; write
  `%SysDir32%/x.dll`. Addresses compose and are owned by their spelling, so two spellings of one place would never meet.

Because packages write anchors, the same package runs under Proton, under plain Wine (a different user name) and
natively on Windows (the real profile), and what the user owns is found again after switching runners.

Next: [Content layers](04-content-layers.md).
