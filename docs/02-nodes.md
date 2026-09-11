# 02 · The Node object

A node is a single JSON object. It is **one layer**: one `TYPE`, that type's payload hoisted directly onto the node,
and the `PARENTS` edges that place it in the graph. This chapter is the exhaustive field reference for the envelope;
each `TYPE`'s payload has its own chapter ([ch. 3](03-roles.md) indexes them).

There is **no `ROLE`** and **no `LAYERS` array**. A node that would once have carried twenty layers is twenty nodes in
a chain, and the order those layers had is the `PARENTS` edges between them.

## 2.1 Identity & the minimal node

The only structurally-required field is `NODE_ID`. A file that does not parse as an object with a non-empty string
`NODE_ID` is **not a node** and MUST be ignored by the indexer (it might be unrelated JSON sitting in a bundle).

The minimal valid node:

```json
{ "NODE_ID": "some_unique_id", "TYPE": "Group" }
```

A node with no `TYPE` is a `Group`: pure composition, no payload, existing only to gather `PARENTS` under one name.

## 2.2 Envelope field reference

These fields apply to a node of **any** `TYPE`. Every one is optional except `NODE_ID`. Unknown fields MUST be ignored
(forward-compatibility). Defaults are applied exactly as listed.

| Field | JSON type | Default | Meaning |
|-------|-----------|---------|---------|
| `NODE_ID` | string | — (required) | Globally-unique identity. The currency of every edge. MUST be non-empty and unique across the whole graph (invariant **I1**). Convention: lowercase snake-case slug, specific enough not to collide — `tonic_trouble_retail_registry`, `ge-proton10-30`, `aom_base_content`. Name length is free; overlap is not. |
| `TYPE` | string | `"Group"` | Which layer this node *is*. One of `Content`, `RegEdit`, `FileEdit`, `BinaryPatch`, `DllOverride`, `Persist`, `CustomVar`, `DeclareExec`, `DeclareLibraryItem`, `Group`. An unknown `TYPE` MUST be treated as an error, not silently ignored — a node whose payload nobody applies is a package that quietly does less than it says. |
| `PARENTS` | array of string | `[]` | The `NODE_ID`s this node depends on. Every parent is pulled into the closure, and every parent is applied **before** this node — that is how layer order survives the flattening. A parent id that names no node in the index is a **dangling reference** and MUST be an error. |
| `TOGGLE` | string | `"on"` | `"on"` \| `"off"`. `"off"` makes this node a *toggleable add-on*: present in the graph, not applied unless the user (or `--module <id>=on`) switches it on. `"on"` with a parent that is off still means "on when reached". This single field replaced the old `OPTIONAL`+`DEFAULT` pair, which were never independent — `OPTIONAL:false` made `DEFAULT` meaningless, and `OPTIONAL:true, DEFAULT:false` is exactly `TOGGLE:"off"`. (It also had to go: once payloads moved onto the node, `DEFAULT` collided with `CustomVar`'s own `DEFAULT`.) |
| `EXCLUDE` | array of string | `[]` | `NODE_ID`s this node is mutually exclusive with. **Symmetry is expected** (both nodes should list each other); validators warn if not. When two excluded nodes would both be enabled, first-kept wins. See [ch. 12 §12.4](12-resolution.md). |
| `WHEN` | string | `""` | A boolean condition over `%variables%` ([ch. 8 §8.8](08-variables.md)). When it does not hold the node is **inert**: its payload is not applied, but it is still a graph node, so its `PARENTS` are still reached. Not accepted on `DeclareExec`/`DeclareLibraryItem`, whose payloads become the node's identity at index time — use `TOGGLE` there. |
| `COMMENT` | string | `""` | Free text for a human. Never interpreted. |

**Canvas position is deliberately NOT a node field.** An authoring tool needs somewhere to remember where a node
sits on screen, and the obvious place — a `POS` key on the node — is wrong: a package is content-addressed, and a
Meta-CID is minted **add-by-reference, in place, over the author's own files** ([ch. 14](14-content-addressing.md)),
so anything in a node file is in the CID. Storing layout there means dragging a box republishes the package for
every peer. Layout belongs in a sidecar the publisher already excludes — the reference implementation uses
`<bundle>/LAYOUT.vglayout`, keyed by `NODE_ID`: a Meta-CID is text-only-JSON by construction, and nothing mounts
the bundle directory itself, so a non-`.json` file there reaches neither peers nor the running game. (`USERDATA`
looks like the obvious home and is not one — a whole-runtime `Persist` makes it the writable top branch of the
runtime union, so anything in it appears inside the game's own directory.)

Two fields are **derived, not authored** (an implementation computes them at index time; they never appear in JSON):

- **source file path** — the `.json` file the node was read from.
- **bundle directory** — the directory containing that file. **Every relative path on this node resolves against this
  node's own bundle directory** — which makes cross-bundle references correct (a runner's build content resolves
  against the runner's bundle even when pulled into a game's closure). See [ch. 5 §5.2](05-layers.md).

### A file may hold more than one node

The canonical layout is one node per file, named after the `NODE_ID`. A `.json` file MAY instead contain a **JSON array**
of node objects; an indexer MUST accept both. This keeps a twenty-node chain from becoming twenty files when the author
would rather keep it together. Entries in such a file that are not nodes MUST be preserved verbatim by any tool that
rewrites it.

## 2.3 A representative chain (single-variant game)

A whole small game. Read it bottom-up: the launchable's `PARENTS` name what must be applied before it, and each of those
names what comes before *it*.

```jsonc
[
  { "NODE_ID": "aom_content",  "TYPE": "Content", "FORM": "zip", "PATH": "aom.zip",
    "SOURCE": { "TYPE": "ipfs", "CID": "Qm…" } },

  { "NODE_ID": "aom_registry", "TYPE": "RegEdit", "PARENTS": ["aom_content"],
    "EDITS": [ { "ARCHITECTURE": ["32"],
                 "HKLM": { "Software": { "Microsoft": { "Microsoft Games": { "Age of Mythology": {
                     "InstallationDirectory": "C:\\7804" } } } } } } ] },

  { "NODE_ID": "aom",          "TYPE": "DeclareLibraryItem", "UID": "7804",
    "TITLE": "Age of Mythology",
    "COVER": { "PATH": "AoM_Cover.jpg", "SOURCE": { "TYPE": "ipfs", "CID": "Qm…" } },
    "META":  { "RELEASEDATE": "2002-10-30", "DEVELOPER": "Ensemble Studios", "UMUID": "266840" } },

  { "NODE_ID": "aom_game",     "TYPE": "DeclareExec", "PARENTS": ["aom_registry", "aom"],
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

- Name the file after the `NODE_ID` (`aom_game.json`), or keep one chain in one array file. The indexer does not
  require either, but it keeps a bundle navigable.
- Prefix every node in a chain with the thing it belongs to (`tonic_trouble_retail_registry`,
  `tonic_trouble_retail_files`, `tonic_trouble_retail_exec`). Ids are global; a bare `registry` is a collision waiting.
- Keep `NODE_ID`s descriptive and stable. They are the public contract: other packages, runner libraries and saved user
  settings reference them by id. Renaming a node id is a breaking change.

Next: [Node types](03-roles.md).
