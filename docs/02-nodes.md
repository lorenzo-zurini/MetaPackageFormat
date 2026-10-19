# 02 · Nodes

A node is one JSON object in one file. This chapter defines its fields, its identity and how node files are kept on
disk. Its layers are chapters 03–09.

## 2.1 A minimal node

```json
{ "LABEL": "Cave Story pristine",
  "LAYERS": [ { "ZIP": "Doukutsu Monogatari 1.0.0.6.zip", "TARGET": "FILES/%GameDir%" } ] }
```

`LAYERS` is the only required field. A node with an empty list is valid and does nothing.

## 2.2 Field reference

| Field | Type | Meaning |
|-------|------|---------|
| `LAYERS` | array, required | The ordered layers, bottom → top (chapter 03). |
| `LABEL` | string | A cosmetic name. Never a key; may repeat. Shown wherever the node is named. |
| `VARIANT` | non-empty string | The node is on the shelf under this name (chapter 09 §9.4). |
| `RECOMMENDED` | array of strings | Tile `UID`s under which this node is recommended: the default row for a variant, pre-ticked for a graft (chapters 09 and 12). |
| `SECTION` | non-empty string | Where a graft sits in the launch window's graft tree: a path whose `/` nests (`"Mods/Graphics"`). |
| `COMMENT` | string | Why the node exists, for authors. Part of the node's identity. |
| `POS` | `[x, y]` | Where an editor draws the node. Not part of the identity. |
| `CID` | string | A working-tree handle only (§2.5). Not part of the identity. |

Any other field MUST be refused by a validator. `VARIANT`, `RECOMMENDED` and `SECTION` describe the node itself and
**do not fold**: a node that contains a variant is not that variant, and a node that contains a graft is not a
graft.

## 2.3 Identity

A node's identity is the **CID of its canonical bytes** (invariant I1):

1. Drop `CID` and `POS` from the object.
2. Serialise as JSON in UTF-8 with object keys sorted by code point, arrays in order, no insignificant whitespace.
3. The CID is CIDv1, raw codec (`0x55`), multihash sha2-256 of those bytes, written in base32 multibase
   (`bafkrei…`).

This is exactly the CID an IPFS node gives the canonical bytes added as a file with Kubo's default parameters
(256 KiB chunks, raw leaves): a node of up to 256 KiB is one raw block. An implementation MAY refuse a node larger
than 256 KiB (VidyaGod does).

The same bytes give the same CID on every machine, with no network. Changing anything that is part of the node,
including `COMMENT`, `LABEL` or the order of layers, makes a new node. Every node that contains it then names an old
CID and must be re-minted to pick the change up (§2.6).

*(VidyaGod: `Cid::Canonical` / `Cid::OfNode` in `cid.cpp`.)*

## 2.4 References

Every reference to another node is its CID, as a plain string:

| Where | Followed? |
|-------|-----------|
| `NODE` | yes: fetched and kept with whatever contains it |
| `ANY`, `NOT` | no: compared against a resolution, never fetched |

Content files are referenced by their own CIDs (`SOURCE`, `TILE.COVER.SOURCE`, chapter 15). Runners are never
referenced (I9).

## 2.5 Files on disk

**One node per file.** A package folder holds node files, each named `<cid>.json` and containing the node, beside
the content files those nodes name (chapter 15 §15.1). An implementation reads every `.json` file in a package
folder that is a node object (it has a `LAYERS` array) and keys it by its CID.

The **working tree** is what an author edits. A hand-written tree MAY give each node a stored `CID` field, a handle
that other nodes' `NODE`, `ANY` and `NOT` layers use before any real CID exists. **Freezing** resolves the handles:
nodes are frozen in dependency order (contained before containing), each handle in a frozen node is replaced by the
CID its target froze to, and the frozen node's own CID follows from its bytes. A reference to a handle that did not
freeze is an error for that node only; the rest of the library freezes.

A node whose file carries no `CID` field is keyed by the CID in its file name, or, failing that, by a synthetic key
that nothing can reference. Two files with the same handle and different content mean a stale or forged handle: the
first one read keeps the handle, and the other is still indexed under its own CID, never dropped.

*(VidyaGod: `NodeGraph::GatherWorkingTree` and `FreezeToIndex` in `nodegraph*.cpp`.)*

## 2.6 Editing and re-minting

Editing a node changes its CID. An editor that saves a node MUST re-mint every node in the library that references
it (by `NODE`, `ANY` or `NOT`), in dependency order, so that no reference is left pointing at the old bytes, and
SHOULD verify afterwards that every reference resolves. A reference held by someone else's package (a friend's graft
whose `ANY` names your node) cannot be re-minted by you: it goes stale, and the graft is simply no longer offered on the new version.
This is why a graft's `ANY` should name the lowest node it needs, usually a pristine that never changes.

## 2.7 One home

**A node has exactly one home package**: its file, and the content files it names, live in one package folder.
Another package that uses it contains it by CID with a `NODE` layer, taking all of it or parts (chapter 05), and never
copies it. Something several packages use (a library, a shared engine) is therefore a package of its own. A node
found in two package folders is a validator error: delete the copy, keep the home.

One home is what makes every "where" question a fact rather than a guess: where a node's files are read from, whether
it is installed, which package an engine belongs to, what a package's pin holds. A friend's catalog copy of a package
you have installed is a view of that package, not a second home (chapter 15 §15.5).

Next: [Layers & addresses](03-layers-and-addresses.md).
