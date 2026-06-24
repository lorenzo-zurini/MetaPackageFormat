# 05 · VFS content layers

`LAYERS` is the array of typed payloads a node contributes. There are three families of layer:

- **VFS layers** (this chapter): `VFSZipLayer`, `VFSDirLayer`, `VFSFileLayer` — contribute *files* to the overlay.
- **Edit layers** ([chapter 6](06-edit-layers.md)): `RegEdit`, `DllOverride`, `FileEdit` — *mutate* files/registry.
- **Persistence & knob layers** ([chapter 7](07-persistence.md), [chapter 8](08-variables.md)): `Persist`,
  `CustomVar` — declare durable state (one self-describing primitive) and user variables.

Every layer is an object with a `TYPE`. Order within `LAYERS` is preserved and, combined with the node closure order
(chapter 12), determines overlay priority. Unknown `TYPE`s MUST be ignored (a validator MAY warn).

## 5.1 The three VFS layer types

A VFS layer mounts file content into the runtime overlay.

| `TYPE` | Source is a… | Mounted as | Publishable? |
|--------|--------------|-----------|--------------|
| `VFSZipLayer` | **STORE (uncompressed) zip** | its entries, served zero-copy from inside the zip | yes |
| `VFSFileLayer` | a single regular file | that one file, placed into the target directory under its basename | yes |
| `VFSDirLayer` | a directory tree | its files, walked from disk | **no** (authoring-only) |

All three contribute files to the same overlay; they differ only in how the bytes are stored on disk.

### `VFSZipLayer` — the production form

The canonical way to ship content. The payload is a **zip archive whose every entry is STOREd (uncompressed)**. The
runtime mounts the archive's entries directly, reading each file at its offset inside the zip without inflating — so a
multi-gigabyte game is "installed" by mounting one zip, with no extraction step and no second copy on disk.

> **STORE is mandatory.** A `VFSZipLayer` whose archive contains any DEFLATE-compressed entry **will not mount** and MUST
> be rejected (at validation/publish, not silently at launch). Create archives with no compression:
> `zip -0 -r out.zip files…`. (VidyaGod: `ZipFullyStored` / `ZipFirstCompressedEntry`; the validator and mounter both
> enforce it.)

```json
{ "TYPE": "VFSZipLayer", "PATH": "aom.zip",
  "SOURCE": { "TYPE": "ipfs", "CID": "Qm…" } }
```

### `VFSFileLayer` — a single loose file

For content that is naturally one file: a ROM, a single patched executable, a loose DLL. The file is placed into the
layer's target directory keeping its own basename.

```json
{ "TYPE": "VFSFileLayer", "PATH": "StarVoyager.sfc" }
```

With no `TARGET`, this lands at the content root as `StarVoyager.sfc`.

### `VFSDirLayer` — an unzipped authoring intermediary

A directory of loose files, walked from disk. It is the convenient form **while authoring/testing** a package, but it
**cannot be published** — the content network seeds files and zips, not arbitrary directory trees. A validator MUST warn
that a `VFSDirLayer` should be converted to a STORE `VFSZipLayer` before publishing. It still runs locally, so packagers
can iterate before sealing content into a zip. (VidyaGod surfaces a one-click “→ ZIP” conversion.)

```json
{ "TYPE": "VFSDirLayer", "PATH": "game_files/" }
```

## 5.2 Locating a layer's bytes: `PATH` and `SOURCE`

Every VFS layer needs to name the bytes. Two keys cooperate:

- **`PATH`** (string) — the **local** path of the content, relative to the node's bundle directory (or absolute). This is
  where the bytes live (or will live after fetching). A VFS layer MUST declare a `PATH` (directly or via `SOURCE.PATH`);
  a VFS layer with no path at all is an error.
- **`SOURCE`** (object) — a content-addressed locator describing how to *obtain* the bytes if the local `PATH` is absent.
  Fully specified in [chapter 14](14-content-addressing.md). Shape:
  ```json
  "SOURCE": { "TYPE": "ipfs", "CID": "Qm…", "PATH": "optional override of the local path" }
  ```
  - `SOURCE.PATH`, if present, **overrides** the top-level `PATH` as the local location.
  - `SOURCE.TYPE: "ipfs"` + `CID` means "fetch this CID to the local path." `SOURCE.TYPE: "path"` (the default) means
    local-only, no remote.

**Resolution rule (invariant I8).** A file present at the resolved local path is authoritative and is used as-is. The
`SOURCE`/`CID` is consulted **only** when that local file is missing — to fetch it. A layer with neither a present local
file nor a fetchable `CID` is *unavailable*, and any launch that needs it MUST be refused with a clear diagnostic.

(VidyaGod: `LayerLocator` computes `(localPath, cid)`; `EnsureSources`/`MaterializeLayers` apply I8.)

## 5.3 Placement: `TARGET` and the content root

By default a VFS layer's files mount at the **content root** — a runner-chosen location within the runtime (chapter 9/13;
`""` for native content, `pfx/drive_c/<uid>` inside a Proton prefix). A layer MAY shift its files to a subdirectory with
`TARGET`:

- **`TARGET`** (string) — a path, relative to the content root, that this layer's files are mounted *under*. Joined as
  `contentRoot + "/" + TARGET`. Use it to place a mod under a subfolder, or to lay an overlay at a specific game-relative
  location.

```json
{ "TYPE": "VFSZipLayer", "PATH": "hd_textures.zip", "TARGET": "Data Files/Textures" }
```

For a `VFSFileLayer`, `TARGET` is the directory the single file is placed into (the file keeps its basename). For zip/dir
layers, `TARGET` is prepended to every entry's in-archive path.

## 5.4 Sub-mounts: `SUBMOUNTS`

- **`SUBMOUNTS`** (array, optional) — passed through to the overlay mounter to declare nested mount points within this
  layer (e.g. a layer that itself contains directories that should be independent mount roots). It is an
  overlay-implementation detail; most layers omit it (default `[]`). An implementation that does not support nested
  submounts MAY ignore the key, but SHOULD document the limitation. (VidyaGod forwards `SUBMOUNTS` verbatim into the
  vidyagodfs layer spec.)

## 5.5 Stacking, priority and conflicts

When multiple VFS layers contribute the same path, the **higher-priority** layer wins (its file is what the runtime
sees). Priority is determined by the resolved closure order (chapter 12) and, within one node, by `LAYERS` array order:
later = higher. The launchable's own layers are highest of all. This is what makes mods work: a mod node listed *after*
the base in the closure overrides the base's files; an `OVERRIDE` edit (chapter 6) wins over everything including the
user's saved state.

### Case sensitivity (a real-world hazard)

The overlay is **case-sensitive**, but a lot of content targets case-insensitive platforms (Windows). Two failure modes:

- **`CONTENTPATH` case mismatch** — a launchable's `EXEC.CONTENTPATH` whose casing differs from the actual file (e.g.
  `MW4Mercs.exe` vs `MW4mercs.exe`) resolves to nothing → silent crash. Validators MUST check `CONTENTPATH` against the
  real (locally-present) content, case-exactly, and error on a case-only mismatch (chapter 15).
- **Cross-layer case collisions** — two *different* layers contributing paths that differ only in case (base ships
  `MAPS/foo`, a patch ships `maps/foo`). On the case-sensitive mount both exist and a lookup can hit the wrong one →
  crashes/missing data. Validators MUST treat this as an error. (Collisions *within a single layer* are the upstream
  content's own business — they merge fine on the real target OS — and are ignored.)

(VidyaGod: `GatherLaunchContentFiles`, `FindCrossLayerCaseCollisions` in `manifestmodel.cpp`.)

## 5.6 Why zero-copy zips matter (non-normative)

Serving STORE zips by offset means a package's "install size" is its download size: no decompressed second copy, no
extraction time, and the same bytes can be content-addressed (the zip's CID *is* the install). It is also why DEFLATE is
forbidden — you cannot serve a compressed entry at a stable file offset without inflating it. The constraint is the
price of making "download = installed."

Next: [Edit layers](06-edit-layers.md).
