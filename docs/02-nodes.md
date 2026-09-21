# 02 · The Node object

A node is a single JSON object. It is **one layer**: one `TYPE`, that type's payload hoisted directly onto the node,
and the `PARENTS` edges that place it in the graph. This chapter is the exhaustive field reference for the envelope;
each `TYPE`'s payload has its own chapter ([ch. 3](03-roles.md) indexes them).

There is **no `ROLE`** and **no `LAYERS` array**. A node that would once have carried twenty layers is twenty nodes in
a chain, and the order those layers had is the `PARENTS` edges between them.

## 2.1 Identity & the minimal node

The only structurally-required field is `TYPE`. A file that does not parse as an object with a string `TYPE` is
**not a node** and MUST be ignored by the indexer (it might be unrelated JSON sitting in a bundle). A node's
identity is its **CID**; `LABEL` is optional.

The minimal valid node:

```json
{ "TYPE": "Group", "LABEL": "Some Pretty Name" }
```

A node with no `TYPE` is a `Group`: pure composition, no payload, existing only to gather `PARENTS` under one name.

## 2.2 Envelope field reference

These fields apply to a node of **any** `TYPE`. Every one is optional (identity is the CID). Unknown fields MUST be ignored
(forward-compatibility). Defaults are applied exactly as listed.

| Field | JSON type | Default | Meaning |
|-------|-----------|---------|---------|
| `CID` | string | (working-tree only) | The node's **authoring handle** — the CID it last minted to. `PARENTS`/`LIBRARYITEM` reference a node by this handle; the index and the editor key on it. It is **stripped at freeze** (a block cannot contain its own hash) and so is NEVER part of identity and never ships — identity is purely the CID *derived* from the frozen block. It is unique by construction (a content hash), so distinct nodes never collide on it. A node not yet minted carries a stable placeholder handle (e.g. `"draft-7"`); the next publish mints the real CID and writes it back here, remapping every reference. Between edits the stored `CID` may be stale (its content changed but it has not been re-minted) — `recompute ≠ stored CID` is exactly the "edited since publish" signal. |
| `LABEL` | string | `""` (optional) | A pretty, human name (`"Age of Empires II — Retail Clean Install"`), shown in every UI. It is **purely cosmetic**: NEVER a key and NEVER a reference target. Distinct nodes MAY share a `LABEL` freely (`v1.21b` for both a RoC and a TfT edition, two `Vanilla` editions) — identity is the CID, never the label. It travels in the frozen block, so a receiver shows the name with no re-labeling, and editing it changes the node's CID (it is part of the block). A node MAY omit `LABEL`. Renaming touches nothing else (the handle is the `CID`). |
| `TYPE` | string | `"Group"` | Which layer this node *is*. One of `Content`, `RegEdit`, `FileEdit`, `BinaryPatch`, `DllOverride`, `DeclarePersist`, `CustomVar`, `DeclareExec`, `DeclareLibraryItem`, `Group`. An unknown `TYPE` MUST be treated as an error, not silently ignored — a node whose payload nobody applies is a package that quietly does less than it says. |
| `PARENTS` | array of string | `[]` | The nodes this node depends on — each is another node's `CID` handle (a placeholder handle in an unminted tree, a real CID once frozen). Every parent is pulled into the closure, and every parent is applied **before** this node — that is how layer order survives the flattening. A parent handle that names no node in the index is a **dangling reference** and MUST be an error. |
| `TOGGLE` | string | `"on"` | `"on"` \| `"off"`. `"off"` makes this node a *toggleable add-on*: present in the graph, not applied unless the user (or `--module <id>=on`) switches it on. `"on"` with a parent that is off still means "on when reached". This single field replaced the old `OPTIONAL`+`DEFAULT` pair, which were never independent — `OPTIONAL:false` made `DEFAULT` meaningless, and `OPTIONAL:true, DEFAULT:false` is exactly `TOGGLE:"off"`. (It also had to go: once payloads moved onto the node, `DEFAULT` collided with `CustomVar`'s own `DEFAULT`.) |
| `EXCLUDE` | array of string | `[]` | nodes this node is mutually exclusive with, matched by `LABEL` (NOT the CID handle). This is the one place a label is a soft match key, and it is safe because exclusion is only ever evaluated within a **single game's selected variant set**, where labels are distinct — so it needs no forward-resolvable CID and is not remapped at freeze. **Symmetry is expected** (both nodes should list each other); validators warn if not. When two excluded nodes would both be enabled, first-kept wins. See [ch. 12 §12.4](12-resolution.md). |
| `WHEN` | string | `""` | A boolean condition over `%variables%` ([ch. 8 §8.8](08-variables.md)). When it does not hold the node is **inert**: its payload is not applied, but it is still a graph node, so its `PARENTS` are still reached. Not accepted on `DeclareExec`/`DeclareLibraryItem`, whose payloads become the node's identity at index time — use `TOGGLE` there. |
| `COMMENT` | string | `""` | Free text for a human. Never interpreted. |

**Canvas position is a node field, and where an author MOVES a node is not.** The two halves answer different
questions and MUST be stored in different places.

- **`POS`** — optional, `[x, y]` numbers — is the node's **default** position: where a tool SHOULD draw it when
  nobody on this machine has an opinion. It is written by the publisher, not by dragging (see below), so it
  changes only when a package is republished. Without it a received package opens as a pile at the origin or is
  re-arranged from scratch by whatever heuristic the reader happens to implement, which means no two people
  looking at the same package see the same picture.
- **Where this machine has since dragged a node** is a LOCAL PREFERENCE and MUST NOT be written into the node. A
  package is content-addressed and a Meta-CID is minted **add-by-reference, in place, over the author's own
  files** ([ch. 14](14-content-addressing.md)), so anything in a node file is in the CID: writing a drag back
  would republish the package for every peer on every mouse-up. It belongs in the tool's own per-user
  configuration, keyed by bundle. (The reference implementation keeps it in `GlobalConfig.JSON` under
  `EDITORLAYOUT`, keyed by bundle PATH then `LABEL`.)

A reader resolves a node's position weakest-to-strongest: the computed default from its own layout algorithm,
then `POS`, then the local override. A tool that publishes SHOULD stamp the layout as it stands at that moment —
local overrides included, since that is the picture the author arranged — into every node's `POS`, and MUST leave
a node whose `POS` is already correct untouched, so republishing an unchanged bundle mints the same CID.

Stamping is an AUTHORING act, so a tool SHOULD do it only for a bundle that machine actually authored. Writing
`POS` into a bundle merely fetched from someone else changes bytes that a content address elsewhere still claims
to serve, and adds nothing: a reader with no `POS` computes the same default anyway.

`POS` carries no semantics whatsoever: it never affects resolution, ordering, closure, or what a package does. An
implementation with no canvas ignores it, and a malformed `POS` (not two numbers) MUST be treated as absent
rather than as an error.

Two fields are **derived, not authored** (an implementation computes them at index time; they never appear in JSON):

- **source file path** — the `.json` file the node was read from.
- **bundle directory** — the directory containing that file. **Every relative path on this node resolves against this
  node's own bundle directory** — which makes cross-bundle references correct (a runner's build content resolves
  against the runner's bundle even when pulled into a game's closure). See [ch. 5 §5.2](05-layers.md).

### A file may hold more than one node

The canonical layout is one node per file, named after the `LABEL`. A `.json` file MAY instead contain a **JSON array**
of node objects; an indexer MUST accept both. This keeps a twenty-node chain from becoming twenty files when the author
would rather keep it together. Entries in such a file that are not nodes MUST be preserved verbatim by any tool that
rewrites it.

## 2.3 A representative chain (single-variant game)

A whole small game. Read it bottom-up: the launchable's `PARENTS` name what must be applied before it, and each of those
names what comes before *it*.

```jsonc
[
  { "LABEL": "aom_content",  "TYPE": "Content", "FORM": "zip", "PATH": "aom.zip",
    "SOURCE": { "TYPE": "ipfs", "CID": "Qm…" } },

  { "LABEL": "aom_registry", "TYPE": "RegEdit", "PARENTS": ["aom_content"],
    "EDITS": [ { "ARCHITECTURE": ["32"],
                 "HKLM": { "Software": { "Microsoft": { "Microsoft Games": { "Age of Mythology": {
                     "InstallationDirectory": "C:\\7804" } } } } } } ] },

  { "LABEL": "aom",          "TYPE": "DeclareLibraryItem", "UID": "7804",
    "TITLE": "Age of Mythology",
    "COVER": { "PATH": "AoM_Cover.jpg", "SOURCE": { "TYPE": "ipfs", "CID": "Qm…" } },
    "META":  { "RELEASEDATE": "2002-10-30", "DEVELOPER": "Ensemble Studios", "UMUID": "266840" } },

  { "LABEL": "aom_game",     "TYPE": "DeclareExec", "PARENTS": ["aom_registry", "aom"],
    "HOST": "win32", "PATH": "aom.exe",
    "ARGS": ["xres=%ScreenWidth%", "yres=%ScreenHeight%"] }
]
```

Three things to notice, because they are the whole model:

1. **The tile is a parent of the launchable**, not the other way round. `aom` carries no content; `aom_game` reaches it
   through `PARENTS` and so inherits its metadata. Several launchables naming the same tile *are* the same game's
   variants ([ch. 3 §3.4](03-roles.md)).
2. **`aom_game` has no `GUEST`**, so it is a launchable. Give it a `GUEST` list and the same type declares a runner
   instead ([ch. 9](09-exec.md)).
3. **Nothing declares an order.** `aom_content` is applied before `aom_registry` because it is its parent. Delete the
   edge and they become unordered — which the format says is *unspecified*, so if you need an order, express it as an
   edge.

## 2.4 Sibling order is unspecified

Within one node's `PARENTS`, the order of the **listed ids** is an implementation detail. Two parents of the same node
are not ordered with respect to each other by this specification, and an author MUST NOT rely on the array order to
resolve a write conflict between them. If two nodes both write the same path or the same registry value and the result
depends on which goes last, **make one a parent of the other**; that is the only way to say it.

An implementation SHOULD be deterministic anyway (VidyaGod does a post-order DFS in `PARENTS` list order, so the same
graph always resolves the same way) — reproducibility is worth more than enforcing the rule by shuffling. But a
validator MUST report an **unordered write conflict** when it finds one, and it MUST do so over a *resolved closure*,
never over a whole library: two nodes that never meet in any closure are not in conflict.

## 2.5 Authoring conventions (non-normative)

- Name the file after the `LABEL` (`aom_game.json`), or keep one chain in one array file. The indexer does not
  require either, but it keeps a bundle navigable.
- Prefix every node in a chain with the thing it belongs to (`tonic_trouble_retail_registry`,
  `tonic_trouble_retail_files`, `tonic_trouble_retail_exec`). Ids are global; a bare `registry` is a collision waiting.
- Keep `LABEL`s descriptive and stable. They are the public contract: other packages, runner libraries and saved user
  settings reference them by id. Renaming a node id is a breaking change.

Next: [Node types](03-roles.md).
