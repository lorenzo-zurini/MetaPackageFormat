# 15 · Packages, CIDs & sharing

Node files are small text; the files they name (game zips, ROMs, runner builds, covers) are large. MPF keeps them
apart and links them by CID: the descriptions are shared freely, the bytes are fetched on demand and verified by hash.

## 15.1 The library and package folders

```
<data root>/LIBRARY/
├── VidyaGod/                                   a library: a named collection of packages
│   ├── [8333] Cave Story/                       a package folder
│   │   ├── Cave Story 1.0.0.6.json              node files, each named by its label (chapter 02 §2.5)
│   │   ├── doukutsu-rs 1.0.0.json
│   │   ├── Doukutsu Monogatari 1.0.0.6.zip      content files the nodes name
│   │   └── doukutsu-rs 1.0.0 - Linux x64.zip
│   └── …
├── VidyaGodRunners/                            runners are packages like any other
└── VidyaGodLibraries/                          shared libraries (DirectX redistributables, dgVoodoo, …)
<data root>/CATALOG/
└── <peer> - <library>/<package>/            what peers shared, not yet downloaded: node files only
```

A **package** is one folder (for a game, the game's folder): node files and the content files they name. Its folder
name is presentation; nothing references it. A game's folder is named `[<uid>] <Title>`; a mod folder that is not a
game `[<game uid>] <Game> - <Author or Mod>` (chapter 23 §23.8). Every node file in every package folder is read into one index keyed by
CID (chapter 02 §2.5).

Every node has **one home package** (chapter 02 §2.7). A package that uses another package's nodes (a game using
DxWnd, Proton using DXVK) contains them by CID with `NODE` layers and holds no copy. Shared things are therefore
packages of their own: libraries in a library collection, shared engines among the runners.

## 15.2 Content CIDs: `SOURCE` and present-beats-fetch

A content layer's `SOURCE` is the CID of its file's bytes, as IPFS gives it to the file added with Kubo's default
parameters; `SIZE` is its byte count (a download-total hint, never an integrity check). A tile cover carries the same
pair (`COVER {FILE, SOURCE, SIZE}`).

**Present beats fetch (I8).** A file present at its place in its package folder is used as is: no hashing, no
network. `SOURCE` is used only to fetch a missing file into that place, and fetched bytes are verified against it.
Hence:

- a package whose files are all present runs offline;
- a missing file with a `SOURCE` is fetchable, not an error until the fetch fails;
- a layer with neither is unavailable, and a launch needing it MUST be refused with a diagnostic;
- a runtime-sourced layer (a `%variable%` path) has no file by design and is outside the rule.

A delta is addressed like any file: a 900-version chain is 900 small objects plus one base, and getting version *N*
fetches only the deltas on its path.

## 15.3 Hydrate and dehydrate

**Hydrating** a row fetches every content file its resolution names, plus the builds of its runner chain (the chain
the resolver picks when nothing is installed: chapter 11 §11.4), plus covers. The CIDs to fetch are followed from the
variant node through `NODE` layers, `SOURCE` and `TILE.COVER.SOURCE`, whichever package they belong to: a library
another package publishes comes along, and so does what a part goes `INTO` (its hard link). A node CID inside a
`WHEN` is never followed. Runners are found by capability among
installed and shared runner packages.

A tile is shown in the library when one of its rows has content present; the rest is catalog.

**Dehydrating** a local package deletes its content files and keeps its node files and covers: it is back to
downloadable.

## 15.4 Publishing

To share packages, an implementation **publishes** them:

1. **Seed content.** Add every content file and cover to the content network (by reference: no copy), and record its
   CID and size in the node (`SOURCE`, `SIZE`). Recording a `SOURCE` changes the node's bytes, so the node, and every
   node that references it, is re-minted (chapter 02 §2.6).
2. **Freeze** every node into its canonical bytes (chapter 02 §2.3) and store them.
3. **Name the node files** by the rule of chapter 02 §2.5.
4. **Record the share sheet**: every published node's CID and where it lives,
   `<library>/<package>/<file>`:

   ```json
   { "bafkreidjdddotl4tkepmmh2ipd4e36bdkotwkomzb3hsr5b45dy455slyi": "VidyaGodLibraries/xinput/xinput_content.json",
     "bafkreics4hd3k4unjsyv2ajblg5zqk75qdkmuysxrsb4jr4ukzs6zzcgwu": "VidyaGod/[6518] Angry Birds Seasons/Angry Birds Seasons 4.1.0.json" }
   ```

   A package is the entries under its `<library>/<package>/`, and nothing else: a package has no CID and no record of
   its own. A CID is one key, so a node has one path: the sheet cannot hold a copy. What a package uses from
   another one is read from its nodes (their `NODE` layers name CIDs) and found in the sheet, never recorded again.
5. **Make a pin folder** per package, for a pinning service: a UnixFS folder of the package's node files under their
   names (`package/`) and its content files (`content/`, by CID, nothing copied), so one pin holds exactly that
   package's bytes. A library a package uses is pinned by its own package, once. The pin folder is the publisher's:
   it is never shared, and a receiver never reads one.
6. **Announce** every node and pin folder.

Publishing is reproducible: the same nodes and files give the same CIDs on any machine, so two people who publish the
same package are indistinguishable seeders. Anything that changes node bytes for a cosmetic reason (an editor writing
empty fields, a canvas moving a node) changes the CIDs and is a bug; layout lives in `POS`, outside the identity.

Never published: the instance (owned state, variable values, graft choices, pinned chains).

An implementation SHOULD report what did not publish whole: nodes that did not freeze, and content this machine does
not hold (a receiver relying on a pinning service could never fetch it).

## 15.5 Receiving

What a peer receives is the share sheet's entries of the libraries shared with them, **closed over hard links**:
every node one of those entries contains (`NODE` layers) or goes into (`INTO`), transitively, comes along with its
whole package, under its own library name. A mod package shared alone brings the game it goes into. A peer given a game also gets the libraries it contains, each as its own package, once. The
closure is worked out by the sender each time it sends, from the nodes themselves. A peer's catalog copy of a
package you have installed is a view of that package, never a second home.

The sheet a peer sends is the receiver's **allow-list**: a block is stored only when its CID is an entry, and only at
that entry's path, once its bytes hash to that CID.

The receiver:

1. lands each entry as `CATALOG/<peer> - <library>/<package>/<file>`, every segment made a safe name (chapter 02
   §2.5); fetched one file at a time, so a partial transfer resumes. Names that clash are compared without case, and
   **every** claimant is told apart, so no name depends on the order entries arrive in: packages whose names make one
   folder name each land in `<folder> (<the last 8 hex digits of the 64-bit FNV-1a hash of its name>)`; entries of one
   package whose names make one file name are each `<stem> [<last 8 characters of its CID>].json`. A receiver bounds
   how many entries it takes, and takes a package whole or not at all;
2. builds the catalog shelf from those nodes (variants × tiled entries, merged by `UID`, chapter 09) and the graft
   index; a received stub is browsable and never offered as a graft (its content is not here);
3. prunes a node file of a received package that the package's entries do not name (an older generation's).

A changed package is changed entries; an unchanged one is not landed again. There is no library-level CID. Nothing is
ever fetched into a package to fill a gap: a node a package contains arrives with its own package. A package that
contains a node that has not arrived (or is not shared) cannot be downloaded yet, and the implementation names the node
by its path in the sheet when one is known.

**Downloading** a row **adopts** the packages its closures reach: each moves from `CATALOG` into `LIBRARY` under the
same library and folder names, before any content is fetched, and is from then on an ordinary local package (its
variants launch, its grafts are offered, it is re-published under the same CIDs).

An install that would collide with an existing package name is refused, never merged, and a package already
installed (or authored here) is never landed again as a stub beside itself: either would put a second copy of its
nodes beside their home.

## 15.6 Other versions of an installed package

A peer may share another version of a package you have installed: they re-minted it (a library, a new engine build
for a recompiled game). There is **no ordering** between versions: two people who build the same package have equally
good histories, so nothing is "newer". An installed package changes only when **you** choose a peer's version.

- **Offered.** A peer's entries for a package installed here (same library, and the folder that package would land
  in, §15.5) are compared with the installed package's nodes, CID for CID. If they differ, the package is shown as "<peer> has a different
  version". A version whose every node is already installed (switched to before, with nodes beyond it kept for other
  games) is not offered again until one of those kept nodes is contained by nothing.
- **Staged.** The version's nodes are fetched beside the library, never inside `LIBRARY` or `CATALOG`, one stage per
  peer and version. A package holding never-saved (draft) node files, whose bytes are not their canonical form, is
  refused.
- **Switched.** Every node the version lists comes in. An installed node it does not list goes **only when nothing
  installed still contains it**: other packages, and the new version itself. A library another game still uses stays
  at the version that game contains, beside the new one. Hard links count (`NODE` and `INTO`: a mod installed for an
  end keeps that end); a condition's CIDs do not. The node files
  are then named by the rule (chapter 02 §2.5) over the package's new set of nodes.
- **Content follows the node set.** A file only removed nodes named is deleted. Bytes already here under another
  name are moved rather than fetched again. When two CIDs claim one file name in a package, each lives at its
  **bumped name** (§15.6.1). The confirmation lists the files to be deleted and their size.
- **Planned twice.** The switch is planned again when applied, against the library as it is then. If that plan
  would delete a file the confirmation did not list, the switch is refused and must be reviewed again.
- **Resumable.** The plan is recorded before the first change. A switch that stops part-way is finished from that
  record when the same version is chosen again; a package half-switched is never planned afresh, since its files no
  longer match its node files.
- **Refused** while a game is running or a download into the package is in progress.
- **Unaffected:** saves and kept files (owned by the family, chapter 13). Choices saved by node CID fall back to
  defaults when the version they named is gone. Content the version names is fetched when next needed.

A peer's game that contains nodes of a version you have not switched to waits, naming the node it needs. An
implementation MUST NOT copy or merge nodes into a package that is not their home to work around it.

### 15.6.1 Bumped names

A node names its content file (`"ZIP": "DxWnd.zip"`) and its `SOURCE` says which bytes; the name is presentation,
the CID is the identity. When one CID claims a name in a package, the file is that name. When two or more do (two
versions of a library kept side by side), **each** lives at `<stem> [<last 8 characters of its CID>]<ext>`
(`DxWnd [1a2b3c4d].zip`), so no version reads another's bytes. Names are compared **without case**: `dx.zip` and
`DX.zip` are one file on a case-insensitive file system, so two CIDs claiming them are bumped apart on every
platform. The rule depends only on the package's node files, so it is the same on every machine and before and after
a download. Node files are named the same way (chapter 02 §2.5). Looking up a content file (a layer's, or a tile's cover) applies it.

## 15.7 The content backend

The current backend is **IPFS**: CIDs are both the address and the hash, and any peer holding a CID can serve it, so
a popular package is served by everyone who has it. Peers are preferred providers. Fetches, provider searches and
announcements share one bounded network queue, so a large download cannot flood a home router. Nothing in the format
excludes other backends; an implementation MUST support the backends the packages it consumes use.

## 15.8 Integrity and trust (non-normative)

A CID is a hash, so fetched bytes are self-verifying: content that does not hash to its CID is rejected. A CID does
not say whether the publisher is trustworthy. Byte integrity is the format's; trust in a peer's packages is the
user's choice of peers, and the sandbox (chapter 14 §14.5) limits what a package can reach.

Next: [Validation](16-validation.md).
