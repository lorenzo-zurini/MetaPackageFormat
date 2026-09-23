# 04 · Bundles, the library & indexing

This chapter defines the on-disk layout, how an implementation discovers nodes, and how a global graph is assembled from
many sources.

## 4.1 Bundles

A **bundle** is a directory containing one or more node `.json` files and the loose content those nodes reference (zips,
ROMs, cover images, generated prefixes). A bundle is purely a *file grouping* — it has no `LABEL`, no metadata of its
own, and no semantic role. It exists so that related files (a launchable, its content nodes, and the bytes they point at)
sit together and can be copied/shared/deleted as a unit.

```
[10972] Silent Hill 2/                 ← a bundle directory
├── silent_hill_2.json                 ← launchable node
├── silent_hill_2_base.json            ← content node (the game files)
├── silent_hill_2_enhanced.json        ← content node (an optional mod)
├── Silent Hill 2.zip                  ← bytes referenced by a layer
├── SH2 Enhanced Edition.zip
├── SH2_Cover.png                      ← bytes referenced by the tile's COVER
└── USERDATA/                          ← per-package durable saves (ch. 7) — created at runtime
```

A bundle MAY contain any number of nodes. A "game package" is typically a launchable (carrying its `TILE`) over its
content chain, plus its grafts; a "runner package" is a node with a `GUEST` entry plus its build. Nothing prevents
one bundle from holding many launchables (e.g. a multi-game collection, or 903 Minecraft versions).

A `.json` file in a bundle holds **one node or a JSON array of them** ([ch. 2 §2.2](02-nodes.md)); an indexer MUST
accept both. Since a chain of twenty nodes is the normal shape now, keeping one chain in one array file is common —
file grouping remains pure presentation with no semantics.

**Non-`.json` files are never nodes.** Publishing is text-only-JSON by construction (§4.4), so anything else in a
bundle reaches neither peers nor the running game — which makes a bundle a safe place for bytes a layer
references (zips, covers) but a poor one for per-machine state, since it still travels with the folder when the
author copies or backs it up. Per-machine state belongs in the tool's own configuration: canvas positions split
into a published default on the node (`POS`) and a local override outside the bundle entirely
([ch. 2 §2.1](02-nodes.md)).

### Relative-path resolution

Every relative path written inside a node — a layer's `PATH`, a `SOURCE.PATH`, a `TILE.COVER.PATH` —
resolves against **the bundle directory of the node that declares it**, not against the launchable being run. This is essential for cross-bundle
composition: when a game's closure pulls in a runner whose build layer says `PATH: "GE-Proton10-30.zip"`, that path
resolves against the *runner's* bundle, wherever the runner lives. (VidyaGod: each node records its `BundleDir`;
`LayerLocator` joins relative paths against it.)

## 4.2 Library roots

A **library root** is a directory whose **immediate subdirectories are bundles**. The graph is built by scanning library
roots. Multiple roots are common and compose into one flat graph:

```
~/.VidyaGod/LIBRARY/                    ← contains library roots, one per configured source
├── VidyaGod/                           ← a library root (a games source)
│   ├── [9001] Vortex Quest/            ← bundle
│   ├── [7804] Age of Mythology/        ← bundle
│   └── …
└── VidyaGodRunners/                    ← a library root (a runners source)
    ├── proton/                         ← bundle
    ├── snes9x/                         ← bundle
    └── native-passthrough/             ← bundle
```

The scan is **two levels deep and no more**: for each library root, the implementation iterates the root's immediate
subdirectories (the bundles), and within each bundle reads the `.json` files **non-recursively**. Nodes are not searched
for at arbitrary depth — content can be nested in subfolders, but the node *files* live at the top of their bundle.
(VidyaGod: `BuildNodeIndex` → `ScanBundleNodes`.)

## 4.3 Building the index

The indexing algorithm, precisely:

```
function BuildNodeIndex(libraryRoots):
    index = {}                                  # CID handle -> node
    for root in libraryRoots:
        if not isDirectory(root): continue
        for bundle in immediateSubdirectories(root):
            if not isDirectory(bundle): continue
            for file in files(bundle) where extension == ".json":
                json = parse(file)              # skip unparseable files (warn)
                # A file holds ONE node or an ARRAY of them (ch. 2 §2.2).
                for entry in (json is array ? json : [json]):
                    node = ParseNode(entry, file, bundle)
                    if node is null: continue    # no TYPE ⇒ not a node, ignore
                    key = node.CID                # the stored CID handle; else a synthetic per-node key
                    if key is empty: key = synthetic(file, entry)
                    if key in index:
                        warn("duplicate node handle, keeping first-seen")
                        index[synthetic(file, entry)] = node   # re-key the loser — never drop it
                        continue                 # invariant I1: first-seen wins
                    index[key] = node
    return index
```

Two builders exist and MUST agree on identity: this fast **working-tree** index keys each node by its stored `CID`
handle (what the editor and the on-disk graph reference), while the **frozen** index re-derives each node's real CID
and keys by *that*. For an unedited tree the two coincide; a stored handle that differs from the derived CID is simply
a node edited-but-not-yet-re-minted. `ParseNode` reads the fields in [chapter 2 §2.2](02-nodes.md), applies defaults,
expands the node's payload into the layers the runtime consumes, and records the source file and bundle directory. A
JSON entry that is not an object, or lacks a string `TYPE`, yields no node and is silently skipped (it may be unrelated
data living in the bundle). `LABEL` is never a key here — it is cosmetic.

**`ParseNode` MUST be total.** A node file is untrusted input — it arrives from a peer, or from an author's typo — and
indexing happens at startup, so an exception escaping here takes the whole runtime down before it can be used to
remove the offending source. A node whose payload cannot be understood (unknown `TYPE` or `FORM`, a wrong-typed field
at any depth, a malformed `EDITS`) MUST therefore be **indexed carrying its error, not dropped**:

- Dropping it removes it from the graph entirely, so a *referrer* dangles loudly but a **leaf** mistake — an unknown
  `FORM` on a `Content` node — is reported by nothing at all, and validation prints a clean bill of health over a
  package that has silently lost a layer.
- Indexed-with-an-error, it contributes **no layers**, validation names it ([ch. 15](15-validation.md)), and
  resolution treats it as missing so a launch that would route through it is refused rather than quietly doing less.

**Duplicate handles (invariant I1).** A `CID` handle is a content hash, so a collision only happens on a **stale** stored
CID (a node edited but not re-minted) or a **forged** one (an untrusted received block claiming a local node's CID). The
**first one seen wins** the handle and the loser is **re-keyed under a synthetic key, never dropped** (dropping it would
vanish a real node — a library legitimately holds many nodes that share a cosmetic `LABEL`). Scan order across roots is
therefore observable; an implementation SHOULD make it deterministic (e.g. roots in a configured order, bundles and files
sorted) AND scan trusted local roots before untrusted received ones, so a received block can never shadow a local handle.
A shared `LABEL` is **not** a collision — cosmetic labels repeat freely. (One legitimate use of handle shadowing: a
user's *local* package overriding a source's copy of the
same id — see §4.5.)

## 4.4 Distribution: sources

Bundles are distributed as ordinary directories, so any transport works. The format mandates none — only that
refreshing node descriptions MUST NOT destroy locally-hydrated content.

The reference implementation uses exactly one channel: a **source is an immutable content-addressed folder** (an IPFS
directory CID) listed in the user's settings and mirrored into a library root. Publishing a new version of a collection
mints a **new CID**; subscribing to it means pointing at that CID. There is no mutable remote, no branch, no merge, and
nothing to go wrong halfway.

Two properties make this work, and both are worth stating as requirements on any transport:

- **A source folder is TEXT-ONLY.** Only the node `.json` files are part of it. Covers, zips, ROMs and deltas travel as
  *content* CIDs referenced **from** those files (ch. 14), so the folder CID is small, reproducible, and identical for
  everyone who mints it from the same nodes. This is also why a non-`.json` file in a bundle (§4.1) is invisible to
  distribution.
- **Refreshing must be content-safe.** The heavy, content-addressed payloads live *inside* the bundles and are hydrated
  locally. Updating the descriptions MUST leave them alone: a sync replaces node text, never the bytes beside it. The
  reference fetches the new source folder and reconciles the node files, leaving hydrated content in place.

(VidyaGod: `Settings.PackageSources[]`, `PackageCatalog::SyncPackageSources`, `PublishMetaCid`.)

> *Historical note.* An earlier generation of the reference implementation modelled a source as a **git remote** cloned
> into a library root, with `--ff-only` pulls and a hard-reset fallback. It was removed: a mutable remote gives a
> package graph no property a content address does not, while adding merge states, partial clones, and a second thing
> that can be out of date. A CID either resolves to exactly those bytes or it does not.

## 4.5 Sharing: the package manifest

Publishing a library freezes every node into its dag-json block and mints, per bundle dir, one **package
manifest**:

```json
{ "VIDYAGOD_PACKAGE_MANIFEST": 1, "PKG": "[749][v1.0] Age of Empires II",
  "NODES": [ { "/": "baguqeera…" }, { "/": "baguqeera…" }, … ] }
```

It is not a node (a scan refuses its vocabulary) and links **every** node block of the package. A share is a list
of `{ "cid": <package manifest>, "pkg": <dir name> }` per library name. A receiver lands each manifest as
`<nick> - <lib>/<pkg>/.package.json`, fetches the blocks it names into that dir under their CID names, completes
any cross-package closure the same way, and prunes a node file that neither the manifest names nor its closure
reaches (an older generation's copy). A changed package is a changed manifest CID; an unchanged one is not
re-landed. There is no library-level CID. (VidyaGod: `PublishLibrary`, `PlanReceivedFetches`, `LandReceivedPackages`,
`PruneStaleReceived`.)

## 4.6 Local packages and shadowing

A user's own bundle (authored locally, not from a source) is part of the graph like any other. Because of
first-seen-wins (I1), an implementation MAY order scanning so that a **local** bundle shadows a source's bundle
declaring the same id — the user's edited copy wins. This is the one intended use of id shadowing; it MUST be
deterministic (local before remote), and a validator SHOULD still flag cross-*source* collisions.

## 4.7 What "installed" means

A node being *in the index* means it is *known*, not that it is *runnable*:

- A launchable is **hydrated** when every `Content` node in its closure is present locally (its bytes are on disk).
  Otherwise some content must be fetched first (chapter 14). A node whose `PATH` is *runtime-sourced* — a `%variable%`
  that only resolves to a real path at mount time, like a runner's prefix-assembly mounts — has no on-disk file at all
  and is never counted as missing.
- A runner is **installed** when its **build** is hydrated and (if it generates a prefix) its prefix artifact exists.
  "Its build" is the real on-disk content in its closure, again excluding the runtime-sourced nodes: counting those
  makes every prefix-generating runner look permanently un-installed.

Implementations distinguish "known" from "hydrated/installed" to drive UI (browse vs. play) and to gate launches.
(VidyaGod: `NodeHydrated`, `RunnerInstalled`.)

Next: [`Content` nodes](05-layers.md).
