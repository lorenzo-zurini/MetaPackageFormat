# 02 · The Node object

A node is a single JSON object. There is **one node kind**: a node is facets + any subset of the **payload
sections** + the **one edge**, `OVER`. A node is *one meaningful change*, and a change may span kinds — a widescreen
fix that is a byte patch, an ini edit and a knob is ONE node carrying `PATCHES`, `FILEEDITS` and `VARS`. A node with
no payload is just a node: it exists to be `OVER` other nodes under one name.

There is **no `TYPE`**, no `ROLE`, no `PARENTS`, no `EXCLUDE`, no `LIBRARYITEM`. Everything a node *is* — launchable,
a title, content, a graft, a library — is **derived** from what it carries and where it sits ([ch. 3](03-roles.md)).

## 2.1 Identity & the minimal node

A file is a node iff it is a JSON object carrying at least one node field (`CID`, `LABEL`, `OVER`, `TILE`,
`ENTRYPOINTS`, or a payload section). Anything else in a bundle directory is not a node and MUST be ignored by the
indexer. **A legacy object carrying `TYPE` is not a node**: the library is migrated once, never read two ways.

A node's identity is its **CID**; `LABEL` is optional. The minimal valid node:

```json
{ "LABEL": "Some Pretty Name" }
```

## 2.2 Field reference

Every field is optional (identity is the CID). **A key outside this vocabulary MUST be refused**, not ignored: a
typo'd payload key (`"LAYER"`) is otherwise a payload that silently never applies — the one failure the format
exists to make impossible.

| Field | JSON type | Meaning |
|-------|-----------|---------|
| `CID` | string | The node's **authoring handle** — the CID it last minted to. `OVER` references a node by this handle; the index and the editor key on it. It is **stripped at freeze** (a block cannot contain its own hash), so it is NEVER part of identity and never ships. A node not yet minted carries a stable placeholder handle (`"draft-7"`); the next publish mints the real CID and writes it back here, remapping every reference. |
| `LABEL` | string | A pretty, human name, shown in every UI. **Purely cosmetic**: never a key, never a reference target; distinct nodes MAY share one. It travels in the frozen block (editing it changes the CID). |
| `OVER` | array | **The one edge.** See §2.3. |
| `TOGGLE` | string | `"on"` \| `"off"`. Present ⇒ the node is user-toggleable and the value is the author's default. On a node inside a launchable's composition it is an optional module; on a graft it is whether the graft is pre-selected. Absent ⇒ a plain requirement (always applied when reached) or an unselected graft. Any other value MUST be refused. |
| `WHEN` | string | A boolean condition over `%variables%` ([ch. 8 §8.8](08-variables.md)). When it does not hold the node's payload is **inert** (its `OVER` is still reached). A `WHEN` on a node with no payload has nothing to gate and MUST be an error ([ch. 15](15-validation.md)). |
| `PUBLISH` | bool | This node is a **share-list root** ([ch. 4](04-bundles-and-library.md)). Minted IN (it is identity). |
| `TILE` | object | `{ UID, TITLE, COVER, META }` — the identity of a **title**. On a launchable it IS the tile; every node OVER a tile-carrying node inherits it. See [ch. 3 §3.2](03-roles.md). |
| `ENTRYPOINTS` | array of object | What to run. Each entry is a **variant** ([ch. 9](09-exec.md)). Present ⇒ the node is launchable (or a runner, when an entry lists `GUEST`). Never conditional: an entry MUST NOT carry `WHEN`. |
| `LAYERS` | array | VFS content: zips, dirs, files, deltas ([ch. 5](05-layers.md)). |
| `PATCHES` | array | Byte patches over pristine files, one entry per `FILE` ([ch. 6 §6.4](06-edit-layers.md)). |
| `FILEEDITS` | array | Text edits, one entry per `FILE` ([ch. 6 §6.3](06-edit-layers.md)). |
| `REGEDITS` | array | Registry hive trees, per architecture ([ch. 6 §6.1](06-edit-layers.md)). |
| `DLLOVERRIDES` | object | `dll → order` ([ch. 6 §6.2](06-edit-layers.md)). |
| `VARS` | array | Variables the player sets before launch ([ch. 8](08-variables.md)). |
| `PERSISTS` | array | What survives the run ([ch. 7](07-persistence.md)). |
| `COMMENT` | string | Free text for a human. Never interpreted. |
| `POS` | `[x, y]` | Canvas position — the author's default layout, stamped at publish, **non-semantic**, stripped at freeze. A local drag never writes here (see the note at the end of this chapter). |

An **empty** payload section (`"VARS": []`) MUST be refused like an unknown key: a node that says it contributes
something and contributes nothing.

Two fields are **derived, not authored** (computed at index time, never in JSON): the source file path, and the
**bundle directory** containing it — every relative path on the node resolves against its own bundle directory,
which is what makes cross-bundle references correct.

## 2.3 `OVER` — the one edge

`OVER` means **"I am made of you; you are under me."** It is a list of *requirements*, read as a conjunction (CNF):

| Entry | Meaning |
|-------|---------|
| `"cid"` | a plain requirement: that node must be present — it is mounted beneath me |
| `["cid", "cid", …]` | an **any-of group**: one of these must be present. A group of one is a plain entry. An empty group is unsatisfiable and MUST be refused. |
| `{ "NOT": "cid" }` | an **exclusion**: that node must not be selected alongside me |

```json
"OVER": [ "…skse", ["…sk-640", "…sk-659"], { "NOT": "…old-quest" } ]
```
reads *skse ∧ (640 ∨ 659) ∧ ¬old-quest*. Order among a node's own entries is the mount order (later = higher).
Nothing else is an edge: composition, compatibility ("works on either version"), dependency ("needs SKSE"),
exclusion and identity are all this one list. The direction is **newer → older**: the newer node names the CIDs of
what it builds on; the older side never enumerates what builds on it, so nothing that grows is ever listed.

A ref that names no node in the index is a **dangling reference** and MUST be an error, except a `NOT` (which
simply has no effect, and MAY be warned). A node MUST NOT both require and exclude the same node.

The edge is **not transitive for execution** ([ch. 12](12-resolution.md)): `1.16.5 OVER [1.16.4]` puts 1.16.4's bytes
under 1.16.5 and nothing else — not its entrypoint, not its mods.

### Canvas position

`POS` is the node's **default** position, written by the publisher; **where this machine has since dragged a node is
a local preference and MUST NOT be written into the node** (a package is content-addressed; a drag would republish
it). A reader resolves weakest-to-strongest: computed layout → `POS` → local override. A publisher SHOULD stamp the
layout as it stands and MUST leave a node whose `POS` is already correct untouched, so an unchanged bundle mints the
same CID. Stamping is an authoring act: never write `POS` into a bundle merely fetched from someone else. A
malformed `POS` is treated as absent, never as an error.

### One node per file

Each node lives in its own `.json` file, named after its `LABEL`. An indexer MUST also accept a file holding a JSON
array of nodes; tools that rewrite a package SHOULD emit one node per file and MUST preserve non-node entries
verbatim.

Next: [What a node is](03-roles.md).
