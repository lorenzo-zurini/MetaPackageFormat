# 05 · `Content` nodes

A `Content` node contributes **files** to the runtime overlay. It is the only type that does, and it is the bulk of any
real package.

The other payload-bearing types are covered elsewhere: the edit types `RegEdit`/`FileEdit`/`BinaryPatch`/`DllOverride`
in [chapter 6](06-edit-layers.md), `Persist` in [chapter 7](07-persistence.md), `CustomVar` in
[chapter 8](08-variables.md), and `DeclareExec`/`DeclareLibraryItem` in [chapter 9](09-exec.md) /
[chapter 3 §3.3](03-roles.md).

Any node MAY carry a **`WHEN`** condition ([chapter 8 §8.8](08-variables.md#88-when--conditional-layers)): when it does
not hold, the node is **inert** — its payload is not applied, though it remains in the graph and its `PARENTS` are still
reached. This is how the format expresses conditional, data-driven behaviour.

## 5.1 `FORM` — the four shapes of content

A `Content` node's `FORM` says how its `PATH` is to be read. All four contribute files to the same overlay; they differ
only in how the bytes are stored on disk.

| `FORM` | Source is a… | Mounted as | Publishable? |
|--------|--------------|-----------|--------------|
| `zip` | **STORE (uncompressed) zip** | its entries, served zero-copy from inside the zip | yes |
| `file` | a single regular file | that one file, placed into the target directory under its basename | yes |
| `dir` | a directory tree | its files, walked from disk | **no** (authoring-only) |
| `delta` | a `.vgdelta` over a **base** | the reconstructed archive's entries, read on demand | yes |

`FORM` is required, and an unknown value MUST be an error.

### `zip` — the production form

The canonical way to ship content. The payload is a **zip archive whose every entry is STOREd (uncompressed)**. The
runtime mounts the archive's entries directly, reading each file at its offset inside the zip without inflating — so a
multi-gigabyte game is "installed" by mounting one zip, with no extraction step and no second copy on disk.

> **STORE is mandatory.** A `zip` whose archive contains any DEFLATE-compressed entry **will not mount** and MUST be
> rejected (at validation/publish, not silently at launch). Create archives with no compression:
> `zip -0 -r out.zip files…`. (VidyaGod: `ZipFullyStored` / `ZipFirstCompressedEntry`; the validator and mounter both
> enforce it, and the editor offers a one-click **re-store** on a node whose zip is compressed.)

```json
{ "NODE_ID": "aom_content", "TYPE": "Content", "FORM": "zip", "PATH": "aom.zip",
  "SOURCE": { "TYPE": "ipfs", "CID": "Qm…" } }
```

### `file` — a single loose file

For content that is naturally one file: a ROM, a single patched executable, a loose DLL. The file is placed into the
node's target directory keeping its own basename.

```json
{ "NODE_ID": "starvoyager_rom", "TYPE": "Content", "FORM": "file", "PATH": "StarVoyager.sfc" }
```

With no `TARGET`, this lands at the content root as `StarVoyager.sfc`.

### `dir` — an unzipped authoring intermediary

A directory of loose files, walked from disk. It is the convenient form **while authoring/testing** a package, but it
**cannot be published** — the content network seeds files and zips, not arbitrary directory trees. A validator MUST warn
that a `dir` node should be converted to a STORE `zip` before publishing. It still runs locally, so packagers can
iterate before sealing content into a zip. (VidyaGod surfaces a one-click **→ zip** conversion on the node itself.)

```json
{ "NODE_ID": "aom_wip", "TYPE": "Content", "FORM": "dir", "PATH": "game_files/" }
```

### `delta` — content expressed as a diff of other content

A `.vgdelta`: a random-access binary delta over a **base**, which is the content some other `Content` node already
supplies. The runtime reconstructs entries on demand, so a chain of deltas still reads in `O(log n)` — this is what
makes 900 versions of one game ship as one chain instead of 900 archives.

**The base is implicit: the composed view already mounted at this layer's own `TARGET`** — i.e. everything below it
in the overlay. That is what a chain is, and it is why the overwhelming majority of deltas declare no base at all:

```json
{ "NODE_ID": "mc_1_20_2", "TYPE": "Content", "FORM": "delta",
  "PATH": "1.20.2.vgdelta", "TARGET": "%PrefixRoot%/drive_c/minecraft",
  "SOURCE": { "TYPE": "ipfs", "CID": "Qm…" } }
```

`BASE_TARGETS` names a **different** base, and is only needed when the delta is not over the thing it mounts onto:

- **`BASE_TARGETS`** (array of string) — the mount targets whose composed content is this delta's byte base. The
  base is the **concatenation** of those targets, in the order given, which lets one delta dedup against several
  independent trees at once (a wine build ‖ a DXVK build ‖ the previous prefix). It is **always a list**: a
  one-element list is the ordinary cross-target delta, and there is deliberately no singular spelling — `""` is a
  real target (the mount root), so a lone string could not distinguish a base declared *at* the root from no base
  declared at all, and the runtime would silently reconstruct against the wrong bytes.

  Each named target MUST be one that some layer in the closure **already mounts at, earlier in the plan** — and it
  must be a target whose content is a byte stream (a `zip` or a `delta`); a `dir` or `file` layer mounts fine and is
  not a base. A base naming anything else is an error ([chapter 15](15-validation.md)): the runtime finds no bytes,
  skips the layer, and the package launches with that content silently absent.

  The order is load-bearing — it must match the concatenation the delta was generated against.

> **A delta must reconstruct a COMPLETE tree.** Authoring a delta from an *overlay* zip (one that only contains the
> files that changed) produces a delta that masks the base rather than replacing it. The base is the full tree, and so
> is the result.

## 5.2 Locating the bytes: `PATH` and `SOURCE`

Two keys cooperate:

- **`PATH`** (string) — the **local** path of the content, relative to the node's bundle directory (or absolute). This
  is where the bytes live (or will live after fetching). A `Content` node MUST declare a `PATH` (directly or via
  `SOURCE.PATH`); one with no path at all is an error.
- **`SOURCE`** (object) — a content-addressed locator describing how to *obtain* the bytes if the local `PATH` is
  absent. Fully specified in [chapter 14](14-content-addressing.md). Shape:
  ```json
  "SOURCE": { "TYPE": "ipfs", "CID": "Qm…", "PATH": "optional override of the local path" }
  ```
  - `SOURCE.PATH`, if present, **overrides** the top-level `PATH` as the local location.
  - `SOURCE.TYPE: "ipfs"` + `CID` means "fetch this CID to the local path." `SOURCE.TYPE: "path"` (the default) means
    local-only, no remote.

**Resolution rule (invariant I8).** A file present at the resolved local path is authoritative and is used as-is. The
`SOURCE`/`CID` is consulted **only** when that local file is missing — to fetch it. A node with neither a present local
file nor a fetchable `CID` is *unavailable*, and any launch that needs it MUST be refused with a clear diagnostic.

(VidyaGod: `LayerLocator` computes `(localPath, cid)`; `EnsureSources`/`MaterializeLayers` apply I8.)

## 5.3 Placement: `TARGET` and the content root

By default a `Content` node's files mount at the **content root** — a runner-chosen location within the runtime
(chapters 9/13; `""` for native content, `pfx/drive_c/<uid>` inside a Proton prefix). A node MAY shift its files to a
subdirectory with `TARGET`:

- **`TARGET`** (string) — a path this node's files are mounted *under*. Targets are anchored with `%variables%`
  (`%PrefixRoot%/drive_c/%PackageUID%`) rather than written as bare relative paths, so that the same package lands
  correctly under every runner.

```json
{ "NODE_ID": "mw_hd_textures", "TYPE": "Content", "FORM": "zip", "PATH": "hd_textures.zip",
  "TARGET": "%PrefixRoot%/drive_c/%PackageUID%/Data Files/Textures" }
```

For `FORM: "file"`, `TARGET` is the directory the single file is placed into (the file keeps its basename). For `zip`,
`dir` and `delta`, `TARGET` is prepended to every entry's path.

## 5.4 Sub-mounts: `SUBMOUNTS`

- **`SUBMOUNTS`** (array, optional) — declares nested mount points within this node's content: entries of the form
  `source/path:dest/path` relocate a subtree of the archive to somewhere else in the runtime, without repacking it.
  Most nodes omit it (default `[]`). It applies to every `FORM`, deltas included. An implementation that does not
  support nested submounts MAY ignore the key, but SHOULD document the limitation. (VidyaGod forwards `SUBMOUNTS`
  verbatim into the vidyagodfs layer spec.)

## 5.5 Stacking, priority and conflicts

When multiple `Content` nodes contribute the same path, the **higher-priority** one wins. Priority is the resolved
closure order (chapter 12): a node is applied after every node it depends on, so **a child wins over its parents**. The
launchable is the terminal node of its chain and therefore highest of all. This is what makes mods work: a mod node that
lists the base as a parent overrides the base's files; an `OVERRIDE` edit (chapter 6) wins over everything including the
user's saved state.

Two nodes with **no dependency relation** that write the same path are an **unordered write conflict** — see
[ch. 2 §2.4](02-nodes.md). Do not resolve it by reordering an array; resolve it with an edge.

### Case sensitivity (a real-world hazard)

The overlay is **case-sensitive**, but a lot of content targets case-insensitive platforms (Windows). Two failure modes:

- **`PATH` case mismatch** — a launchable's `DeclareExec.PATH` whose casing differs from the actual file (e.g.
  `MW4Mercs.exe` vs `MW4mercs.exe`) resolves to nothing → silent crash. Validators MUST check it against the real
  (locally-present) content, case-exactly, and error on a case-only mismatch (chapter 15).
- **Cross-layer case collisions** — two *different* nodes contributing paths that differ only in case (base ships
  `MAPS/foo`, a patch ships `maps/foo`). On the case-sensitive mount both exist and a lookup can hit the wrong one →
  crashes/missing data. Validators MUST treat this as an error. (Collisions *within a single archive* are the upstream
  content's own business — they merge fine on the real target OS — and are ignored.)

(VidyaGod: `GatherLaunchContentFiles`, `FindCrossLayerCaseCollisions` in `manifestmodel.cpp`.)

## 5.6 Why zero-copy zips matter (non-normative)

Serving STORE zips by offset means a package's "install size" is its download size: no decompressed second copy, no
extraction time, and the same bytes can be content-addressed (the zip's CID *is* the install). It is also why DEFLATE is
forbidden — you cannot serve a compressed entry at a stable file offset without inflating it. The constraint is the
price of making "download = installed."

Next: [Edit nodes](06-edit-layers.md).
