# 14 · Content addressing & distribution

Node descriptions are tiny text; the bytes they reference (game zips, ROMs, runtime builds, cover art) are large. MPF
keeps the two separate and links them with a **content-addressed source**, so the same description can be shared freely
and the bytes fetched on demand and verified by hash. This is what makes "the graph is the manifest of what to fetch"
(chapter 1) literal, and what enables peer-to-peer distribution.

## 14.1 The `SOURCE` block

Any VFS layer (chapter 5) and any cover (`META.COVER`, chapter 3) MAY carry a `SOURCE`:

```json
"SOURCE": { "TYPE": "ipfs", "CID": "QmRVRg1spGsV96NuKv7HH27kwhnPKwadNnBp3ZyarDPFwY",
            "PATH": "optional local-path override" }
```

| `SOURCE` field | Meaning |
|----------------|---------|
| `TYPE` | the content backend. `"ipfs"` = fetch by `CID`; `"path"` (the default if no SOURCE) = local-only. |
| `CID` | for `"ipfs"`: the content identifier (hash) of the bytes. Both the *address to fetch from* and the *integrity check*. |
| `PATH` | optional; overrides the layer's top-level `PATH` as the local file location. |

The pairing of a **local `PATH`** with a **content `CID`** is the heart of the model:

- The local path is where the bytes live (or will live) on disk.
- The CID is how to obtain them if the path is empty, and how to verify them.

(VidyaGod: `LayerLocator` returns `(localPath, cid)`; the IPFS backend is an embedded node.)

## 14.2 The present-vs-fetch rule (invariant I8)

A file **present** at its resolved local path is authoritative and used as-is — the `CID` is *not* consulted (no re-hash,
no network). The `CID` is consulted **only** to fetch a **missing** file. Therefore:

- A package whose layers are all present locally runs fully offline, immediately.
- A package whose layers are missing but have CIDs is *fetchable* — not an error until a fetch fails.
- A layer with **neither** a present file **nor** a CID is *unavailable*; a launch needing it MUST be refused with a clear
  diagnostic (don't mount nothing and pretend success).

(VidyaGod: `EnsureSources` classifies each layer; `MaterializeLayers` fetches missing-but-fetchable ones to their path,
self-healing the package.)

## 14.3 Hydrate (install)

**Hydrating** a launchable = ensuring every VFS layer in its closure (and the runner chain's builds) is present locally,
fetching missing-but-fetchable layers by CID to their declared paths. After hydration the package is fully local and
subsequent launches need no network. The set of CIDs to fetch is just *the closure's layer CIDs* — the graph is the
download manifest. (VidyaGod: `HydrateNode`, and `CollectFetchTargets` for batching a game's content with its runners'
builds into one concurrent download.)

The inverse, **dehydrating** a *local* package, deletes the closure's local content files (keeping the node `.json`s and
the cover), drops the CIDs from the local content store, and returns the package to a re-downloadable state — reclaiming
disk while keeping it installable. (VidyaGod: `DehydrateNode`.)

## 14.4 Publish (dehydrate-for-sharing) & seed

To *share* a package you've authored, you **publish** it: for each VFS layer and each cover, the implementation adds the
local file to the content network, obtaining its `CID`, and records that CID in the node's `SOURCE`. The result is a
**dehydrated** package — node `.json`s with CIDs but (optionally) without the heavy bytes — that anyone can hydrate.
Publishing is the bridge from "a folder of files on my disk" to "an address others can fetch."

- **Publish** writes the CIDs into the nodes and seeds the bytes. (VidyaGod: `PublishPackage`, which can also export a
  manifest-only dehydrated copy to a destination directory.)
- **Seed / re-seed** re-establishes serving of an already-published package: walk a directory's nodes and re-add each
  CID-referenced file by its recorded SOURCE, so a machine becomes a provider for that content again. Files whose bytes
  changed since publish (so they no longer match the recorded CID) are reported rather than silently re-hashed. (VidyaGod:
  `SeedDirectory`, `MirrorDehydrated`.)

The STORE-zip requirement (chapter 5) matters here: a layer's bytes are addressed and served as the zip file itself, so
the same zip is simultaneously the install medium, the integrity unit, and the content-network object.

## 14.5 The content backend

The format names the backend in `SOURCE.TYPE` and is otherwise backend-agnostic. The current backend is **IPFS**:
content-addressed, peer-to-peer, with CIDs as both address and hash. A CID can be fetched from any peer that has it,
which gives MPF P2P distribution for free — a popular package is served by everyone who has it, no central host required.
(The reference implementation embeds an IPFS node in-process; it has been demonstrated fetching a game's content
cross-network between different NATed networks and seeding it onward.)

Nothing in the format precludes other backends (an HTTP mirror, a different content-addressed store); a conforming
implementation MUST support the backends named in the packages it consumes, and SHOULD support `ipfs` for interoperability
with the existing ecosystem. A `SOURCE.TYPE` an implementation doesn't understand makes that layer unavailable (treat as
no source) — so always ship at least a backend the target audience supports.

## 14.6 Integrity & trust (non-normative)

Because a `CID` is a hash, fetched content is self-verifying: bytes that don't hash to the CID are rejected by the
content layer, so a fetch can't substitute tampered content for a given CID. What a CID does *not* tell you is whether the
*publisher* is trustworthy — that's a higher-level concern (signing a set of node files, curating which repos you sync).
The format provides integrity of *bytes-for-a-CID*; provenance of *CIDs-for-a-package* is left to distribution policy.

Next: [Validation](15-validation.md).
