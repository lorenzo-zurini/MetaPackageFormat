# 04 · Content layers: `ZIP` `FILE` `DELTA` `DIR`

Content layers put **files** into the runtime. They are the bulk of any package.

```json
{ "ZIP": "Doukutsu Monogatari 1.0.0.6.zip", "SOURCE": "QmXLSRno…", "SIZE": 2392064, "TARGET": "FILES/%GameDir%" }
```

| Field | Meaning |
|-------|---------|
| type key | the file (or directory) name, relative to the node's package folder; or, for `DIR` only, a runtime directory anchored at a `%variable%` (`%DefaultPfxDir%`) |
| `SOURCE` | (`ZIP` `FILE` `DELTA`) the CID of the file's bytes: how to obtain it when absent (chapter 15) |
| `SIZE` | (`ZIP` `FILE` `DELTA`) its size in bytes, stamped at publish: a download-total hint, not an integrity check |
| `TARGET` | the `FILES` address the content is placed under (default `FILES`, the root of the current placement) |

A file name MUST stay inside the package: no `..` segment, no absolute path. Nodes arrive from peers, and an escaping
name would make a download write outside the package folder.

## 4.1 The four kinds

| Type | The bytes are… | Mounted as | Published? |
|------|----------------|-----------|------------|
| `ZIP` | a **STORE (uncompressed) zip** | its entries, under `TARGET`, served from inside the zip without extraction | yes |
| `FILE` | one regular file | that file, at `TARGET/<its name>` | yes |
| `DELTA` | a `.vgdelta` over the content beneath it | the reconstructed archive's entries, under `TARGET`, read on demand | yes |
| `DIR` | a directory tree | its files, under `TARGET` | no: local only |

### `ZIP`

The normal way to ship content. **Every entry MUST be STOREd** (no compression): the runtime serves each file from
its offset inside the zip, so the download *is* the install, with no extraction and no second copy. A zip with any
DEFLATE entry MUST be refused at validation and publish, never discovered at launch. Create archives with
`zip -0 -r`.

### `FILE`

One file: a ROM, a loose DLL, a cover image. **`TARGET` is the containing folder**; the file keeps its name and
appears at `TARGET/<name>`. Naming the file itself in `TARGET` makes a folder of that name. To land a file under
another name, contain its node and rename it with a `TAKE` pair (chapter 05).

### `DELTA`

A `.vgdelta` is a random-access binary delta that reconstructs a **complete** archive from a base. **The base is the
nearest content beneath the delta at its own `TARGET`**: the archive (or delta) of the node it was built on, which
the delta's node contains below it. A chain of deltas therefore reads like a version history:

```jsonc
{ "LABEL": "wc3_1_27a", "LAYERS": [
    { "NODE": "bafkrei…wc3_1_26a" },                                    // its base, at FILES/%GameDir%
    { "DELTA": "delta_from__wc3_1_26a.vgdelta", "SOURCE": "Qm…", "TARGET": "FILES/%GameDir%" },
    … ] }
```

- The delta is **opaque**: what it reconstructs replaces everything beneath it at its target, including loose files
  and directories that were not part of its base.
- A delta MUST reconstruct the complete tree. Authoring one from an overlay zip that holds only the changed files
  produces a delta that hides the rest of the base.
- Reading an entry costs `O(log n)` in the chain length, so a thousand versions can ship as one chain. An
  implementation MAY flatten a chain internally.
- A delta whose nearest content beneath is not the one it was built on fails its base size check. It MUST NOT be
  mounted as if it had reconstructed, and the implementation MUST report it.

### `DIR`

A directory of loose files. Two uses:

- **Authoring.** A package folder's subdirectory, handy while iterating. It is never published (the network carries
  files and zips, not trees); a validator SHOULD warn and an editor SHOULD offer to turn it into a STORE zip.
- **Runtime assembly.** A runner places a directory that exists only at launch, named by a variable:
  `{"DIR": "%DefaultPfxDir%", "TARGET": "FILES/pfx"}` puts the runner's default Wine prefix under its own root.

## 4.2 Locating the bytes

A content file lives in its node's package folder under its file name. **Present beats fetch** (I8): a file present
there is used as is; `SOURCE` is consulted only when it is missing, to fetch it into that place. A content layer with
neither a present file nor a `SOURCE` is unavailable, and a launch that needs it MUST be refused with a diagnostic
naming the file.

When one node is held by several package folders (chapter 02 §2.6), any folder's copy will do.

## 4.3 Placement

A content layer's files land under its `TARGET`, which is placed in turn by every `NODE` layer that contains its node
(chapter 05 §5.3). Write targets from an anchor (`FILES/%GameDir%`, `FILES/%SysDir32%`), so the same package lands
correctly under every runner. A runner's own build uses relative targets in its own layout (`FILES/files/lib64`,
`FILES/pfx`).

## 4.4 Stacking

Per file, the **later** content layer in the fold wins (chapter 12): a patch above the base replaces the base's
files, a mod above the patch replaces the patch's. An `EDIT` above them changes the winning file.

### Case

The runtime file system is case-sensitive, while much content targets case-insensitive platforms. Two hazards:

- **An entry's `EXE` (or any path) whose case differs** from the real file resolves to nothing. A validator MUST
  check paths against present content, case-exactly, and refuse a case-only mismatch.
- **Two layers providing paths that differ only in case** (`MAPS/foo` in the base, `maps/foo` in a patch) both
  exist on the mount, and a lookup can hit the wrong one. A validator MUST refuse it. The fix is to rename the
  later layer's entries to the base's case. Collisions inside one archive are the upstream content's own and merge
  on its real platform; they are ignored.

## 4.5 Why zero-copy zips matter (non-normative)

Serving STORE zips by offset makes a package's install size its download size: no decompressed copy, no extraction
time, and the zip's CID is the install. DEFLATE is forbidden because a compressed entry cannot be served at a stable
offset. That constraint is the price of "downloaded = installed".

Next: [`NODE` layers](05-node-layers.md).
