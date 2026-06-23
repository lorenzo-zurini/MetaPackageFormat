# 04 · Bundles, the library & indexing

This chapter defines the on-disk layout, how an implementation discovers nodes, and how a global graph is assembled from
many sources.

## 4.1 Bundles

A **bundle** is a directory containing one or more node `.json` files and the loose content those nodes reference (zips,
ROMs, cover images, generated prefixes). A bundle is purely a *file grouping* — it has no `NODE_ID`, no metadata of its
own, and no semantic role. It exists so that related files (a launchable, its content nodes, and the bytes they point at)
sit together and can be copied/shared/deleted as a unit.

```
[10972] Silent Hill 2/                 ← a bundle directory
├── silent_hill_2.json                 ← launchable node
├── silent_hill_2_base.json            ← content node (the game files)
├── silent_hill_2_enhanced.json        ← content node (an optional mod)
├── Silent Hill 2.zip                  ← bytes referenced by a layer
├── SH2 Enhanced Edition.zip
├── SH2_Cover.png                      ← bytes referenced by META.COVER
└── USERDATA/                          ← per-package durable saves (ch. 7) — created at runtime
```

A bundle MAY contain any number of nodes of any roles. A "game package" is typically one launchable plus its content
nodes; a "runner package" is a runner node plus its build content node(s). Nothing prevents one bundle from holding many
launchables (e.g. a multi-game collection).

### Relative-path resolution

Every relative path written inside a node — a layer's `PATH`, a `SOURCE.PATH`, `META.COVER.PATH` — resolves against **the
bundle directory of the node that declares it**, not against the launchable being run. This is essential for cross-bundle
composition: when a game's closure pulls in a runner whose build layer says `PATH: "GE-Proton10-30.zip"`, that path
resolves against the *runner's* bundle, wherever the runner lives. (VidyaGod: each node records its `BundleDir`;
`LayerLocator` joins relative paths against it.)

## 4.2 Library roots

A **library root** is a directory whose **immediate subdirectories are bundles**. The graph is built by scanning library
roots. Multiple roots are common and compose into one flat graph:

```
~/.VidyaGod/LIBRARY/                    ← contains library roots, one per source
├── VidyaGodPackages/                   ← a library root (a games repo)
│   ├── [299] Super Metroid/            ← bundle
│   ├── [7804] Age of Mythology/        ← bundle
│   └── …
└── VidyaGodRunners/                    ← a library root (a runners repo)
    ├── ge-proton10-30/                 ← bundle
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
    index = {}                                  # NODE_ID -> node
    for root in libraryRoots:
        if not isDirectory(root): continue
        for bundle in immediateSubdirectories(root):
            if not isDirectory(bundle): continue
            for file in files(bundle) where extension == ".json":
                json = parse(file)              # skip unparseable files (warn)
                node = ParseNode(json, file, bundle)
                if node is null: continue        # no NODE_ID ⇒ not a node, ignore
                if node.NODE_ID in index:
                    warn("duplicate NODE_ID, keeping first-seen")
                    continue                     # invariant I1: first-seen wins
                index[node.NODE_ID] = node
    return index
```

`ParseNode` reads the fields in [chapter 2 §2.2](02-nodes.md), applies defaults, and records the source file and bundle
directory. A JSON file that is not an object, or lacks a non-empty string `NODE_ID`, yields no node and is silently
skipped (it may be unrelated data living in the bundle).

**Duplicate ids (invariant I1).** If two files declare the same `NODE_ID`, the **first one seen wins** and the second is
dropped with a warning. Scan order across roots is therefore observable; an implementation SHOULD make it deterministic
(e.g. roots in a configured order, bundles and files sorted). Authors MUST treat `NODE_ID` collisions as errors to avoid
depending on scan order. (One legitimate use of shadowing: a user's *local* package overriding a repo copy of the same id
— see §4.5.)

## 4.4 Distribution: repositories

Bundles are distributed as ordinary directories, so any transport works (a zip, a synced folder, a git repo). The
reference implementation models a **repository** as a git remote cloned into a library root and refreshed on launch. A
repository is just a library root whose bundles happen to be version-controlled.

Repository sync MUST be **content-safe**: the heavy, content-addressed payloads (downloaded layer zips, generated
prefixes, fetched ROMs) live *inside* the bundles but are **untracked** (gitignored). A sync updates only the tracked
node `.json` files and MUST NEVER delete the clone or its untracked content. The reference sync:

1. `git pull --ff-only` — the normal fast-forward; leaves untracked content alone.
2. On failure (diverged/rewritten history): `git fetch` then `git reset --hard @{u}` — realigns *tracked* files to
   upstream; untracked content survives.
3. On any failure of both: keep the last-synced clone untouched. Never nuke it.

(VidyaGod: `SyncGitRepository`/`SyncRepositories` in `packagecatalog.cpp`.) The format does not mandate git — only that
*whatever* the transport, refreshing node descriptions MUST NOT destroy locally-hydrated content. An implementation MAY
support non-git transports identically.

## 4.5 Local packages and shadowing

A user's own bundle (authored locally, not from a repo) is part of the graph like any other. Because of first-seen-wins
(I1), an implementation MAY order scanning so that a **local** bundle shadows a repo bundle declaring the same id — the
user's edited copy wins. This is the one intended use of id shadowing; it MUST be deterministic (local before remote),
and a validator SHOULD still flag cross-*repo* collisions.

## 4.6 What "installed" means

A node being *in the index* means it is *known*, not that it is *runnable*:

- A launchable is **hydrated** when every VFS layer in its content closure is present locally (its bytes are on disk).
  Otherwise some content must be fetched first (chapter 14).
- A runner is **installed** when its build is hydrated and (if it generates a prefix) its prefix artifact exists.

Implementations distinguish "known" from "hydrated/installed" to drive UI (browse vs. play) and to gate launches.
(VidyaGod: `NodeHydrated`, `RunnerInstalled`.)

Next: [VFS content layers](05-layers.md).
