# 15 · Packages, CIDs & sharing

Node files are small text; the files they name (game zips, ROMs, runner builds, covers) are large. MPF keeps them
apart and links them by CID: the descriptions are shared freely, the bytes are fetched on demand and verified by hash.

## 15.1 The library and package folders

```
<data root>/LIBRARY/
├── VidyaGod/                                   a library: a named collection of packages
│   ├── [8333][v1.0] Cave Story/                 a package folder
│   │   ├── bafkreihh2d2rpsgouxvmjfoni3az….json  node files, each named by its CID
│   │   ├── bafkreifjjxmj2pay24ffvgdejkpg….json
│   │   ├── Doukutsu Monogatari 1.0.0.6.zip      content files the nodes name
│   │   └── doukutsu-rs 1.0.0 - Linux x64.zip
│   └── …
├── VidyaGodRunners/                            runners are packages like any other
└── VidyaGodLibraries/                          shared libraries (DirectX redistributables, dgVoodoo, …)
<data root>/CATALOG/
└── <friend> - <library>/<package>/.package/    what friends shared, not yet downloaded
```

A **package** is one folder: node files and the content files they name. Its folder name is presentation; nothing
references it. Every node file in every package folder is read into one index keyed by CID (chapter 02 §2.5). The
same node may be held by several folders (an engine two games ship, a friend's stub of a package you have); the
implementation remembers every copy and reads from an installed one whose files are present, before a stub
(chapter 10 §10.6).

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
another package publishes comes along. `ANY` and `NOT` are never followed. Runners are found by capability among
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
3. **Make the package folders.** Per package, a **share folder**: a UnixFS directory of the package's node files,
   each named `<cid>.json`, text only. Its CID is the package's address; its listing is the package's node list. And
   a **pin folder**: the same plus the package's content files (by CID, nothing copied), what a pinning service pins,
   so a pin holds exactly that package's bytes. Libraries a package uses are pinned by their own packages, once.
4. **Announce** and record the share sheet: per library, the list of `{package name, share folder CID}`.

Publishing is reproducible: the same nodes and files give the same CIDs on any machine, so two people who publish the
same package are indistinguishable seeders. Anything that changes node bytes for a cosmetic reason (an editor writing
empty fields, a canvas moving a node) changes the CIDs and is a bug; layout lives in `POS`, outside the identity.

Never published: the instance (owned state, variable values, graft choices, pinned chains).

An implementation SHOULD report what did not publish whole: nodes that did not freeze, and content this machine does
not hold (a receiver relying on a pinning service could never fetch it).

## 15.5 Receiving

A friend shares their share sheet. The receiver:

1. lands each package's share folder as `CATALOG/<friend> - <library>/<package>/.package/`: node files only, fetched
   one by one under their CID names, so a partial transfer resumes;
2. completes each package against its folder's listing, and completes cross-package references the same way;
3. builds the catalog shelf from those nodes (variants × tiled entries, merged by `UID`, chapter 09) and the graft
   index; a received stub is browsable and never offered as a graft (its content is not here);
4. prunes a node file that neither its folder lists nor any closure reaches (an older version's copy).

A changed package is a changed folder CID; an unchanged one is not landed again. There is no library-level CID.

**Downloading** a row **adopts** the packages its closures reach: each moves from `CATALOG` into `LIBRARY` under the
same library and folder names, before any content is fetched, and is from then on an ordinary local package (its
variants launch, its grafts are offered, it is re-published under the same CIDs). An install that would collide with
an existing package name is refused, never merged.

Received bytes are untrusted: every path segment from a share is sanitised, sizes are bounded, and a node's content
file names MUST stay inside its package (chapter 04).

## 15.6 The content backend

The current backend is **IPFS**: CIDs are both the address and the hash, and any peer holding a CID can serve it, so
a popular package is served by everyone who has it. Friends are preferred providers. Fetches, provider searches and
announcements share one bounded network queue, so a large download cannot flood a home router. Nothing in the format
excludes other backends; an implementation MUST support the backends the packages it consumes use.

## 15.7 Integrity and trust (non-normative)

A CID is a hash, so fetched bytes are self-verifying: content that does not hash to its CID is rejected. A CID does
not say whether the publisher is trustworthy. Byte integrity is the format's; trust in a friend's packages is the
user's choice of friends, and the sandbox (chapter 14 §14.5) limits what a package can reach.

Next: [Validation](16-validation.md).
