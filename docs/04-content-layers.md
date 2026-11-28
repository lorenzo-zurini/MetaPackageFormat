# 04 · Content layers: `ZIP` `FILE` `DELTA` `CODED` `DIR`

Content layers put **files** into the runtime. They are the bulk of any package.

```json
{ "ZIP": "Doukutsu Monogatari 1.0.0.6.zip", "SOURCE": "QmXLSRno…", "SIZE": 2392064, "TARGET": "FILES/%GameDir%" }
```

| Field | Meaning |
|-------|---------|
| type key | the file (or directory) name, relative to the node's package folder; or, for `DIR` only, a runtime directory anchored at a `%variable%` (`%DefaultPfxDir%`) |
| `SOURCE` | (`ZIP` `FILE` `DELTA` `CODED`) the CID of the file's bytes: how to obtain it when absent (chapter 15) |
| `SIZE` | (`ZIP` `FILE` `DELTA` `CODED`) its size in bytes, stamped at publish: a download-total hint, not an integrity check |
| `TARGET` | the `FILES` address the content is placed under (default `FILES`, the root of the current placement) |
| `BASE_SOURCES` | (`DELTA` `CODED`, required) the `SOURCE` CIDs of the layers whose bytes, concatenated in that order, are the delta's base (`[]`: a delta over nothing, the file stored compressed); for `CODED`, exactly one: its content |
| `DESCRIPTION` | (`CODED`, required) the format's coding description, inline: an object (chapter 22 §22.16) |
| `NAME` | (`CODED`, required) the file name the shipped stream appears by, in `TARGET` |

A file name MUST stay inside the package: no `..` segment, no absolute path. Nodes arrive from peers, and an escaping
name would make a download write outside the package folder.

## 4.1 The five kinds

| Type | The bytes are… | Mounted as | Published? |
|------|----------------|-----------|------------|
| `ZIP` | a **STORE (uncompressed) zip** | its entries, under `TARGET`, served from inside the zip without extraction | yes |
| `FILE` | one regular file | that file, at `TARGET/<its name>` | yes |
| `DELTA` | a `.vgdelta` over the layers it names | the reconstructed archive's entries, under `TARGET`, read on demand | yes |
| `CODED` | a record of decisions (chapter 22) | ONE file, `TARGET/<NAME>`: the shipped stream coded back from its content and the record, bit-exact | yes |
| `DIR` | a directory tree | its files, under `TARGET` | no: local only |

### `ZIP`

The normal way to ship content. **Every entry MUST be STOREd** (no compression): the runtime serves each file from
its offset inside the zip, so the download *is* the install, with no extraction and no second copy. A zip with any
DEFLATE entry MUST be refused at validation and publish, never discovered at launch. Create archives with
`zip -0 -r`.

**Links.** A symbolic link stored in the archive (a Unix link entry, its target as the entry's bytes) is served in one
of two ways:

- **As the file it points to,** when it resolves to a file inside the archive. Its target is read against the link's own
  folder, in the archive's own names. Such a link was written for the archive's tree, and its target may not exist
  where the layer is mounted (a template's `drivers/x.sys -> ../../lib/x.sys`).
- **As the link it is, its target as stored,** for any other link: an absolute target, one that climbs out of the
  archive, one to a folder, or a dangling one. It is resolved where the game is, inside the runtime and the sandbox, as
  an installed copy's link would be (a self-mounting bundle's `entrypoint -> /nix/store/…`).

A `DELTA`'s archive is served the same way. A program outside the sandbox that reads or writes the runtime never
follows a link that leads out of it (chapters 06, 13).

### `FILE`

One file: a ROM, a loose DLL, a cover image. **`TARGET` is the containing folder**; the file keeps its name and
appears at `TARGET/<name>`. Naming the file itself in `TARGET` makes a folder of that name. To land a file under
another name, contain its node and rename it with a `TAKE` pair (chapter 05).

### `DELTA`

A `.vgdelta` is a random-access binary delta that reconstructs a **complete** archive from a base. **A delta names its
base:** `BASE_SOURCES` lists the `SOURCE` CIDs of the `ZIP`, `FILE` and `DELTA` layers it was built over, and their
bytes, concatenated in that order, are the base. A delta's bytes are an archive, so a delta can name another delta, and
a chain of deltas reads like a version history:

```jsonc
{ "LABEL": "wc3_1_27a", "LAYERS": [
    { "NODE": "bafkrei…wc3_1_26a" },                         // its 1.26a archive: { "DELTA": …, "SOURCE": "Qm…26a" }
    { "DELTA": "delta_from__wc3_1_26a.vgdelta", "SOURCE": "Qm…27a", "BASE_SOURCES": ["Qm…26a"],
      "TARGET": "FILES/%GameDir%" },
    … ] }
```

A `FILE`'s bytes are the file as it is: a disc image is stored as it was dumped, and the files extracted from it are a
delta over it, with no archive around the image:

```jsonc
{ "LABEL": "jsrf_files", "LAYERS": [
    { "NODE": "bafkrei…jsrf_disc" },                         // { "FILE": "JSRF (USA).iso", "SOURCE": "Qm…iso", "TARGET": "FILES/%GameDir%" }
    { "DELTA": "JSRF - disc files.vgdelta", "SOURCE": "Qm…files", "BASE_SOURCES": ["Qm…iso"], "TARGET": "FILES/%GameDir%" },
    … ] }
```

Where the files are mounted the image they were rebuilt from does not show beside them (what it replaces, below).

There is no implicit base. A base is named by its content, never by where it is placed, so a delta keeps its base
however the package around it is rearranged, and one delta can be built over any bytes the package holds: a version
over the version before it, a macOS build over the Windows build at another address, a Wine prefix over Wine and DXVK.

**A delta over nothing** (`"BASE_SOURCES": []`, 2026-11-27) is a file STORED COMPRESSED: every byte is its own,
in independently decompressible zstd blocks, so it reads with random access like any delta. It is how a `CODED` layer's
decoded content is kept at rest (decoded content is several times the size of the shipped stream; compressed, it is
about the shipped size or less), and any later build's content is a delta over it as usual.

- **Naming.** `BASE_SOURCES` is REQUIRED on a `DELTA` and allowed on nothing else. A validator refuses it absent, an
  entry that is not a non-empty string, a CID named twice, and `BASE_SOURCES` on any other layer (`[]` is a delta over
  nothing, above). A layer named as a base
  MUST carry its `SOURCE`: a tool that makes a delta over a layer without one computes that layer's CID first.
- **Resolving.** Each entry is the file its CID names: any `ZIP`, `FILE` or `DELTA` layer in the launch's composition
  (chapter 12) whose `SOURCE` is that CID. One CID is one file, so where it is placed — beneath the delta or above it,
  whatever its `TARGET`, seen whole or through a `TAKE` — never changes the bytes: a zip's file, a file's own bytes, or
  a delta's reconstructed archive.
  A delta naming a file that is not in its launch (not contained, or gated off by `WHEN`) MUST be refused at validation
  and MUST NOT be mounted.
- **What it replaces.** Where the delta is mounted, it hides every placement of a file it names (and of whatever that
  file was itself rebuilt from) that lies beneath it in the composition **inside the places the delta is mounted** (its
  `TARGET`, or where a `TAKE` places what it shows) — so a file the newer
  version dropped never shows through, however many times the package contains its base. Outside it, a base is only a
  source of bytes and stays as it is: a macOS depot rebuilt over the Windows depots at `%GameDir%` leaves them there
  untouched. Nothing the delta does not name is hidden: another archive at the same target (a Steam language depot
  beside the main depot) shows through as it would through any zip, and a `ZIP` stacked above the delta is an ordinary
  layer above it. A delta that cannot rebuild hides nothing.
- A delta MUST reconstruct the complete tree of what it replaces. Authoring one from an overlay zip that holds only
  the changed files produces a delta that hides the rest of its base.
- Reading an entry costs `O(log n)` in the chain length, so a thousand versions can ship as one chain. An
  implementation MAY flatten a chain internally.
- A delta whose named bases are not the bytes it was built over fails its base check: their joined size against the
  size the delta records (an authoring tool also checks the hash it records). It MUST NOT be mounted as if it had
  reconstructed, and the implementation MUST report it.

### `CODED`

A shipped file (a compressed archive, an installer's stream) kept as its **decoded content** plus a small **record** of
the decisions its encoder made, through a **description** of its format: data, never code (chapter 22). The game reads
the shipped bytes, bit-exact; the package stores what decodes, dedups and deltas well.

```jsonc
{ "LABEL": "Skyrim SE - 2024-12-05 (Depot 489833) - Interface", "LAYERS": [
    { "FILE": "Skyrim - Interface.bsa.content", "SOURCE": "bafk…content", "TARGET": "FILES/%GameDir%/Data" },
    { "CODED": "Skyrim - Interface.bsa.vgcr", "SOURCE": "bafk…record", "NAME": "Skyrim - Interface.bsa",
      "DESCRIPTION": { "NAME": "BSA 105", "ISA": 6, "CODER": "BITS", "…": "…", "CODE": ["…"] },   // 22.16
      "BASE_SOURCES": ["bafk…content"], "TARGET": "FILES/%GameDir%/Data" } ] }
```

- **Its content is named by CID**, like a delta's bases: any `ZIP`, `FILE` or `DELTA` layer in the launch whose
  `SOURCE` is that CID. The content may itself be a `DELTA` (the next build's content as a delta over the last one's,
  the dedup chapter 22 measures). A `CODED` layer naming a content CID that is not in its launch MUST be refused at
  validation and MUST NOT be mounted.
- **Its description is the layer's own** (USER 2026-11-27: "the description needs to exist in the nodes that have that
  format"): data inline in `DESCRIPTION` (22.16), never a file, never a reference. It MUST read and validate; a node
  carrying one that does not is refused where nodes are checked. Each node coding a format carries its copy (a record
  reads only through the description it was minted with, so the two travel together); a node has no `DESCRIPTION` of
  its own.
- **The content is bytes, never files.** It is mounted for its bytes only and shows nowhere: the game sees `NAME`, not
  `NAME.content`.
- **The record is the layer's own file** (`CODED`, its `SOURCE`, its `SIZE`), fetched like any content file.
- **Made by minting** (`coding mint`, chapter 23 §23.9): decode the shipped file into content and record, code it back,
  and refuse unless every byte matches. A record is only ever published after that check.
- **Predicted records** (2026-11-27). A description with an ENCODER FAMILY (chapter 22.15.4: the encoder that made the
  stream, e.g. Bethesda's packer for Skyrim SE's archives) mints a PREDICTED record (`VGCQ`): only what the encoder did
  otherwise than its family predicts, so it is KB where a plain record (`VGCR`, every decision) is hundreds of MB. That
  is the form a package keeps and sends. Reading through it runs the family, so an implementation SHOULD keep, per
  machine, its PLAIN TWIN (the same stream's every decision: `coding expand`) and mount that: derived data, keyed by the
  record's, the content's and the description's CID (of its canonical text), made once (the first launch that needs it; `coding twins` ahead of
  it), refused unless every segment matches the hash the predicted record carries for it. A twin that cannot be made
  (no room, content that is not the record's) is remembered as such and leaves the predicted record in use: the same
  bytes, slower. A record claiming a stream far bigger than its content is refused before any scratch is made. Packages
  never carry twins. (Windows: twins are not made yet; predicted records are read as they are.)

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

A node has one home (chapter 02 §2.7), so a content file has exactly one place: its name in its package, or its
bumped name when two versions of that name are kept side by side (chapter 15 §15.6.1).

## 4.3 Placement

A content layer's files land under its `TARGET`, which is placed in turn by every `NODE` layer that contains its node
(chapter 05 §5.3). Write targets from an anchor (`FILES/%GameDir%`, `FILES/%SysDir32%`), so the same package lands
correctly under every runner. A runner's own build uses relative targets in its own layout (`FILES/files/lib64`,
`FILES/pfx`).

## 4.4 Stacking

Per file, the **later** content layer in the composition wins (chapter 12): a patch above the base replaces the base's
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
